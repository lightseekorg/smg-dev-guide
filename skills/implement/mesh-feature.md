# Adding Mesh/HA Features to SMG

Crate `smg-mesh` (`crates/mesh/`). SWIM-style gossip + CRDT replication. Optional — only matters with multiple gateway nodes. Mesh transport is application-agnostic: it moves opaque bytes keyed by prefix, never interprets them.

## Architecture

```
crates/mesh/src/
  service.rs          → MeshServerBuilder, MeshServerHandler; ClusterState = Arc<RwLock<BTreeMap<String, NodeState>>>
  gossip_controller.rs, gossip_service.rs → client/server sync loops (per-round send + ack)
  partition.rs        → PartitionDetector (quorum / split-brain)
  mtls.rs             → MTLSManager (cluster TLS)
  types.rs            → WorkerState (opaque value type)
  kv.rs               → MeshKV, CrdtNamespace, StreamNamespace, Subscription, DrainHandle
  proto/gossip.proto  → NodeStatus, StreamMessage, CrdtBatch/CrdtAck
  crdt_kv/            → CrdtOrMap router, OperationLog, MergeStrategy, CrdtWatermark, ReplicaId
    engine/           → per-namespace engines: lww.rs, rate_limit.rs
  transport/          → sync_stream.rs, crdt_batch.rs, chunk_assembler.rs, chunking.rs, limits.rs
```

`NodeStatus` (proto, `proto/gossip.proto`): `INIT → ALIVE → SUSPECTED → DOWN`, plus `LEAVING` (graceful shutdown). NOT a Joining/Healthy/Suspect/Dead machine.

## The common task: add a CRDT-replicated value

Mesh state is namespaced by key prefix (`worker:`, `policy:`, `rl:`, `config:`). To add a new replicated value, pick a `MergeStrategy` and configure a prefix — do NOT write a new engine.

1. Choose a `MergeStrategy` (`crdt_kv/merge_strategy.rs`):
   - `LastWriterWins` — higher `(timestamp, replica_id)` wins. Default for `worker:`, `policy:`, `config:`.
   - `EpochMaxWins` — rate-limit counters; raw put payload MUST be exactly 16 bytes `(epoch u64-be, count i64-be)` via `epoch_max_wins::encode`.
2. Configure the prefix once on the shared `MeshKV` (from `handle.mesh_kv()`):
   ```rust
   let ns = mesh_kv.configure_crdt_prefix("worker:", MergeStrategy::LastWriterWins);
   ```
   Fail-closed: configuring a prefix twice panics. `config:` is auto-registered at `MeshKV::new` — reach it via `mesh_kv.configs()`, never re-configure it.
3. Read/write through the `CrdtNamespace` handle (`kv.rs`) — keys must start with the prefix:
   ```rust
   ns.put("worker:7", bincode::serialize(&worker_state)?);   // opaque bytes
   let v = ns.get("worker:7");
   ns.delete("worker:7");                                     // tombstone
   ```
4. React to remote changes with `ns.subscribe(sub_prefix)` → `Subscription { receiver }`. Events are `(key, Option<Vec<Bytes>>)`; `None` = delete. Delivered for both local writes and remote merges, carrying the canonical post-merge value.
5. Replication is automatic — no gossip wiring needed. Each round `collect_round_batch()` snapshots the op-log; `transport/crdt_batch.rs` (`build_crdt_batches`) frames it under `MAX_MESSAGE_SIZE`; the peer's `dispatch_crdt_batch` merges via `MeshKV::merge_crdt_ops`. Merge is idempotent by op-id.

Model on the integration tests in `crates/mesh/src/tests/crdt_integration.rs` (LWW `worker:` and EpochMaxWins `rl:`).

## Send watermark / ack / retry

Per-peer, per-KEY send watermark (`crdt_kv/watermark.rs`, `CrdtWatermark`):
- Sender filters the op-log with `acked.allows(op)` (ts > acked version for that key) before batching — see `gossip_controller.rs` / `gossip_service.rs` send loops.
- Receiver returns `CrdtWatermark::from_ops` of what it merged; this rides back as a `CrdtAck` (`watermark_to_crdt_ack`) and the sender advances via `merge_max`.
- Un-acked keys (dropped batch, slow peer) are resent next round. Keying by key — not by author replica — means one lost op only delays that one key.

## Adding a new merge strategy (rare)

Only if neither LWW nor EpochMaxWins fits. Strategy logic lives entirely inside an engine; the router (`crdt_kv/crdt.rs`) stays strategy-agnostic.

1. Add a variant to `MergeStrategy` (`merge_strategy.rs`).
2. Implement `NamespaceCrdtEngine` (`crdt_kv/engine/mod.rs`) — byte-oriented `put_local` / `delete_local` / `get` / `apply_remote_ops` / `export_ops` / `gc_tombstones` / `generation`. Own your live state, per-key locks, `OperationLog`, and the shared `Arc<LamportClock>`. Model on `engine/lww.rs` (simplest) or `engine/rate_limit.rs` (typed state).
3. Wire the variant in `CrdtOrMap::register_merge_strategy` (`crdt.rs`) to construct your engine.
4. Compaction: call `OperationLog::compact_by_key(fold)` with your per-key fold; truncate-oldest (`truncate_oldest_over_threshold`, fires past `AUTO_COMPACT_THRESHOLD = 10_000`) is LOCAL-write path only — dropping on the remote-merge path sheds remotely-learned keys.

## Key rules

- App code uses `CrdtNamespace` / `StreamNamespace` handles, never `MeshKV` or `CrdtOrMap` directly.
- Values are opaque bytes — the mesh does not interpret them. Serialize at the edge (see `types.rs::WorkerState`).
- Op-id `(replica_id, timestamp)` must be globally unique per node — all engines share one `LamportClock`.
- `apply_remote_ops` must emit a `CrdtChange` only when `get` actually changes (suppress idempotent re-applies), or subscribers get spurious events.
- Ephemeral/lossy traffic (tenant deltas, tree repair) belongs in `StreamNamespace`, not CRDT — it is dropped under backpressure, not retried.

**Verify:** `cargo test -p smg-mesh`
