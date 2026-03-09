# SMG Anti-Patterns Reference

Per-subsystem anti-patterns to check during PR review.

## Config Plumbing

| Anti-Pattern | Consequence | What to Look For |
|-------------|-------------|------------------|
| Only wiring `to_router_config()`, missing `to_server_config()` | Field silently ignored in one code path | `grep -n "new_field" model_gateway/src/main.rs` — must appear in BOTH functions |
| Missing `#[serde(default)]` on new optional field | Existing YAML configs fail to deserialize | Check all new `Option<T>` fields in `config/types.rs` |
| No `value_parser` on CLI flag | Invalid values accepted, crash at runtime | Check `CliArgs` struct for new `#[clap]` fields |

## Worker Lifecycle & Labels

| Anti-Pattern | Consequence | What to Look For |
|-------------|-------------|------------------|
| Adding `_override` field to WorkerSpec | Bypasses label pipeline, creates parallel data path | New fields on `WorkerSpec` in `crates/protocols/src/worker.rs` |
| Post-hoc ModelCard mutation | Race conditions, stale data in routing | `model_card.model_id = ...` after `build_model_card()` |
| Injecting K8s-specific data into `crates/protocols/` types | Tight coupling to K8s, breaks non-K8s deployments | New fields in `crates/protocols/` that reference namespaces, pods, labels |

## Routing

| Anti-Pattern | Consequence | What to Look For |
|-------------|-------------|------------------|
| Only testing HTTP path, missing gRPC | Feature breaks for gRPC backends | Test files that only use `RequestType::Http` |
| Using `RwLock` on hot routing path | Contention under load | `RwLock` in routing policy structs (use `DashMap` instead) |
| `.unwrap()` on empty worker slice | Panic when no workers available | `workers[0]` or `.unwrap()` on worker selection |
| Skipping circuit breaker check | Routing to unhealthy backends | Missing `w.is_healthy() && w.circuit_breaker().can_execute()` |

## Parsers (Tool / Reasoning)

| Anti-Pattern | Consequence | What to Look For |
|-------------|-------------|------------------|
| Not resetting parser state between requests | Stale buffer from previous request bleeds into next | Missing `reset()` call or incorrect reset timing |
| Losing partial token prefix (e.g. `</` that isn't `</think>`) | Text silently dropped during streaming | Buffer handling when partial match fails |
| Missing factory registration | Parser unreachable at runtime | New parser not in `ParserFactory::new()` |

## Bindings

| Anti-Pattern | Consequence | What to Look For |
|-------------|-------------|------------------|
| Missing default in Python struct literal | `maturin develop` build failure | New config field without `new_field: None` in `bindings/python/src/lib.rs` |
| Missing Go type mapping | Go SDK compile failure | New enum/type not mirrored in `bindings/golang/` |

## Storage

| Anti-Pattern | Consequence | What to Look For |
|-------------|-------------|------------------|
| Missing hook integration | Audit trail gaps | New backend without `on_write` / `on_delete` hook calls |
| No schema migration | Data loss on upgrade | New fields without migration handling |
