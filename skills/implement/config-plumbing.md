# Adding a Config Field to SMG

The #1 contributor mistake is wiring only one of two conversion paths. Follow every step.

## Steps

### Step 1: Add to DiscoveryConfig / RouterConfig

**File:** `model_gateway/src/config/types.rs`

```rust
#[serde(default, skip_serializing_if = "Option::is_none")]
pub my_field: Option<MyType>,
```

**Verify:** `cargo build`
**Anti-pattern:** Missing `#[serde(default)]` — existing YAML configs will fail to deserialize.

### Step 2: Update Default impl

**File:** `model_gateway/src/config/types.rs`

Add `my_field: None` (or appropriate default) to the `Default` impl for the struct.

**Verify:** `cargo build`

### Step 3: Add CLI flag

**File:** `model_gateway/src/main.rs` (CliArgs struct)

```rust
#[arg(long, value_parser = parse_my_type)]
pub my_field: Option<String>,
```

Add validation function:
```rust
fn parse_my_type(s: &str) -> Result<String, String> {
    MyType::parse(s).map_err(|e| e.to_string())?;
    Ok(s.to_string())
}
```

**Verify:** `cargo build`
**Anti-pattern:** No `value_parser` — invalid values accepted, crash at runtime instead of parse time.

### Step 4: Wire BOTH conversion paths (CRITICAL)

**File:** `model_gateway/src/main.rs`

Find BOTH functions and add the field:

```rust
fn to_router_config(...) -> RouterConfig {
    // ... add: my_field: cli.my_field.or(config.my_field),
}

fn to_server_config(...) -> ServerConfig {
    // ... add: my_field: cli.my_field.or(config.my_field),
}
```

**Verify:** `grep -n "my_field" model_gateway/src/main.rs` — must show assignments in BOTH functions.
**Anti-pattern:** Only wiring one path. CLI flag works but config file doesn't (or vice versa). This is silent — no error, just ignored.

### Step 5: Update Python bindings

**File:** `bindings/python/src/lib.rs`

1. Add parameter to `Router.__init__()`: `my_field: Option<String> = None`
2. Search file for ALL struct literals of the config type you modified
3. Add `my_field: None` to each literal

**Verify:** `make python-dev`
**Anti-pattern:** Missing one struct literal — build fails with inscrutable error about missing field.

### Step 6: Update Go SDK (if exposed)

**Files:** `bindings/golang/` — add to Go struct + FFI bridge if the field is user-facing.

**Verify:** Go tests pass.

### Step 7: Add tests

- Config deserialization test: YAML with and without the new field
- Validation test: invalid value rejected at parse time
- Update existing test struct literals that construct the modified config type

**Verify:** `cargo test`

## Common Mistakes

| Mistake | Consequence | Prevention |
|---------|-------------|------------|
| Only wiring `to_router_config()` | Config file value silently ignored | Always grep for BOTH functions |
| Missing `#[serde(default)]` | Existing configs break on deserialize | Every `Option<T>` field needs it |
| String instead of typed enum at runtime | Re-parsing on every request | Parse at boundary, store typed |
| Missing Python struct literal | `maturin develop` build failure | Search for ALL occurrences of struct name |
