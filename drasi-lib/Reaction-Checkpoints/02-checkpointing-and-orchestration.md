# Reaction Checkpointing and Orchestration

> **Parent**: [Reaction Recovery Overview](./00-reaction-recovery-overview.md)

---

## 1. Reaction Checkpoints

Reactions that multiplex over multiple queries track a separate `(sequence, config_hash)` per query. The `config_hash` detects query resets so a stale sequence is not trusted against a reconfigured query:

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ReactionCheckpoint {
    pub sequence: u64,
    pub config_hash: u64,
}

type ReactionCheckpoints = HashMap<String, ReactionCheckpoint>;  // query_id -> checkpoint
```

### Storage

For reactions with `is_durable = true`, checkpoints are persisted to the configured `StateStoreProvider` with a uniform layout:

- Partition: `store_id = reaction_id` (aligns with the existing `deprovision_common` convention).
- Key layout: one entry per subscribed query, keyed `"checkpoint:{query_id}"`.
- Value layout: bincoded `ReactionCheckpoint`.
- Startup hydration: `ReactionBase` uses `get_many` to load all of a reaction's checkpoints in a single round-trip. Read failure fails reaction startup rather than silently proceeding unchecked.

A reaction declaring `is_durable = true` requires the state store itself to report `is_durable() = true`. Startup rejects a durable reaction paired with the default in-memory store.

### Primitives

The framework does not own the event loop. Reactions continue to run their own processing tasks, each embedding a `ReactionBase` and spawning a loop around the per-reaction `priority_queue` as they do today. The framework's role is to provide the primitives reactions need for durability and bootstrap:

- **Checkpoint helpers on `ReactionBase`**: `read_checkpoint(query_id)`, `read_all_checkpoints()`, and `write_checkpoint(query_id, checkpoint)`. Serialization, key layout, and batched startup reads are handled internally.
- **Bootstrap hook on the `Reaction` trait**: `async fn bootstrap(&self, ctx: BootstrapContext) -> Result<()>`. The framework invokes this once after subscriptions are wired but before the bootstrap gate opens, on fresh start (when `needs_snapshot_on_fresh_start = true`) and on `AutoReset` gap recovery. Default implementation is a no-op. The reaction decides how to fetch and apply a snapshot or outbox, whether to wipe downstream state, and when to call `write_checkpoint`. Live events buffer in the priority queue until the hook returns.

`BootstrapContext` exposes access to `fetch_snapshot` and `fetch_outbox` on the relevant query, the checkpoint helpers above, and an `is_reset: bool` so the reaction can distinguish fresh bootstrap from `AutoReset` reconciliation. Snapshots are delivered as a stream so the reaction can paginate or transform at its own pace.

### Ordering

Reactions wanting at-least-once semantics call `write_checkpoint` after the side effect has committed. A crash between side effect and checkpoint replays the event on restart; writing before the side effect silently converts the reaction to at-most-once.

Write-failure behavior follows the reaction's `ReactionRecoveryPolicy`: `Strict` errors out, `AutoReset` retries with bounded backoff then errors out, `AutoSkipGap` logs and proceeds so the next successful write supersedes.

### Optional loop helper

For reactions that do not need custom scheduling, `ReactionBase` exposes an optional `run_standard_loop` helper that wraps dequeue, dedup against stored checkpoints, side-effect invocation, and checkpoint write in the correct order. Reactions call it with their side-effect closure. Reactions that need batching, timers, or bespoke backpressure skip the helper and write their own loops. High-throughput reactions typically call `write_checkpoint` at batch boundaries rather than per event, which cuts state-store write pressure by an order of magnitude.

## 2. Framework-Level Dedup

The `ReactionBase` orchestrator can automatically deduplicate results prior to invoking plugin code:

```rust
let result = priority_queue.dequeue().await;
if let Some(checkpoint) = checkpoints.get(&result.query_id) {
    if result.sequence <= checkpoint.sequence {
        continue; // Already processed
    }
}
// Pass to reaction processing
```

## 3. Reaction Archetypes & Recovery Strategies

Because Reactions serve different business purposes, the framework does not force a one-size-fits-all recovery flow. Reaction authors combine the provided tools (`fetch_snapshot`, `fetch_outbox`, and Recovery Policies) to match their specific archetype. 

Broadly, there are four types of reactions:

### 1. Maintain Derived View (State Synchronization)
These reactions replicate Drasi's query results into an external state (e.g., a downstream database like PostgreSQL or a Redis cache). They *require* `fetch_snapshot()` on initial startup to establish a baseline.
*   **1a. Expensive to Re-create**: If the downstream view is large or expensive to write, the reaction uses a **`Strict`** recovery policy. On disconnect, it uses `fetch_outbox()` to catch up. If a gap occurs, it fails safely to prevent data loss or a prohibitively expensive re-sync.
*   **1b. Cheap to Reconstruct**: If the downstream view is small/fast (e.g., an in-memory cache), the reaction uses an **`AutoReset`** policy. If a gap occurs, the reaction drops its state and cleanly rebuilds itself via `fetch_snapshot()`.

### 2. Trigger Events (Stateless Actions)
These reactions execute side-effects for *new* changes (e.g., firing a Webhook, sending an alert). They should **never** invoke `fetch_snapshot()` on startup, as that would trigger massive volumes of actions for historical data. On initial startup, they simply record the query's current sequence and listen to the live channel.
*   **2a. Guaranteed Delivery (At-Least-Once)**: For critical triggers, the reaction uses a **`Strict`** recovery policy and utilizes `fetch_outbox()` on resume to ensure no events are missed during downtime.
*   **2b. Transient Events (At-Most-Once)**: For non-critical updates (e.g., fire-and-forget logging), the reaction uses an **`AutoSkipGap`** policy. If it falls behind, it simply jumps to the current sequence and ignores the missed outbox history.

*(Why `AutoSkipGap` exists here but not for Sources: Missing a source event permanently corrupts the continuous query's internal graph state and math, making source-side skips catastrophically unsafe. However, downstream reactions merely consume the math. If a reaction drops a gap, it only misses intermediate frame updates and can safely snap to the newest correct output state, which is a mathematically sound strategy for systems prioritizing liveliness over exact history.)*

The framework provides the primitives (outbox, snapshot, recovery policies) but cannot determine a reaction's archetype at runtime. Reaction authors choose the appropriate policy for their use case, typically as a configurable default that users can override at deployment. Reactions use a `ReactionRecoveryPolicy` enum (`Strict`, `AutoReset`, `AutoSkipGap`), separate from the query-side `QueryRecoveryPolicy` (`Strict`, `AutoReset`), since `AutoSkipGap` does not make sense for persistent queries but can be allowed for some reactions.

Reactions expose their capabilities via trait methods, with safe defaults so that an author who forgets to override gets conservative behavior:

```rust
fn is_durable(&self) -> bool { false }
fn needs_snapshot_on_fresh_start(&self) -> bool { false }
fn default_recovery_policy(&self) -> ReactionRecoveryPolicy { ReactionRecoveryPolicy::Strict }
```

- `is_durable`: whether the reaction requires a durable state store to persist checkpoints across restarts. Validated against the configured state store at startup.
- `needs_snapshot_on_fresh_start`: whether the framework should call `fetch_snapshot` when the reaction starts with no prior checkpoint. `true` for reactions that maintain a derived view; `false` for reactions that only care about new events (skipping snapshot avoids firing side effects for the entire historical state).
- `default_recovery_policy`: policy used on a gap if the user does not override it via the reaction config's `recovery_policy` field.

Startup validation rejects invalid combinations:
- `is_durable=true` against a volatile query (outbox does not survive restarts).
- `is_durable=true` against a non-durable state store (checkpoints would be lost silently).
- `needs_snapshot_on_fresh_start=true` combined with `AutoSkipGap` (skipping a gap in state replication is silent corruption).
- `needs_snapshot_on_fresh_start=false` combined with `AutoReset` (reset has no snapshot baseline to reset to).

Reactions subscribed to multiple queries with heterogeneous per-query needs (mixed archetypes or recovery policies) handle the variance inside their `bootstrap` hook and custom loop; the declarative trait flags apply reaction-wide.

## 4. The Bootstrap Gate

A race condition exists between capturing initial query state (Snapshot/Outbox API) and polling the live channel where new events continuously buffer.

**Solution**: Handled by an `Arc<Notify>` (`bootstrap_gate`):
1. `ReactionManager::start_reaction` spawns the reaction's processing loop, which immediately `await`s the `bootstrap_gate` before dequeuing.
2. The manager connects live subscriptions (events queue up).
3. Manager performs per-query API calls (`fetch_snapshot` or `fetch_outbox`).
4. Manager sets the initial in-memory checkpoints based on API results.
5. `bootstrap_gate.notify_one()` fires.
6. Processing loop begins dequeuing buffered live events, deduplication filters overlapping sequences naturally.

## 5. Startup / Subscribe Flowchart

During startup, `ReactionManager` validates compatibility per the rules in §3. Then, for each query:

1. Check State Store for a valid `ReactionCheckpoint` (the `(sequence, config_hash)` tuple).
2. **If None Found**:
   - If `needs_snapshot_on_fresh_start` is `true`: execute `fetch_snapshot()` and set the checkpoint to the snapshot's `as_of_sequence` and `config_hash`.
   - If `false`: skip `fetch_snapshot` and set the checkpoint to the query's current sequence and `config_hash`.
3. **If Checkpoint Found & Hash Mismatch**:
   - The query was reconfigured, meaning its sequence space was reset. This is treated identically to a `PositionUnavailable` gap. The orchestrator immediately delegates to the configured `ReactionRecoveryPolicy`:
     - `Strict`: Fails startup to prevent data corruption.
     - `AutoReset`: Wipes the downstream view, calls `fetch_snapshot()` on the new query, and seamlessly builds the new baseline.
     - `AutoSkipGap`: Acknowledges the reset, updates its local `config_hash`, sets its sequence to the new query's current sequence, and continues processing on the live stream.
4. **If Checkpoint Found & Hash Matches(=N)**:
   - Execute `fetch_outbox(after: N)`.
   - If `Ok(OutboxResponse)`: Seamlessly process the buffered entries and open the live gate.
   - If `Err(PositionUnavailable)` (Gap Detected): Immediately execute the configured `ReactionRecoveryPolicy`. `Strict` fails the reaction startup; `AutoReset` wipes state and falls back to `fetch_snapshot()`; `AutoSkipGap` sets the checkpoint to the query's current sequence and continues on the live stream.

## 6. Broadcast Mode: Runtime Gaps

Using `broadcast` dispatch instead of `channel` guarantees non-blocking queries but risks buffer overflows under backpressure (`RecvError::Lagged`).

Sequence tracking allows the framework to detect these dropped events. When the reaction's processing loop sees a sequence jump in the live channel, the framework applies the same `ReactionRecoveryPolicy` used for startup recovery:

- `Strict`: Stop the reaction with an error.
- `AutoReset`: Call `fetch_snapshot()` and rebuild the downstream state inline.
- `AutoSkipGap`: Jump to the current sequence and continue processing.

This unifies gap handling: the recovery policy is the single declaration of "what does this reaction do when it misses events," regardless of whether the miss was caused by a crash, an outbox overflow, or a broadcast drop.

## 7. Query Lifecycle Protections

- **Update/Delete Query**: When a query configuration is updated, the orchestrator computes a new `config_hash` and resets internal execution sequences to `0`. Because reactions now store this `config_hash` alongside their sequence numbers, an old reaction checkpoint of `500` from an old hash won't erroneously discard the new iteration of events. Instead, the reaction safely detects the query reset and triggers its `AutoReset/Strict` recovery model to avoid silent data loss.
- **Stop Query**: Safely auto-stops subscribed reactions. Restarts require manual reaction starts. Sequence spaces are preserved.