# Adding KV-Cache Index Features to SMG

Radix trees for cache-aware routing. Tracks which workers have which prompt prefixes cached.

## Core Data Structures

| Tree | Key Type | Used By |
|------|----------|---------|
| `StringTree` | `&str` (characters) | HTTP routing — prompt text prefix matching |
| `TokenTree` | `&[u32]` (token IDs) | gRPC routing — token sequence prefix matching |

Both implement `RadixTree` trait: prefix insertion, longest-prefix-match, LRU eviction, concurrent access (DashMap-based).

## Steps

### Adding Index Features

1. Implement in `kv_index/src/`
2. Ensure `Send + Sync` (accessed from routing hot path)
3. Support both String and Token variants if applicable
4. Add eviction/cleanup mechanism (prevent unbounded memory)
5. Consider mesh sync if state should be cluster-wide

### PositionalIndexer

Event-driven cache-aware routing at block level — finer granularity than tree-level prefix matching. Tracks which worker has which token blocks cached.

## CacheAware Integration

The `CacheAware` routing policy maintains per-model trees:
```
DashMap<model_id, Arc<StringTree>>   // HTTP
DashMap<model_id, Arc<TokenTree>>    // gRPC
```

Selection: longest-prefix-match → prefer cached worker (weighted by prefix length) → fall back to least-loaded.

## Eviction

Configurable interval (`cache_aware.eviction_interval_secs`). Entries not accessed within window are pruned via LRU.

## Key Rules

- All state must be `Send + Sync` (DashMap, not RwLock)
- Support both String and Token variants for HTTP/gRPC dual mode
- Always add eviction — unbounded trees cause OOM
- Test concurrent access from multiple routing tasks
