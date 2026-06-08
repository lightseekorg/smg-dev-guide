# Conversation/Request Memory in SMG

Per-request memory **decisions** live in `model_gateway/src/memory/`. This is a thin,
internal layer that parses memory headers into a typed `MemoryExecutionContext` and
decides whether store/recall is `Active`. As of this writing the context is built on
every request but **not yet wired into store/recall execution** — the consumers are
TODO stubs. Be honest about that when extending: most realistic work is either adding a
policy variant (fully supported today) or wiring the dormant context into a route.

## How it works

```
x-conversation-memory-config (JSON header)
    │  MemoryHeaderView::from_http_headers   (routers/common/header_utils.rs:33)
    │    long_term_memory.enabled == false  → all fields treated as unset
    │    enabled == true, policy omitted     → defaults to "none" (privacy by default)
    ▼
MemoryExecutionContext::from_http_headers   (memory/context.rs:54)
    │    parse_policy(): store_only | store_and_recall | recall_only | none
    │                    → MemoryPolicyMode; unknown → Unrecognized (warns, no-op)
    │    gate each axis against MemoryRuntimeConfig.enabled
    ▼
MemoryExecutionContext { store_ltm, recall: MemoryExecutionState, policy_mode, … }
    │
    ▼  (intended consumers — currently TODO)
    routers/openai/context.rs:158   "Wire memory_execution_context into Responses … flow"
    routers/conversations/handlers.rs:405  "wire into ingestion flow; see issue #1149"
```

Two gates decide `MemoryExecutionState` (`memory/context.rs:15`):

| requested (policy) | runtime.enabled | state |
|--------------------|-----------------|-------|
| false              | any             | `NotRequested` |
| true               | false           | `GatedOff` |
| true               | true            | `Active` |

`requested()` is `!NotRequested`; `active()` is `== Active`. Use these, not field
matches, when you wire a consumer.

## Config & headers (the real surface)

- **Runtime master switch:** `MemoryRuntimeConfig { enabled: bool }`
  (`config/types.rs:16`), reachable as `router_config.memory_runtime`. Build via
  `RouterConfigBuilder::memory_runtime_config()` (`config/builder.rs:382`). When
  `enabled == false`, every requested axis collapses to `GatedOff`.
- **Per-request header:** `x-conversation-memory-config`, a JSON blob deserialized into
  `ConversationMemoryConfig` (`routers/common/header_utils.rs:340`). Relevant fields:
  `long_term_memory.{enabled, policy, subject_id, embedding_model_id, extraction_model_id}`.
  All strings are trimmed/normalized; empty → `None`.
- The context is constructed via the middleware helper
  `middleware::build_memory_execution_context(&router_config, &headers)`
  (`middleware/storage_context.rs:49`), called from `server.rs:463`,
  `routers/openai/context.rs:154,204`.
- **Scope today:** built for Responses flows; Chat sets
  `MemoryExecutionContext::default()` (`routers/openai/context.rs:191`).

## Relationship to ConversationMemoryWriter (separate half)

`ConversationMemoryWriter` (`crates/data_connector/src/core.rs:623`, `create_memory`) is
the **persistence** trait, plumbed through `AppContext` (`app_context.rs:68`) and the
routers (`routers/openai/router.rs:115`, `routers/grpc/router.rs:151`). It is wired but
**not yet called** anywhere in `model_gateway/src` — and it does not reference
`MemoryExecutionContext`. The decision layer (`memory/`) and the storage layer
(`ConversationMemoryWriter` + `memory.rs`/`memory_background.rs`) are the two halves of an
in-progress feature that are not yet connected. Wiring them is the big task below.

## Extension tasks

### A) Add a `MemoryPolicyMode` variant (supported end-to-end today)

This is the safe, complete change — the parse path is real and tested.

1. **Variant + raw string** — `memory/context.rs`. Add the user-facing case to the
   private `Policy` enum and its `from_value` match (e.g. `"recall_and_summarize" =>
   Self::RecallAndSummarize`), and the public `MemoryPolicyMode` variant. Map them in
   `parse_policy()` (`memory/context.rs:123`).
2. **Axis semantics** — update `Policy::allows_ltm_store` / `allows_recall`
   (`memory/context.rs:114,118`) so the new mode sets `store_ltm`/`recall` correctly.
3. **Test** — mirror the existing `#[cfg(test)]` cases (`memory/context.rs:150,164`):
   one for the header → state mapping, one for runtime-gated → `GatedOff`.

**Verify:** `cargo test -p smg memory::context`
**Anti-pattern:** Adding a `MemoryPolicyMode` variant but forgetting the `Policy` enum /
`from_value` — the header value then falls through to `Unspecified` → `Unrecognized`
(warns, no-op) and your mode is never selected.

### B) Wire the context into a route (the dormant consumer)

Pick the real TODO you're closing: Responses execution
(`routers/openai/context.rs:158`) or conversation-item ingestion
(`routers/conversations/handlers.rs:405`, param is `_memory_execution_context`).

1. The context is already on `RequestContext.memory_execution_context`
   (`routers/openai/context.rs:25`) and threaded into
   `process_item(..., _memory_execution_context)` — drop the `_` and branch on it.
2. Gate on `ctx.recall.active()` / `ctx.store_ltm.active()`; read `subject_id`,
   `embedding_model`, `extraction_model` for the call. Do nothing on `GatedOff` /
   `NotRequested` so the no-memory path stays zero-cost.
3. For persistence, call `conversation_memory_writer.create_memory(NewConversationMemory
   { … })` (`crates/data_connector/src/core.rs:624`). This is currently uncalled; you are
   the first caller — keep it behind the `active()` gate.
4. If headers can change mid-pipeline, recompute via
   `RequestContext::refresh_memory_execution_context()`
   (`routers/openai/context.rs:200`; Responses-only, else resets to default).

**Verify:** `cargo check -p smg`, then `cargo test -p smg`
**Anti-pattern:** Re-parsing `x-conversation-memory-config` in the route. Always consume
the prebuilt `MemoryExecutionContext`; the header→view→context parsing lives in exactly
one place.

## Key rules

- Parse once at the boundary; store typed `MemoryExecutionState`/`MemoryPolicyMode`, not
  raw header strings.
- Two independent gates: policy (per-request) **and** `memory_runtime.enabled` (global).
  An axis is `Active` only when both allow it.
- Default to no-op: missing/`enabled:false`/`none`/unrecognized policy must leave the
  common path untouched.
- The persistence half (`ConversationMemoryWriter`) is plumbed but not yet called — adding
  the first `create_memory` call is real work, not boilerplate.

## Step: contribute

Invoke `smg:contribute` to run fmt -> clippy -> test -> bindings -> commit.
