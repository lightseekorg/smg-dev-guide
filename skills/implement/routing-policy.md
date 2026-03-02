# Adding a Routing Policy to SMG

All routing state must be `Send + Sync`. Must handle both HTTP and gRPC paths.

## Steps

### Step 1: Create policy file

**File:** `model_gateway/src/core/routing/my_policy.rs`

```rust
use async_trait::async_trait;

pub struct MyPolicy {
    // State must be Send + Sync (use DashMap, AtomicUsize, Arc)
}

#[async_trait]
impl RoutingPolicy for MyPolicy {
    async fn select_worker(
        &self,
        workers: &[Arc<dyn Worker>],
        info: &SelectWorkerInfo,
    ) -> Option<Arc<dyn Worker>> {
        let available: Vec<_> = workers.iter()
            .filter(|w| w.is_healthy() && w.circuit_breaker().can_execute())
            .collect();
        if available.is_empty() {
            return None;
        }
        // Selection logic here — handle BOTH variants:
        match info {
            SelectWorkerInfo::Http { prompt, .. } => { /* string-based */ }
            SelectWorkerInfo::Grpc { token_ids, .. } => { /* token-based */ }
        }
    }
}
```

**Verify:** `cargo build`
**Anti-pattern:** Using `.unwrap()` on empty worker slice — panics in production. Using `RwLock` on hot path — contention under load.

### Step 2: Register in factory

**File:** `model_gateway/src/core/routing/` (factory module)

Add enum variant and factory match arm.

**Verify:** `cargo build`

### Step 3: Add config (if needed)

Follow @config-plumbing.md for policy-specific parameters.

### Step 4: Write tests

- Empty worker slice → returns `None`
- All workers unhealthy → returns `None`
- Both `SelectWorkerInfo::Http` and `SelectWorkerInfo::Grpc` paths
- Concurrent access (spawn multiple tasks calling `select_worker`)

**Verify:** `cargo test`
**Anti-pattern:** Only testing HTTP path — gRPC breaks silently.

## Critical Rules

- **Never** `.unwrap()` on worker access
- **Always** filter `is_healthy() && circuit_breaker().can_execute()` before selection
- **Always** handle both `SelectWorkerInfo` variants
- Use `DashMap` for concurrent state, not `RwLock`
- Use `Arc::clone()` from the slice, never clone workers
