# Adding Mesh/HA Features to SMG

SWIM gossip protocol with CRDT stores. Optional — only active with multiple gateway nodes.

## Architecture

```
crates/mesh/src/
  service.rs          → MeshServerBuilder, cluster state
  ping_server.rs      → SWIM gossip (60KB), message batching
  sync.rs             → MeshSyncManager, state reconciliation
  controller.rs       → Node state machine, membership
  stores.rs           → CRDT stores (WorkerState, PolicyState, RateLimitConfig)
  crdt_kv/            → CRDT implementations
  partition.rs        → Split-brain prevention
  node_state_machine.rs → Joining → Healthy → Suspect → Dead
```

## Steps

### Adding a New CRDT Store

1. Define CRDT type in `crates/mesh/src/crdt_kv/`
2. Register in `StateStores` (`crates/mesh/src/stores.rs`)
3. Add sync logic in `MeshSyncManager` (`crates/mesh/src/sync.rs`)
4. Emit updates in gossip messages (`crates/mesh/src/ping_server.rs`)
5. Version with `version: u64` for causality tracking

### Adding a Cluster Integration

1. Identify which state needs to be cluster-wide
2. Choose existing CRDT store or create new one
3. Emit state changes through gossip
4. Handle partition recovery (merge on reconnect)
5. Use mTLS for all cluster communication

## Existing CRDT Stores

| Store | Content | Sync |
|-------|---------|------|
| WorkerState | worker_id, url, health, load | On discovery/health change |
| PolicyState | policy name, config JSON | On config update |
| RateLimitConfig | max_rps, burst_size | On rate limit change |

## Node State Machine

```
Joining → Healthy → Suspect → Dead
              ↑         ↓
              └─────────┘ (recovery)
```

## Integration Points

- CacheAware routing: Radix tree ops sync across cluster
- Rate limiter: Global rate limit state shared
- Health checker: Worker health propagated to all nodes
- Service discovery: Workers found on one node appear on all
