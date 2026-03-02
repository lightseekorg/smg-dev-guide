# Adding a WASM Plugin Feature to SMG

Wasmtime component model with WIT interface. Plugins intercept requests/responses.

## Attachment Points

| Hook | When | Use Case |
|------|------|----------|
| `OnRequest` | Before routing | Auth, rate limiting, request modification |
| `OnResponse` | After backend response | Logging, response modification |

## Actions

| Action | Effect |
|--------|--------|
| `Continue` | Pass through unchanged |
| `Reject(status)` | Block request with HTTP status |
| `Modify(changes)` | Modify headers/body/status |

## Steps

### Step 1: Define types in WIT (if new interface needed)

**File:** `wasm/src/interface/spec.wit`

```wit
interface middleware-types {
  record request { method, path, query, headers, body, request-id, now-epoch-ms }
  record response { status, headers, body }
  variant action { continue, reject(u16), modify(modify-action) }
}
```

### Step 2: Add attachment point (if new hook)

**File:** `wasm/src/module.rs`

Add to `MiddlewareAttachPoint` enum.

### Step 3: Implement handler matching

**File:** `wasm/src/runtime.rs`

Match the new attachment point and execute WASM module.

### Step 4: Update module validation

**File:** `wasm/src/module_manager.rs`

### Step 5: Write example guest plugin

**Directory:** `wasm/examples/`

```rust
wit_bindgen::generate!({ world: "smg" });

struct MyPlugin;
impl Guest for MyPlugin {
    fn middleware_on_request(req: Request) -> Action {
        // Plugin logic
        Action::Continue
    }
    fn middleware_on_response(_req: Request, _resp: Response) -> Action {
        Action::Continue
    }
}
export!(MyPlugin);
```

### Step 6: Write tests

- Module upload via `/admin/wasm` API
- Execution at attachment point
- Each action variant (Continue, Reject, Modify)
- Memory limits (64MB) and timeouts (1s)

**Verify:** `cargo test`

## Key Rules

- Modules uploaded via admin API (requires Admin role)
- SHA-256 integrity verification on upload
- Memory limit: 64MB, timeout: 1s
- LRU component cache per worker thread
- `storage-hooks` feature flag for database interceptors
