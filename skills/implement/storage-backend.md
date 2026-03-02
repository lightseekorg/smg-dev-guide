# Adding a Storage Backend to SMG

5 existing backends (Memory, None, PostgreSQL, Oracle, Redis). All share the same trait interface.

## Steps

### Step 1: Create module

**Directory:** `data_connector/src/mybackend/`

Implement all storage trait methods with consistent behavior across operations.

### Step 2: Handle schema migrations

- Version tracking table
- Idempotent migration scripts
- Rollback support where possible

### Step 3: Integrate hook system

Call storage hooks for audit trails and validation:
```rust
// on_write, on_delete hooks
hooks.on_write(&key, &value).await?;
```

**Anti-pattern:** Skipping hooks — breaks audit trail and multi-tenant data enrichment.

### Step 4: Add to backend factory

Wire into RouterConfig's backend selection enum and factory.

### Step 5: Support batch operations

All backends must support bulk read/write with pagination (cursor-based for large result sets).

### Step 6: Error handling

Wrap backend-specific errors in the common error type:
```rust
MyBackendError(#[from] specific_error) => StorageError::Backend(...)
```

### Step 7: Write tests and update bindings

- Integration tests with real backend (or test container)
- Batch operations, pagination, error cases
- Update Python bindings if exposed (follow @bindings-update.md)

**Verify:** `cargo test`

## Storage Hooks Feature

`storage-hooks` feature flag enables WASM modules as database interceptors. If your backend interacts with the WASM hook system, test with `--features storage-hooks`.
