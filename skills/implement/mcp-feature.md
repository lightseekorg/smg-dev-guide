# Adding MCP Features to SMG

`smg-mcp` (`crates/mcp/`) is the Model Context Protocol client: it discovers tools on external servers, gates execution behind an approval policy, and proxies calls. The crate is **OpenAI-protocol-free** — response-format adapter logic lives in `model_gateway::routers::common::openai_bridge`, not here (`lib.rs:2`). Built on `rmcp` 0.8.x (`Cargo.toml:35`).

Central type: `McpOrchestrator` (`core/orchestrator.rs`) owns the inventory, connection pool, and an `Arc<ApprovalManager>` built from `McpConfig` (`core/config.rs`).

```
crates/mcp/src/
  annotations.rs     → ToolAnnotations, AnnotationType
  core/  orchestrator.rs (central) · session.rs · config.rs · pool.rs
         oauth.rs · handler.rs · proxy.rs · reconnect.rs · metrics.rs
  approval/  manager.rs · policy.rs · audit.rs
  inventory/ index.rs · types.rs
```

The two realistic tasks are **customizing the approval policy** and **adding a transport**. Pick the matching section.

---

## Task A: Customize the approval policy

`PolicyEngine` (`approval/policy.rs:203`) decides per call. `evaluate()` (`:251`) checks, in order:
1. Explicit tool policy (`tool_policies`, keyed `QualifiedToolName`)
2. Server policy + `TrustLevel` (`evaluate_with_trust`, `:317`)
3. Pattern `rules` (`Vec<PolicyRule>`, evaluated in insertion order)
4. Annotation default (read_only→Allow, destructive→deny, else `default_policy`)

`PolicyDecision` (`:17`) is `Allow | Deny | DenyWithReason(Arc<str>)`. `TrustLevel` (`:95`) is `Trusted | Standard (default) | Untrusted | Sandboxed`.

### Add a config-driven server/tool policy

Edit YAML — no code. `from_yaml_config` (`policy.rs:438`) reads `default`, `servers`, and `tools` into the engine. Mirror the `policy:` block in `config.rs` tests (`core/config.rs:1116`):

```yaml
policy:
  default: allow
  servers:
    untrusted_server: { trust_level: untrusted, default: deny }
  tools:
    "dangerous_server:delete_all": deny
    "risky_server:format_disk": { deny_with_reason: "too dangerous" }
```

Config enums are the `*Config` mirrors (`TrustLevelConfig`, `PolicyDecisionConfig` in `config.rs:79,89`) with `From` impls at `policy.rs:404-434`. Tool keys MUST be `"server:tool"` — a bad key is logged and skipped (`policy.rs:461`).

**Anti-pattern:** expecting pattern `rules` from YAML. `PolicyRule` holds a `Regex` and has **no** serde derive; `from_yaml_config` never populates `rules`. Pattern rules are code-only (below).

### Add a pattern rule (regex / annotation-conditioned)

`PolicyRule { name, pattern: RulePattern, condition: RuleCondition, decision: PolicyDecision }` (`policy.rs:166`). `RulePattern` (`:125`) = `Server(Regex) | Tool(Regex) | Qualified(Regex) | Any`; `RuleCondition` (`:148`) = `Always | HasAnnotation(AnnotationType) | LacksAnnotation(AnnotationType)`. Add via the builder `with_rule` (`:245`), modeled on the `test_pattern_rule` test (`policy.rs:566`) and the `Default` impl (`:387`):

```rust
use smg_mcp::approval::{PolicyRule, RulePattern, RuleCondition, PolicyDecision};
use smg_mcp::AnnotationType;
use regex::Regex;

let engine = PolicyEngine::new(audit_log)
    .with_rule(PolicyRule::new(
        "block_delete",
        RulePattern::Tool(Regex::new("^delete_").unwrap()),
        RuleCondition::Always,
        PolicyDecision::Deny,
    ));
```

To make this reachable from config you'd extend `from_yaml_config` and `PolicyConfig` (see @config-plumbing.md) — currently it is wired only through the constructor.

`AnnotationType` (`annotations.rs:64`) = `Destructive | ReadOnly | Idempotent | OpenWorld`. `ToolAnnotations { read_only, destructive, idempotent, open_world }` (`:12`) come from rmcp with conservative defaults (destructive=true, open_world=true; `from_rmcp` `:25`). `should_require_approval()` = `destructive && !read_only` (`:57`).

**Verify:** `cargo test -p smg-mcp policy`

---

## Task B: Add a transport

Transports are the enum `McpTransport` (`core/config.rs:385`), tagged by `protocol` in YAML — `Stdio | Sse | Streamable`. Adding one = **add a variant, then patch every exhaustive `match` on it** (the compiler lists them):

1. `core/config.rs:385` — add the variant (with `#[serde]` fields).
2. `core/orchestrator.rs:513` `connect_server_impl` — build the rmcp transport and call `handler.serve(transport)`. Model on the `Sse` arm (`:547`): proxy via `super::proxy::resolve_proxy_config`, client via `build_http_client`, transport from `rmcp::transport::*`.
3. `core/orchestrator.rs:1350` `connect_dynamic_server_with_tenant` — dynamic path (Stdio is rejected here, `:1394`).
4. `core/orchestrator.rs:1292` `server_key` and `core/pool.rs:41` `PoolKey::from_config` — derive the pool key (url + `hash_auth(token, headers)`).

**Anti-pattern:** implementing an `rmcp::Transport` trait. SMG does not define transports as trait impls — it matches the `McpTransport` enum and hands an rmcp-provided transport (`SseClientTransport`, `StreamableHttpClientTransport`, `TokioChildProcess`) to `handler.serve()`. Also update the `Debug` impl (`config.rs:415`) so secrets stay redacted.

**Verify:** `cargo test -p smg-mcp` (transport parsing tests live at `config.rs:841`+).

---

## Testing

Tests sharing process env (proxy vars) use `#[serial]` from `use serial_test::serial;` (`config.rs:644,665`) — **not** `#[serial_test]`. Approval/policy tests construct `PolicyEngine::new(Arc::new(AuditLog::new()))` (`policy.rs:476`).

**Verify:** `cargo test -p smg-mcp`, then invoke `smg:contribute` for fmt → clippy → test → commit.
