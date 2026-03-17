# Source Sequencing and Replay

> **Parent**: [Checkpoint-Based Recovery Overview](./00-checkpoint-recovery-overview.md)  

---

## 1. Overview

This document specifies the source-side changes required for checkpoint-based recovery in Drasi Lib. It covers:

- How sources stamp events with replayable sequence numbers.
- How sources advertise their durability capabilities.
- How sources track confirmed positions from subscribing queries via position handles.
- How the subscription protocol is extended to support `resume_from`.
- Detailed changes for log-tailing sources (Postgres) and transient sources (HTTP, gRPC) including the optional local WAL.
- Multi-query coordination and the min-watermark protocol.

We will try to keep all changes as conditional. Sources that do not support replay (transient without WAL) continue to operate as they do today with no overhead.

---

## 2. Sequence Stamping

### The `sequence` Field

A new optional field is added to `SourceEventWrapper`:

```rust
pub struct SourceEventWrapper {
    pub source_id: String,
    pub event: SourceEvent,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub profiling: Option<ProfilingMetadata>,
    pub sequence: Option<u64>,  // NEW - source-stamped replayable position
}
```

**Semantics**:

| Value | Meaning |
|-------|---------|
| `Some(n)` | This event has a replayable position. The source can replay from position `n` on request. |
| `None` | This event has no replayable position. The source is volatile. Checkpoint machinery ignores this event. |

Downstream code (query checkpoint, position handle updates) branches on `Some` vs `None` to decide whether to act.

### Log-Tailing Sources

For sources backed by an external durable log, the sequence can be the log's native position:

| Source | Sequence Value | Origin |
|--------|---------------|--------|
| Postgres | WAL LSN (`u64`) | `end_lsn` from `handle_xlog_data` in `stream.rs` |
| Kafka | Partition offset (`u64`) | Consumer offset |
| MSSQL | CDC LSN (`u64`) | Change tracking LSN |

Some data sources may have transaction-end LSN apart from each WAL record having its own unique LSN. We will stamp each `SourceChange` event with the respective WAL record's unique LSN and not the transaction commit LSN.

**Postgres transaction semantics**: The Postgres source batches changes within a WAL transaction (Begin -> changes -> Commit). Every WAL record has its own unique LSN - individual INSERT, UPDATE, and DELETE records within a transaction each have a different `end_lsn` in the XLogData header. Each `SourceChange` is stamped with its **own** `end_lsn`, not the transaction's commit LSN.

This gives correct per-event checkpoint granularity:
- If a query processes events 1 and 2 from a 5-event transaction and crashes, the checkpoint is at event 2's LSN (e.g., 200).
- On replay, events 1-2 (LSN ≤ 200) are skipped by the dedup filter. Events 3-5 are processed normally.
- No data loss, no duplication. Partial transaction replay is handled correctly.

**Risk:** It is possible that the Drasi Query results might be updated by some of the operations within a transaction. And this mid-way state might cause a Drasi reaction to trigger, without the transaction fully being applied to Drasi's indexes. This mid-transaction state will normally last for a very brief time interval, but in case of a crash it can persist until the query processing resumes. In either case, eventually the remaining part of the transaction will get processed and the end state of the index will reflect the complete transaction.

### Transient Sources with WAL

Transient sources (HTTP, gRPC, Application) that opt into durable mode maintain an internal monotonic counter persisted in the redb state store:

```
sequence = counter.fetch_add(1, Ordering::SeqCst)
```

The counter is:
- **Scoped per source ID** - persisted under key `source_sequence:{source_id}` in the state store.
- **Persistent across restarts** - on startup, the source reads the last counter value from redb and resumes from `last + 1`.
- **Reset on source removal** - when `delete_source(id, cleanup: true)` is called, the counter key is cleared as part of `deprovision()`. If a new source is created with the same ID, it starts from 0. This is safe because all subscribing queries are also removed (or their checkpoints are invalidated) when a source is deleted.

If a source is removed and re-added with the same ID *without* cleanup, the counter continues from its last value. This prevents sequence collisions with any surviving query checkpoints.

### Transient Sources without WAL

Sources with `durability.enabled = false` (the default) set `sequence = None` on all events. No counter is maintained. No state store interaction. Zero overhead.

### Bootstrap Events

`BootstrapEvent` does **not** carry a stream sequence number. The transition from bootstrap to streaming is handled by the handover protocol (see [doc 02: Bootstrap-to-Streaming Handover](./02-query-checkpointing.md)). The handover metadata (`BootstrapResult`) communicates the snapshot's position to the query, which uses it to set the initial checkpoint.

---

## 3. Source Durability Trait

Sources must advertise whether they support positional replay. A new method is added to the `Source` trait:

```rust
pub trait Source: Send + Sync {
    // ... existing methods ...

    /// Whether this source supports positional replay (resume from a sequence).
    ///
    /// Returns `true` for:
    /// - Log-tailing sources (Postgres, Kafka) that can seek in their upstream log.
    /// - Transient sources with WAL enabled (can replay from redb).
    ///
    /// Returns `false` for:
    /// - Transient sources without WAL (HTTP, gRPC with durability.enabled = false).
    ///
    /// Used by the orchestration layer to validate compatibility with persistent queries.
    fn supports_replay(&self) -> bool {
        false  // Conservative default
    }
}
```

Implementation by source type:

| Source | `supports_replay()` | Determined By |
|--------|---------------------|---------------|
| Postgres | `true` | Always (backed by WAL) |
| MSSQL | `true` | Always (backed by CDC) |
| Platform | `true` | Always (backed by platform log) |
| HTTP | `durability.enabled` | Runtime configuration |
| gRPC | `durability.enabled` | Runtime configuration |
| Application | `durability.enabled` | Runtime configuration |
| Mock | `false` | Always volatile (deferred until deterministic replay is needed for testing)|

The orchestration layer uses this during startup validation to enforce the compatibility matrix (see [doc 03](./03-orchestration-and-recovery.md)).

---

## 4. Position Handles

### Design

After a query commits a source event to its index, it needs to communicate its confirmed position back to the source. This is done via a shared atomic handle:

```rust
pub struct SubscriptionResponse {
    pub query_id: String,
    pub source_id: String,
    pub receiver: Box<dyn ChangeReceiver<SourceEventWrapper>>,
    pub bootstrap_receiver: Option<BootstrapEventReceiver>,
    pub position_handle: Option<Arc<AtomicU64>>,  // NEW
}
```

The `Option` for `position_handle` is `None` when:
- The source does not support replay (`supports_replay() == false`).
- The subscribing query uses a volatile index (no checkpointing - no point in tracking position).

The `bootstrap_receiver` and regular `receiver` handle different phases: a subscription either uses the `bootstrap_receiver` for a fresh start, or receives live/replayed traffic natively through the `receiver`. The query uses the presence of the bootstrap receiver or the `resume_from` capability to determine which gate to use (see ./02-query-checkpointing.md).

### Lifecycle

```
1. source.subscribe() is called
2. Source creates: let handle = Arc::new(AtomicU64::new(u64::MAX));
3. Source keeps a clone: self.position_handles.push(handle.clone());
4. Source returns handle in SubscriptionResponse
5. Query's processor task: after commit -> handle.store(sequence, Relaxed)
6. Source periodically: reads all handles, computes min-watermark
7. On query unsubscribe/drop: Arc refcount drops, source detects and removes
```

### Initialization Value

Position handles are initialized to `u64::MAX` (value meaning "not yet active"). This is necessary because:

- A newly subscribed query hasn't processed any stream events yet (it may be bootstrapping).
- A handle at 0 would immediately hold back the min-watermark to position 0, preventing the source from advancing upstream.
- `u64::MAX` is excluded from the min-watermark calculation (see below).

The handle transitions to a real value when the query commits its first stream event (post-bootstrap) via `handle.store(sequence, Relaxed)`.

### Min-Watermark Calculation

The source periodically computes the confirmed watermark:

```rust
fn compute_confirmed_position(&self) -> Option<u64> {
    let mut min_pos = u64::MAX;
    for handle in &self.position_handles {
        let pos = handle.load(Ordering::Relaxed);
        if pos < min_pos {
            min_pos = pos;
        }
    }
    if min_pos == u64::MAX {
        None  // No active subscribers, or all are bootstrapping
    } else {
        Some(min_pos)
    }
}
```

When the result is `None`, the source does **not** advance its upstream position. This is safe as the upstream log retains data, and the source will advance once a subscriber becomes active.

### Stale Handle Detection and Cleanup

A query may crash or be stopped without cleanly unsubscribing. Its position handle remains in the source's tracking list at its last written value, permanently holding back the min-watermark.

**Detection strategies** (use all three):

1. **`Arc::strong_count() == 1`**: If the source's clone is the only remaining reference, the query has dropped its copy. The handle is stale.

2. **Explicit cleanup in `stop_query` / `delete_query`**: When the query manager stops or deletes a query, it should signal the source to remove the corresponding handle. This can be done by the source tracking handles keyed by `query_id`, with a `remove_subscriber(query_id)` method.

3. **Periodic scan**: The source periodically (e.g., during the min-watermark computation) checks `Arc::strong_count()` and prunes dead handles. This catches cases where explicit cleanup was missed (e.g., query task panicked).

**Cleanup behavior**: When a stale handle is removed, the min-watermark may jump forward. This is correct - the stalled query is no longer a consumer, so the source can advance.

---

## 5. Subscription with `resume_from`

### Extended Subscription Settings

```rust
pub struct SourceSubscriptionSettings {
    pub source_id: String,
    pub enable_bootstrap: bool,
    pub query_id: String,
    pub nodes: HashSet<String>,
    pub relations: HashSet<String>,
    pub resume_from: Option<u64>,  // NEW - resume from this sequence
}
```

### Behavior

| `resume_from` | `enable_bootstrap` | Source Behavior |
|---------------|-------------------|-----------------|
| `None` | `true` | Fresh subscription. Current behavior - bootstrap + live stream. |
| `None` | `false` | Fresh subscription, no bootstrap. Stream from current position. |
| `Some(seq)` | `false` | Resume from position `seq`. Replay events from `seq` onward. Fails with `PositionUnavailable` if `seq` is missing. |
| `Some(seq)` | `true` | Resume from position `seq`. If `seq` is unavailable, fall back to a full snapshot bootstrap. Returns `bootstrap_receiver` in this fallback case. |

When `resume_from` is `Some(seq)`:
- The source seeks its upstream log (or local WAL) to position `seq` (via global min-watermark).
- If the position is available, the source starts streaming events from `seq` onward, and no bootstrap occurs.
- If the position is **unavailable** (WAL truncated, redb pruned, slot dropped):
  - If `enable_bootstrap` is `true`, the source automatically falls back to treating it as a fresh subscription, returning a `bootstrap_receiver`. The query must detect this (by receiving `bootstrap_receiver == Some` despite providing `resume_from`) and wipe its existing index state before processing the snapshot.
  - If `enable_bootstrap` is `false`, the source returns `SourceError::PositionUnavailable`. The caller then consults its `recovery_policy`.

### Error Handling

If the source **cannot honor** the requested `resume_from` (WAL truncated, replication slot invalidated, redb pruned past that position):

```rust
pub enum SourceError {
    // ... existing variants ...
    
    /// The requested resume position is no longer available.
    /// The caller should consult its recovery_policy to decide next steps.
    PositionUnavailable {
        source_id: String,
        requested: u64,
        earliest_available: Option<u64>,
    },
}
```

The `earliest_available` field helps the query (or operator) understand the gap size. The query consults its `recovery_policy` to decide whether to fail (`strict`) or wipe and re-bootstrap (`auto_reset`). See [doc 03](./03-orchestration-and-recovery.md) for the full recovery flow.

---

## 6. Log-Tailing Source Changes (Postgres)

### Split Read vs. Confirmed Position

The current `current_lsn` field in `ReplicationStream` is replaced with two tracking values:

| Field | Meaning | Used For |
|-------|---------|----------|
| `read_lsn` | Where the source has read to in the WAL | Tracking read progress, resuming after disconnect |
| `confirmed_lsn` | `min(all active position handles)` | ACKing upstream via `StandbyStatusUpdate` |

The `StandbyStatusUpdate` to Postgres is updated:

```rust
StandbyStatusUpdate {
    write_lsn: self.read_lsn,        // "I've received up to here"
    flush_lsn: self.confirmed_lsn,   // "My subscribers have durably processed up to here"
    apply_lsn: self.confirmed_lsn,   // Same as flush
    reply_requested: false,
}
```

This matches the Postgres replication protocol semantics precisely:
- `write_lsn` = received by standby
- `flush_lsn` = flushed to disk by standby
- `apply_lsn` = applied to standby's data

### Upstream ACK Flow

```
Source reads WAL event (LSN = 500)
  -> read_lsn = 500
  -> Dispatches event with sequence = 500 to channel

Query processes event, commits to index + checkpoint atomically (nested transaction)
  -> position_handle.store(500, Relaxed)

Source computes min-watermark
  -> confirmed_lsn = min(all handles) = 500

Source sends StandbyStatusUpdate
  -> flush_lsn = apply_lsn = 500

Postgres advances replication slot
  -> confirmed_flush_lsn = 500
  -> WAL before 500 can be reclaimed
```

**Critical invariant**: The source must never send a `flush_lsn`/`apply_lsn` greater than the confirmed watermark. This ensures Postgres retains WAL data that any subscriber might still need.

### No Local State Store Needed

Unlike transient sources, the Postgres source does **not** need to persist its checkpoint to a local state store (redb). The replication slot itself is the durable checkpoint - Postgres persists `confirmed_flush_lsn` as part of the slot metadata, and it survives Postgres restarts.

On startup, the source simply queries the slot:

```sql
SELECT confirmed_flush_lsn FROM pg_replication_slots WHERE slot_name = '...';
```

This returns the last position that was ACK'd via `StandbyStatusUpdate`, which equals the min-watermark at the time of the last successful feedback. The source resumes replication from that LSN.

This is simpler, more reliable, and avoids the write-ordering concerns that a local checkpoint would introduce (e.g., redb checkpoint drifting from the slot's actual `confirmed_flush_lsn`).

The same principle applies to other log-tailing sources where the upstream system natively tracks consumer position:

| Source | External Checkpoint | Query |
|--------|-------------------|-------|
| Postgres | `confirmed_flush_lsn` in `pg_replication_slots` | `SELECT confirmed_flush_lsn FROM pg_replication_slots WHERE slot_name = ?` |
| Kafka | Consumer group offset | Kafka tracks committed offsets natively |

The local redb state store is **only needed for transient sources** (HTTP, gRPC, Application) that have no external system to persist their checkpoint.

### Startup Flow and "Rewind" Mechanics

Sources start autonomously without waiting for query subscriptions. The replication slot's `confirmed_flush_lsn` serves as the initial starting point and retention lock.

```
Source starts
  -> Connect to Postgres
  -> Create or verify replication slot
  -> SELECT confirmed_flush_lsn FROM pg_replication_slots
  -> Start pumping replication from confirmed_flush_lsn into the unified channels
```

When a query subsequently subscribes with `resume_from: Some(seq)`:
1. The source evaluates if `seq` is actively available in the retained WAL (i.e., `seq >= confirmed_flush_lsn`).
2. If `seq` is **older** than the source's *current active read position* but still within the retained WAL, the source immediately severs its upstream replication connection and **rewinds**, restarting replication from `seq`. 
3. Events catch up through the unified channel. Fast queries seamlessly ignore duplicate events via their standard dedup filter.
4. If `seq` is older than `confirmed_flush_lsn` (data was pruned), the source immediately returns `SourceError::PositionUnavailable`.

### Handling Replication Slot Invalidation

If the replication slot has been dropped or its `confirmed_flush_lsn` has been advanced past what subscribers expect (e.g., Postgres restart with `max_slot_wal_keep_size` exceeded):

- `START_REPLICATION` will fail with an error.
- The source detects this during `connect_and_setup()`.
- On `subscribe()` with `resume_from`, the source returns `SourceError::PositionUnavailable`.
- The query consults its `recovery_policy`.

If the slot is completely gone, the source must recreate it. The new slot's `consistent_point` becomes the earliest available position. Any `resume_from` request before that point is `PositionUnavailable`.

---

## 7. Transient Source WAL (HTTP, gRPC, Application)

### Architecture

When `durability.enabled = true`, the source maintains a local write-ahead log backed by redb:

```
redb database (per source):
  -> events table:    sequence (u64) -> serialized SourceChange
  -> metadata table:  "counter" -> last sequence number (u64)
```

### Write Path

The ordering of operations is critical for correctness:

```
1. Receive event from external producer (HTTP request, gRPC call, etc.)
2. Generate sequence: N = counter.fetch_add(1) + 1
3. Persist: Write (N, event) to redb events table
4. Fsync: redb transaction commit (durable on disk)
5. ACK to producer: Return 200 OK / gRPC success
6. Dispatch: Send to in-memory channel with sequence = N
```

**Crash semantics at each step**:

| Crash Point | State | Recovery |
|-------------|-------|----------|
| After step 1, before step 3 | Event lost | Producer retries (because no ACK). Event is re-received. |
| After step 4, before step 5 | Event in redb, producer doesn't know | Producer retries. Source receives duplicate. Counter is ahead, so next event gets N+1. This will cause a duplicate source-change event - therefore at-least-once guarantee. |
| After step 5, before step 6 | Event in redb, ACK'd to producer, not dispatched | On restart, source replays from redb. Event is delivered. No loss. |
| After step 6 | Normal path | Checkpoint-based recovery if process crashes later. |

### Fsync Strategy

The default mode is **fsync per event** - each redb write transaction commits with an fsync before the ACK is sent to the producer. This provides the strongest guarantee but incurs 1-5ms latency per event on typical SSDs.

For the initial implementation, this is the only mode. A batched-fsync mode (accumulate N events or wait T ms, then fsync and ACK all) is deferred to future work as a latency-vs-throughput optimization.

### Startup Flow and "Rewind" Mechanics

When the source starts with an existing redb WAL, it starts autonomously without waiting for queries:

```
1. Read counter from metadata table -> resume counter from last value + 1
2. Source is ready to receive new HTTP/gRPC events.
```

When a query subsequently subscribes with `resume_from: Some(seq)`:
1. The source checks its redb WAL to see if `seq` is available.
2. If `seq` has already been pruned from the local WAL, the source simply returns `SourceError::PositionUnavailable`.
3. If the current read-head for the unified channel is ahead of `seq`, the source **rewinds** its read-head to `seq` and begins blindly pumping from the WAL into the standard receiver channels.
4. Other active queries seamlessly filter out older duplicate events via their local dedup logic until the stream catches up.
5. On subscribe with `resume_from: None`, the source returns the standard `SubscriptionResponse` with `bootstrap_receiver` + `receiver`.

**Unified Replay (no per-subscriber seeking):** Transient WAL sources use the exact same replay strategy as log-tailing sources. The source does not perform independent redb range scans per subscriber. There is only one read head pumping the WAL into the unified broadcast/channel framework.

### Retention and Pruning

Events in the redb WAL are retained until all active subscribers have confirmed past them:

```
prune_threshold = min(all confirmed position handles)
DELETE FROM events WHERE sequence ≤ prune_threshold
```

**Pruning frequency**: Piggyback on the min-watermark computation cycle. When the source reads position handles to compute the confirmed watermark, it also triggers pruning if the threshold has advanced since the last prune. This avoids a separate timer and ensures pruning is proportional to consumer progress.

### Backpressure and WAL Capacity

Since the WAL cannot be pruned until the slowest subscriber confirms, a stalled query can cause unbounded WAL growth. The following two mechanisms prevent disk exhaustion.

**Capacity limit**: Configured via `durability.max_size_mb`. A reasonable default (e.g., 256 MB) is mandatory - if not configured, the source uses the default. This is not optional: unbounded WAL growth in a system designed for durability would be a critical footgun.

**Behavior when full** (`durability.on_full`):

| Strategy | Behavior | Trade-off |
|----------|----------|-----------|
| `reject_incoming` (default) | Returns 503/429 to HTTP callers, RESOURCE_EXHAUSTED to gRPC callers, error to application callers | Preserves data safety. Propagates backpressure to external producer. No data loss. |
| `overwrite_oldest` | Deletes oldest events to make room, regardless of subscriber positions | Favors availability. Creates gaps for slow consumers. Affected queries will detect the gap and consult their `recovery_policy`. |

**Behavioral note for `reject_incoming`**: This changes the contract with external producers. An HTTP webhook source that previously always returned 200 will now sometimes return 503. Producers must handle retries. This is operationally correct but should be documented prominently in source plugin documentation.

### Polling Mode

Some sources like the HTTP one can operate in polling mode (source pulls data periodically) in addition to webhook mode (data pushed in). For polling mode:

- There is no external producer to ACK or reject - the source itself initiates the fetch.
- Backpressure means **stop polling**: when the WAL is at capacity, the source pauses its polling timer until space is available.
- The WAL still provides crash recovery: polled-but-unprocessed events survive restart and are replayed.
- The write path is the same: generate sequence -> persist to redb -> dispatch.

---

## 8. Multi-Query Coordination

### Min-Watermark Rule

When multiple queries subscribe to the same source, the source tracks all their position handles and computes:

```
confirmed = min(handle for handle in active_handles if handle != u64::MAX)
```

The source **must not** advance its upstream position (Postgres `flush_lsn`, Kafka commit, redb prune threshold) past this value. The slowest active subscriber gates upstream advancement.

### Upstream Impact

The min-watermark has direct operational impact on the upstream system:

| Source | Impact of Slow Subscriber |
|--------|--------------------------|
| Postgres | WAL retention grows. `pg_replication_slots.confirmed_flush_lsn` lags behind `pg_current_wal_lsn()`. Risk: `max_slot_wal_keep_size` exceeded -> slot invalidated. |
| Kafka | Consumer group offset lags. Topic retention may delete data past the offset. |
| Transient WAL | redb WAL grows toward `max_size_mb`. WAL full -> backpressure or overwrite. |

### New Query Joining a Running Source

When a new query subscribes to a source that is already streaming:

```
1. Source creates position handle initialized to u64::MAX
2. u64::MAX is excluded from min-watermark - new query doesn't hold back the source
3. Query receives SubscriptionResponse with receiver + bootstrap_receiver + handle
4. Query begins bootstrap (if enabled)
5. During bootstrap, live events buffer in the priority queue via the receiver
6. Bootstrap completes -> handover protocol sets initial checkpoint (doc 02)
7. Query starts processing buffered stream events
8. After first stream event commit: handle.store(sequence, Relaxed)
9. Handle now has a real value -> enters min-watermark calculation
```

**Key invariant**: The handle must not transition from `u64::MAX` to a real value until the query has actually committed a stream event. If the handle were set to the checkpoint from bootstrap, it could hold back the source to the bootstrap snapshot point, which may be far behind the current stream position.

### Query Removal

When a query is stopped or deleted:

```
1. Query task exits -> drops its Arc<AtomicU64> (refcount decreases)
2. Source detects Arc::strong_count() == 1 during periodic scan
3. Source removes the handle from its tracking list
4. Min-watermark recalculated without the removed query
5. If removed query was the slowest -> min-watermark jumps forward -> source can advance
```

Explicit cleanup during `stop_query` / `delete_query` is preferred over relying solely on refcount detection, to ensure timely cleanup. Both mechanisms should be implemented.

### Staggered Restarts and Heterogeneous Checkpoints

On process restart, multiple queries may subscribe to the same source incrementally with different `resume_from` values:

```
Query Q2 joins earliest: resume_from = 500
Query Q1 joins later: resume_from = 100
Query Q3 joins last: resume_from = None (fresh, needs bootstrap)
```

The unified source behavior accommodates queries joining at any time:

1. Source starts independently, connects to upstream (if applicable), and sits at min-watermark or end-of-log.
2. **Q2 joins.** Source evaluates 500. Rewinds its read head to 500 and starts streaming into the unified channel. Q2 processes events.
3. **Q1 joins.** Source evaluates 100. 100 is valid but earlier than the current stream position! Source severs its current stream, rewinds to 100, and restarts streaming.
4. **Q1** receives 100 onward. Processes all.
5. **Q2** receives 100 onward AGAIN, but its local dedup filter skips events ≤ 500 instantaneously. Processes events > 500.
6. **Q3 joins.** It is bootstrapping. Its handle is at `u64::MAX`. After bootstrap completes, it starts processing newly buffered events per the handover protocol.

The source does **not** wait for queries or maintain separate stream positions per subscriber. It simply "rewinds" its single unified read-head whenever a newly subscribing query requests an older valid position. The dedup filter on the query side ensures data correctness without breaking. This beautifully sidesteps any orchestrator startup race conditions or topological locking.

### Multi-Source Join All-or-Nothing Rule

When a single query spans multiple sources (a `JOIN`), the query's persistent state relies on consistent synchronization between all input streams. 
If the query requests `resume_from` for multiple sources, and **even one** source responds with `PositionUnavailable`, the query must revert to a unified recovery plan (e.g. `auto_reset`). 
The query manager must:
1. Signal the healthy sources to cancel the live replay (by dropping their `SubscriptionResponse` receivers).
2. Wipe the central RocksDB index for the query.
3. Resubscribe to ALL sources with `resume_from: None` to force a complete Multi-Source Bootstrap.

Mixing replay from one source with a fresh snapshot from another source for the same query leads to partial index state and silent data corruption. It is an **All-or-Nothing** requirement at the Query orchestration layer.

### Broadcast vs. Channel Dispatch Mode

The position handle and min-watermark protocol work identically in both dispatch modes. The differences are in backpressure behavior during replay:

| Aspect | Channel Mode | Broadcast Mode |
|--------|-------------|----------------|
| Replay events reaching slow query | Block in forwarder task (`enqueue_wait` on full priority queue) | May be dropped (`RecvError::Lagged` on broadcast overflow) |
| Effect on other queries | None (isolated channels) | All queries share the channel; slow subscriber can lag all |
| Priority queue enqueue | `enqueue_wait` (blocking, backpressure) | `enqueue` (non-blocking, drop on full) |
| Position handle behavior | Same | Same |
| Min-watermark behavior | Same | Same |

In Broadcast mode, events dropped due to `RecvError::Lagged` create a gap for the affected query. This gap is detected when the query sees a sequence jump and triggers its `recovery_policy`. This is an existing limitation of Broadcast mode, not introduced by this design.
