# Reaction Recovery Overview

## 1. Problem Statement

Drasi Lib currently provides **at-most-once delivery** between queries and reactions. The channel connecting them is an in-memory Tokio `mpsc` or `broadcast` channel. If the process crashes, all in-flight results are lost permanently because:
1. The query commits the source checkpoint *before* delivering results to the channel.
2. An un-delivered `QueryResult` simply evaporates.
3. On restart, the query skips re-evaluating the event, leaving the side-effect unexecuted.

Additionally, reactions have **no bootstrap story**. New reactions joining a running query miss all past state.

## 2. Design Goals

1. **At-least-once delivery** for persistent queries and durable reactions.
2. **O(1) Live Result Set Mutations** (replace `O(N)` `Vec` scans with a HashMap/KV store).
3. **Strict Decoupling** - Queries use fixed-size outboxes to bound disk growth, rejecting the "min-watermark" / position handle approach.
4. **Minimal Disk I/O overhead for volatile queries** - Volatile (in-memory) queries orchestrate sequence tracking entirely in memory and completely bypass all database transactions, serializers, and `fsync` operations. While there is a minor memory cost to retaining the outbox, this is offset by the CPU gains of O(1) diff application.
5. **Reaction authors control policy** - The framework provides the outbox and dedup; reactions drive checkpointing and recovery logic.

## 3. Architecture Overview

### Target Flow

```text
Query Processor                                    Reaction
  │                                                  │
  │  outer begin()                                   │
  │    process_source_change() → diffs               │
  │    stage_source_checkpoint(src, seq)             │
  │    result_seq = ++last_result_seq                │
  │    outbox.append(result_seq, QueryResult)        │
  │    update_live_results(diffs)                    │
  │  outer commit() [atomic]                         │
  │                                                  │
  │  dispatch_query_result(QueryResult { sequence }) │
  ├──── mpsc/broadcast channel ─────────────────────►│ priority queue
  │                                                  │ framework dedup (skip if seq ≤ checkpoint.seq)
  │                                                  │ execute side-effect
  │                                                  │ write_checkpoint(query_id, seq, config_hash)
  │                                                  │
  │◄── fetch_snapshot() ─────────────────────────────│ (on subscribe, optional)
  │◄── fetch_outbox(after_seq) ──────────────────────│ (on subscribe, optional)
```

## 4. Key Design Decisions

### Ring buffer outbox over position-handles
We use a fixed-size, self-pruning ring buffer (`outbox` namespace) rather than tracking per-reaction position handles. The query has no knowledge of downstream consumer state. This avoids unbounded disk growth from stalling consumers and simplifies the Query hot-path. 

### Dual bootstrap APIs (`fetch_snapshot` & `fetch_outbox`)
Reactions read missed data explicitly on start-up rather than having the query dump history into the live channel. This separates the replay path from the live data path. 

### Memory-Served Reads for Consistency
Both `fetch_snapshot` and `fetch_outbox` read strictly from an in-memory `RwLock<QueryOutputState>`, never from the persistent index during normal operation.
- **Consistency**: Guarantees readers don't see partially committed state.
- **Backend-Agnostic**: Avoids needing backend-specific read-snapshots. Storage namespaces are write-only during normal operation and read explicitly only on crash-recovery startup.

### Immutable Data Structures for Non-Blocking Snapshots
To serve massive materialized views (e.g., 1M+ rows) without blocking the live query pipeline or blowing up memory, `QueryOutputState` uses an immutable HAMT (Hash Array Mapped Trie) via the Rust `im` crate (`im::HashMap`) instead of a standard `std::collections::HashMap`.
- **The Alternative Considered (Storage-Level Snapshots)**: We considered serving snapshots directly from the persistent index (e.g., RocksDB `db.snapshot()`). While highly performant, it severely violated the "Backend-Agnostic" rule. Garnet (Redis), for instance, lacks point-in-time snapshots for hashes, which would force `drasi-lib` into complex, backend-specific dirty-scan-plus-outbox-reconciliation mechanics. 
- **The Trade-off**: `im::HashMap` allows `fetch_snapshot()` to acquire the read-lock, instantly `.clone()` the root of the tree in $O(1)$ time (structural sharing), and drop the lock. The Reaction can then paginate or stream the frozen clone at its own pace. The trade-off is a slight allocation tax on the Query Processor's write path, but this is highly acceptable due to **Amdahl's Law**: since overall query latency is completely dominated by graph evaluation and disk I/O, this minor CPU overhead on final map insertion has a negligible impact on overall throughput.

### Uniform APIs & State
The in-memory `QueryOutputState` and sequence tracking run for all queries. The only thing conditional on the index backend is the actual `write` to the persistent storage. 

**The Overhead Clarification:** For volatile queries, running this machinery introduces a slight memory footprint (buffering the `VecDeque` outbox) and CPU allocation tax (`im::HashMap` node creation). However, this is an intentional structural trade-off: it unifies the execution paths and gives volatile queries O(1) diff application performance (replacing the previous O(N) linear scans). Furthermore, tracking sequences in-memory opens the possibility for volatile reactions to detect `broadcast` channel drop lag, improving system observability even without a persistent disk.

## 5. Delivery Guarantees & Compatibility Matrix

| Query Index | Reaction Type | Allowed | Guarantee |
|-------------|---------------|---------|-----------|
| Volatile (Memory) | Volatile | ✅ | At-Most-Once |
| Volatile (Memory) | Durable | ❌ Error | Rejected at startup - outbox and snapshot do not survive process crash, making the durable reaction's checkpoint unrecoverable |
| Persistent (RocksDB/Garnet) | Volatile | ✅ | At-Least-Once delivery, At-Most-Once processing |
| Persistent (RocksDB/Garnet) | Durable | ✅ | At-Least-Once delivery and processing |

**Note on Idempotency**: Because the "At-Least-Once" guarantee implies that duplicate events will occur during crash recoveries, durable reactions that perform irreversible side effects (e.g., HTTP Webhooks) should ideally pass the `sequence` number as an idempotency key. It is the responsibility of the downstream system to handle deduplication if strict exactly-once semantics are required.

## 6. Recovery Flows

Recovery flows are dictated by the Reaction's archetype (State Synchronization vs Stateless Triggers). See details in `02.md`.

- **New Reaction (State Sync)**: `fetch_snapshot()` → apply state → save checkpoint → open live gate.
- **New Reaction (Event Trigger)**: Skip snapshot → save current sequence as checkpoint → open live gate.
- **Restart (No Gap)**: read state checkpoint → `fetch_outbox(checkpoint)` → process entries → open live gate.
- **Restart (Gap Detected)**: Reaction calls `fetch_outbox(checkpoint)` and receives `PositionUnavailable`. Executes its `recovery_policy` (`strict`, `auto_reset`, or `auto_skip_gap`) based on the plugin's designed behavior.

## 7. Out of Scope

This design assumes single-instance `DrasiLib` deployments. Horizontal scale-out and HA (multiple drasi-lib processes sharing a query or reaction against shared storage) are out of scope. A separate design would be required to coordinate:

- Distributed source subscription without duplicate processing.
- Outbox sequence allocation across processes.
- `live_results` consistency under concurrent writers.
- Reaction-checkpoint coordination across replicas.

Deploying multiple instances against shared persistent storage without such coordination produces silent data corruption and is unsupported.