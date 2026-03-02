# Updating SMG Language Bindings

Required whenever config types, protocols, or public APIs change. The #2 contributor mistake after the config two-path rule.

## Python Bindings (PyO3 + maturin)

### Step 1: Add parameter to Router constructor

**File:** `bindings/python/src/lib.rs`

Add to `#[pymethods] impl Router`:
```rust
#[pyo3(signature = (..., my_field=None, ...))]
fn __init__(..., my_field: Option<String>, ...) {
    // ...
}
```

### Step 2: Update ALL struct literals (CRITICAL)

**File:** `bindings/python/src/lib.rs`

Search for every construction of the config struct you modified:
```bash
grep -n "DiscoveryConfig {" bindings/python/src/lib.rs
grep -n "RouterConfig {" bindings/python/src/lib.rs
```

Add `my_field: None` (or the passed value) to EACH occurrence.

**Anti-pattern:** Finding one literal and missing three others. The file has ~80 params and multiple code paths constructing these structs.

### Step 3: Add enum mapping (if new enum)

```rust
let my_type = match my_field_str.as_str() {
    "option_a" => MyType::OptionA,
    "option_b" => MyType::OptionB,
    _ => return Err(PyValueError::new_err("invalid my_field")),
};
```

### Step 4: Build and test

```bash
make python-dev
python -c "from smg import Router; r = Router(my_field='option_a')"
```

**Verify:** Build succeeds AND import works.

## Go SDK (cgo FFI)

### Step 1: Add to Go struct

**File:** `bindings/golang/` (appropriate struct in `client.go` or types file)

```go
type MyConfig struct {
    MyField string `json:"my_field"`
}
```

### Step 2: Add FFI function (if new functionality)

**File:** `bindings/golang/src/lib.rs`

```rust
#[no_mangle]
pub extern "C" fn sgl_my_function(...) -> ... { ... }
```

### Step 3: Bridge in Go

**File:** `bindings/golang/client.go`

```go
func (c *Client) MyFunction(...) ... {
    result := C.sgl_my_function(...)
    // ...
}
```

### Step 4: Test

```bash
cd bindings/golang && go test ./...
```

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Missing one Python struct literal | `maturin develop` fails with "missing field" |
| Python `None` default missing | TypeError at runtime |
| Go type mismatch with Rust FFI | Segfault or data corruption |
| Not testing import after build | Build succeeds but runtime import fails |
