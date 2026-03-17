# Checkpoint-Based Recovery for Drasi Lib

## 1. Problem Statement

Drasi Lib currently provides **at-most-once delivery** between sources and queries. The channel connecting them is a pure in-memory Tokio `mpsc` or `broadcast` channel. If the process crashes, all in-flight events are lost.

Even when queries use persistent indexes (RocksDB, Garnet) and sources resume from a durable log (e.g., Postgres WAL), there is a window where:

1. The source dispatches an event to the in-memory channel.
2. The source advances its read position (e.g., updates `current_lsn`) in the upstream log.
3. The source could also periodically ACK upstream (e.g., `StandbyStatusUpdate` to Postgres every ~5s).
4. The query has **not yet processed** these events.
5. Process crashes - the channel is gone, upstream won't re-send ACK'd events, and the query's persistent index is missing the corresponding state updates.

The result is **silent state divergence**. The query's materialized state no longer matches the source of truth, with no mechanism to detect or recover from the gap.

This problem also manifests **without crashes**: if `process_source_change` returns an error, the event is dequeued from the priority queue and lost. The source has already moved on. No retry, no recovery.

Additionally, persistent indexes are currently **pointless for crash recovery**. On restart, the query reopens its RocksDB database (retaining stale state) and either:

- **Re-bootstraps**: feeding insert events into an index that already contains entries, causing corruption or duplication.
- **Skips bootstrap**: receiving mid-stream events with no way to reconcile the gap between what's in the index and what the source sends.

There is no mechanism to determine "what has this index already processed?" and resume from that point.

---

## 2. Design Goals

Following are some design goals we are aiming for. Some of them might need to be compromised partly depending on the implementation of the specific changes.

1. **At Least Once processing** - Every source event will be processed exactly once by the query engine when it uses a persistent index. For Transient sources like HTTP that retry without getting ACK, the same incoming event might be processed more than once.
2. **Minimal overhead for volatile queries** - Queries using in-memory indexes should ideally have zero overhead - no sequence tracking, no checkpoint writes and no behavioral changes.
3. **Minimal cost on hot-path** - One extra key-value write per source-change event processed. This too will piggy-back on an already-open RocksDB/Garnet transaction. No extra `fsync`.
4. **Source-agnostic protocol** - The recovery protocol should work uniformly for log-tailing sources (Postgres, Kafka) and transient sources (HTTP, gRPC) with an optional local WAL.
5. **Progressive disclosure** - The default configuration (in-memory indexes, no WAL) should work as they do today. Users opt into durability incrementally.

---

## 3. Architecture Overview

### Data Flow

Current flow (at-most-once):

```
Source                          Query
  │                               │
  │  dispatch(event)              │
  ├──────── mpsc/broadcast ──────►│ priority queue
  │                               │ dequeue
  │  (no feedback)                │ process_source_change
  │                               │ commit to index
  │                               │
  │  periodic ACK to upstream     │
  │  (independent of query)       │
```

Proposed flow (at-least-once for persistent queries):

```
Source                              Query (lib layer)
  │                                   │
  │  dispatch(event, sequence=N)      │
  ├──────── mpsc/broadcast ──────────►│ priority queue
  │                                   │ dequeue
  │                                   │ if seq ≤ checkpoint: skip (dedup)
  │                                   │ session_control.begin()  <- outer transaction
  │                                   │ process_source_change(change)  <- core: nested begin/commit (no-ops)
  │                                   │ checkpoint_writer.stage(source_id, N)
  │                                   │ session_control.commit() <- actual commit:
  │                                   │   - index updates (written by core)
  │                                   │   - source_checkpoint:{source_id} = N (written by lib)
  │◄── position_handle.store(N) ──────│
  │                                   │
  │  read min(all position handles)   │
  │  ACK upstream = min(confirmed)    │
```

Key differences:

1. **Events carry a sequence number** stamped by the source (e.g., Postgres LSN).
2. **Queries persist a checkpoint per source** atomically with index updates via nested transactions. This checkpoint is written by the lib layer into the same database transaction that core uses for index writes.
3. **Queries write confirmed position** to a shared `Arc<AtomicU64>` handle.
4. Sources advance their cursor upstream only to the **minimum of all confirmed positions** for all subscribing queries.
5. **On restart**, queries read their checkpoint and subscribe with `resume_from`, sources replay from that position, queries skip already-processed events.
6. **No checkpoint tracking added in core**. The core crate (`ContinuousQuery`, `process_source_change`, `SessionControl` trait) will likely be left completely untouched. We are aiming to add the checkpoint logic  in lib and the index plugins, using nested transactions.

---

## 4. Key Design Decisions

### Checkpoint-based recovery over persistent queues

We considered three approaches:

| Option | Approach | Verdict |
|--------|----------|---------|
| **Persistent Queue** | Replace in-memory channel with disk-backed queue (RocksDB, Redis Streams) | Rejected because of unacceptable hot-path latency (fsync per event on the critical path), duplicates durability that already exists upstream |
| **Status Quo** | Accept at-most-once, re-bootstrap on crash | Rejected because of possible silent data loss, no recovery for processing failures |
| **Checkpoint Recovery** | Sources stamp sequences, queries persist checkpoints atomically, sources replay from confirmed position | **Selected** -> near-zero overhead, clean recovery, proven pattern (Flink, Kafka Streams) |

### Per-event checkpointing with unique LSNs

Each source-change event will have its own unique LSN. If the upstream system has transactions with 5 different events, with each having a different position in the upstream log then each source-change event generated as a result should get its own sequence number. 

When this event is processed successfully, the core atomically commits this checkpoint alongside updating its indexes.

For example, each WAL record in Postgres has its own unique LSN. A transaction with 5 inserts generates separate WAL records at different positions:

```
BEGIN        -> LSN 100
INSERT row 1 -> LSN 150
INSERT row 2 -> LSN 200
INSERT row 3 -> LSN 250
INSERT row 4 -> LSN 300
INSERT row 5 -> LSN 350
COMMIT       -> LSN 400
```

So a Postgres source must stamp each `SourceChange` with its **own** `end_lsn` from the XLogData header, and not the transaction's commit LSN. Since the query checkpoints after processing each event, If the query crashes mid-transaction (e.g., after processing 2 of 5 events), on recovery:
- The checkpoint points to LSN 200 (the second event's LSN).
- The source replays the transaction from the replication slot.
- The dedup filter skips events with `sequence ≤ 200` (events 1-2).
- Events 3-5 (LSNs 250, 300, 350) pass the filter and are processed.

This gives correct **per-event granularity** within transactions. Partial transaction replay is handled correctly: already-processed events are skipped, unprocessed events are reprocessed. No data loss, no duplication.

### Nested transactions for keeping core unchanged

The checkpoint must be written in the **same database transaction** as the index updates to guarantee atomicity.

However, the core engine's `ContinuousQuery::process_source_change()` owns the transaction internally. It calls `SessionGuard::begin()`, processes the change, and calls `SessionGuard::commit()`. Both the `SessionControl` and the `change_lock` are private fields with no public accessors.

Rather than modifying core's API (adding a checkpoint parameter to `process_source_change`), we will use **nested transaction support** in the index plugin implementations. This is a well-established pattern (e.g., SQL savepoints) where inner begin/commit calls are no-ops when an outer transaction is already active.

The lib layer will wrap core's processing in an outer transaction:

```
Lib:    session_control.begin()              <- creates the real DB transaction (depth=1)
Core:     session_control.begin()            <- nested, no-op (depth=2)
Core:     ...index writes...
Core:     session_control.commit()           <- nested, no-op (depth=1)
Lib:    checkpoint_writer.stage(src, seq)    <- writes into the still-open transaction
Lib:    session_control.commit()             <- actual commit (depth=0): index + checkpoint
```

This will need a few changes to each index plugin (`RocksDbSessionControl`, `GarnetSessionControl`) to add a nesting depth counter. A new `CheckpointWriter` trait defined in lib, implemented by the index plugins.

By using this approach, the core crate remains untouched. We prevent a big blast radius to `shared-tests`, `query-perf`, or `examples`, but that is not the main goal. The main goal here is to prevent leaking the concept of per-source checkpoints into the core engine.

### `Arc<AtomicU64>` position handles

The query needs to communicate its confirmed position back to the source. We selected a shared atomic over alternatives:

| Option | Mechanism | Verdict |
|--------|-----------|---------|
| Source polls query positions | Source holds `Arc<AtomicU64>`, reads periodically | Couples source to query internals |
| **Callback handle in SubscriptionResponse** | Source creates handle, passes in response, query writes | **Selected** -> natural extension of existing pattern |
| Dedicated feedback channel | `mpsc::Sender` in response | Overkill for one number |
| Shared registry in DrasiLib | Central `DashMap<source_id, DashMap<query_id, AtomicU64>>` | To-complicated |

The handle is created by the source during `subscribe()`, returned in `SubscriptionResponse`, and written by the query after each successful commit. The write is a single `store(seq, Relaxed)`. No syscall, no allocation, no async. The source reads all handles periodically to compute the min-watermark.

### Conditional activation

The checkpoint system activates based on the query's storage backend:

| Storage Backend | Checkpointing | Position Handle | `resume_from` |
|-----------------|---------------|-----------------|---------------|
| None / Memory   | Disabled      | Not created     | Always `None` |
| RocksDB         | Enabled       | Created         | Read from index |
| Garnet          | Enabled       | Created         | Read from index |

This will be determined once at query startup using `IndexBackendPlugin::is_volatile()`. No per-event branching.

---

## 5. Delivery Guarantees by Configuration

### Compatibility Matrix

The system enforces compatibility between query persistence and source durability at startup:

| Query Index | Source Type | Allowed | Guarantee |
|-------------|------------|---------|-----------|
| Volatile (Memory) | Any | ✅ | At-Most-Once |
| Persistent (RocksDB/Garnet) | Log-Tailing (Postgres, Kafka) | ✅ | At-Least-Once |
| Persistent (RocksDB/Garnet) | Transient + Durable WAL | ✅ | At-Least-Once |
| Persistent (RocksDB/Garnet) | Transient + No WAL | ❌ Error | Startup failure |

**Rationale for the rejection**: A persistent query against a volatile source creates an unrecoverable state on crash. The query retains stale index state, but the source cannot replay the missing events. The result is permanent, silent divergence.

**Mixed-source queries**: If a persistent query subscribes to multiple sources and any source is volatile (no WAL, not log-tailing), the subscription is rejected. Partial recoverability is worse than no recoverability as the index would contain replayed data from durable sources mixed with gaps from the volatile source.

**Enforcement**: The `QueryManager` checks `source.supports_replay()` for each source during the subscription phase. Error message includes the fix: enable WAL on the source or switch to a volatile index.

---

## 6. Component Responsibilities

### Sources ([doc 01: Source Sequencing and Replay](./01-source-sequencing-and-replay.md))

Sources are responsible for:
- **Stamping events** with a replayable sequence number (`sequence: Option<u64>` on `SourceEventWrapper`).
- **Supporting `resume_from`** in subscription settings, that will allow seeking to a position on subscribe when possible.
- **Managing position handles** by creating `Arc<AtomicU64>` per subscriber query, and tracking the min-watermark. That min-watermark will be used for ACKing upstream conservatively.
- **Log-tailing sources** (Postgres): Will use external LSN as sequence. Will use minimum confirmed position to advance position in upstream log.
- **Transient sources** (HTTP, gRPC) will optionally maintain a local WAL in state-store (like redb) with persist-then-ACK-then-dispatch ordering. When using WAL, they will apply backpressure when WAL is full, and this WAL will be pruning based on minimum of confirmed positions.

### Queries ([doc 02: Query Checkpointing](./02-query-checkpointing.md))

Queries will be responsible for:
- **Persisting checkpoints atomically** using nested transactions to write `source_checkpoint:{source_id} = sequence` into the same database transaction that core uses for index updates, via the `CheckpointWriter` trait.
- **Reading checkpoints on startup**. populating `resume_from` in subscription settings.
- **Dedup on replay**, filtering events where `event.sequence ≤ stored_checkpoint`.
- **Bootstrap-to-streaming handover**, consuming `BootstrapResult` metadata to set initial checkpoint and determine whether dedup applies to the buffered stream.
- **Conditional activation**, disabling all checkpoint machinery when the index is volatile.

### Orchestration ([doc 03: Orchestration and Recovery](./03-orchestration-and-recovery.md))

The orchestration layer (DrasiLib, QueryManager) will be responsible for:
- **Enforcing the Compatibility Matrix validation** at startup, rejecting invalid combinations.
- **Enforcing Recovery policy**, handling gap detection (source can't honor `resume_from`) per the configured policy (Fail, or bootstrap again).
- **Startup decision logic** for choosing between resume-from-checkpoint and fresh-bootstrap based on index state, checkpoint availability, and source capabilities.
- **Configuration validation** for warning on contradictory configs (e.g., `OverwriteOldest` WAL + `Strict` recovery policy).

---

## 7. Bootstrap-to-Streaming Handover

When a query starts fresh (no checkpoint) or falls back to bootstrap, there is a critical moment where bootstrap (snapshot) ends and streaming (live events) begins. The buffered live events in the priority queue may overlap with or be disjoint from the bootstrap data.

The `BootstrapProvider` returns handover metadata:

```rust
pub struct BootstrapResult {
    pub event_count: u64,
    pub last_sequence: Option<u64>,
    pub sequences_aligned: bool,
}
```

The handover behavior depends on whether the bootstrap and streaming source share the same sequence namespace:

| Scenario | Bootstrap | Stream | Handover | Action |
|----------|-----------|--------|----------|--------|
| **Homogeneous** (Postgres/Postgres) | LSN 500 | From LSN 0 | `last_sequence: 500, aligned: true` | Set checkpoint = 500. Filter buffered events ≤ 500. |
| **Heterogeneous** (File/Postgres) | No LSN | From LSN 500 | `last_sequence: None, aligned: false` | Set checkpoint = 0. Process all buffered events. |
| **Heterogeneous** (Postgres/HTTP-WAL) | LSN 10,000 | Counter 1 | `last_sequence: 10000, aligned: false` | Set checkpoint = 0. Ignore bootstrap LSN for filtering. |
| **Heterogeneous** (File/HTTP-WAL) | No LSN | Counter 1 | `last_sequence: None, aligned: false` | Set checkpoint = 0. Process all buffered events. |

**Default behavior**: If bootstrap provider and source are the same type, `sequences_aligned` defaults to `true`. If different types, defaults to `false`. Users can override for custom alignment.

We will design the full protocol in [doc 02: Query Checkpointing](./02-query-checkpointing.md).

---

## 8. Configuration Surface

### Query Configuration

```yaml
queries:
  - id: my_query
    query: "MATCH (o:Order) WHERE o.status = 'active' RETURN o"
    sources: [orders_db]
    storage_backend: rocks_persistent       # Activates checkpointing
    recovery_policy: strict                 # strict (default) | auto_reset
```

| Field | Values | Default | Description |
|-------|--------|---------|-------------|
| `storage_backend` | Name of registered backend, or omitted for in-memory | None (in-memory) | Determines whether checkpointing is active |
| `recovery_policy` | `strict`, `auto_reset` | `strict` | Behavior when source is unable to resume from requested checkpoint |

**`strict`**: If the source cannot honor `resume_from` (data pruned, slot invalidated), the query fails to start. Requires manual intervention (re-bootstrap, investigate cause).

**`auto_reset`**: Automatically clear the persistent index and perform a full bootstrap. Favors availability over consistency. The query resumes with a fresh state.

### Source Configuration (Transient Sources)

This applies to Transient sources only (the ones that do not have an upstream log, like Postgres or Kafka). Examples include HTTP and gRPC sources.

```yaml
sources:
  - id: webhook_source
    source_type: http
    durability:
      enabled: true                   # Enable local WAL (redb)
      max_size_mb: 512                # Capacity limit
      on_full: reject_incoming        # reject_incoming (default) | overwrite_oldest
```

| Field | Values | Default | Description |
|-------|--------|---------|-------------|
| `durability.enabled` | `true`, `false` | `false` | Enable local WAL for crash recovery |
| `durability.max_size_mb` | Integer | TBD | Maximum WAL disk usage |
| `durability.on_full` | `reject_incoming`, `overwrite_oldest` | `reject_incoming` | Behavior when WAL reaches capacity |

**`reject_incoming`**: Returns 503/429 to the caller. Propagates backpressure to the external producer. Preserves data safety.

**`overwrite_oldest`**: Deletes oldest events to make room. Favors availability. Creates gaps for slow consumers.

### Source Configuration (Log-Tailing Sources)

No new Drasi-side configuration. Relies on upstream retention settings (e.g., Postgres `wal_keep_size`, Kafka `retention.ms`). The source's confirmed position (via min-watermark) determines how much upstream log is retained.

### Default Experience

Out of the box, with no configuration changes:
- Queries use in-memory indexes -> no checkpointing, no recovery overhead.
- Sources don't enable WAL -> no fsync, no disk usage.
- Behavior is identical to today's at-most-once semantics.
- Zero performance impact.

Users opt into durability by setting `storage_backend` on queries and `durability.enabled` on transient sources. The system validates the combination and activates the checkpoint machinery automatically.

---

## 9. Recovery Flows

### Flow 1: Fresh Start (No Checkpoint)

```
Query starts
  -> Index is empty (or volatile)
  -> No checkpoint exists
  -> Subscribe with resume_from: None
  -> Source starts from current position
  -> Bootstrap (if enabled) provides initial snapshot
  -> Handover protocol sets initial checkpoint
  -> Stream processing begins
```

### Flow 2: Restart with Valid Checkpoint

```
Query restarts
  -> Open persistent index (RocksDB) that retains prior state
  -> Read checkpoint: source_checkpoint:{source_id} = N
  -> Subscribe with resume_from: N
  -> Source seeks to position N (replication slot, WAL, redb)
  -> Skip bootstrap (index already has the base state)
  -> Receive events starting from N
  -> Dedup: skip events where sequence ≤ N
  -> Resume normal processing
```

### Flow 3: Restart with Stale Checkpoint

```
Query restarts
  -> Read checkpoint: source_checkpoint:{source_id} = N
  -> Subscribe with resume_from: N
  -> Source cannot honor N (WAL truncated, slot invalidated)
  -> Source returns error
  -> Query consults recovery_policy:
      strict     -> Fail startup. Log error with guidance.
      auto_reset -> Clear all indexes. Subscribe with resume_from: None.
                   Full bootstrap. Fresh start.
```