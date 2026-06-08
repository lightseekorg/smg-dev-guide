# Adding a Provider-Compatible API Surface to SMG

A "provider API" is a router that speaks a vendor's wire protocol on a dedicated route — Anthropic's Messages API on `/v1/messages`, Gemini's Interactions API on `/v1/interactions`. Each is a `RoutingMode` variant + a `RouterTrait` impl + a factory wiring + a mounted route. **Model new work on the Anthropic router** (`routers/anthropic/`): it is a flat module (`router.rs`, `streaming.rs`, `non_streaming.rs`, `sse.rs`, `mcp.rs`, `context.rs`, `worker.rs`, `utils.rs`), simpler than Gemini's step/driver design (`routers/gemini/{driver,state,steps/}`).

The gateway picks **one** router per process (`RouterFactory::create_router`, `factory.rs:51`), keyed on `connection_mode` then `mode`. Every other `RouterTrait` method falls through to a `501 NOT_IMPLEMENTED` default (`routers/mod.rs:55`), so a provider router only implements the one route method it serves (e.g. `route_messages`, `routers/mod.rs:211`).

> Worth checking first: is the surface already a `RouterTrait` method? `route_messages`, `route_interactions`, `route_responses`, `route_embeddings`, etc. already exist on the trait. If yes, you only implement that method on a router; you do **not** add a new trait method or route.

---

## The flow (Anthropic, end to end)

```
CLI --backend anthropic        →  RoutingMode::Anthropic { worker_urls }   (main.rs:1119, types.rs:308)
config validation              →  Anthropic mode: HTTP only, no discovery   (validation.rs:339,586)
RouterFactory::create_router   →  HTTP + Anthropic → create_anthropic_router (factory.rs:91)
create_anthropic_router        →  AnthropicRouter::new(ctx)                  (factory.rs:182)
server route  /v1/messages     →  v1_messages handler → route_messages(...)  (server.rs:903,310)
route_messages                 →  select worker, then streaming|non_streaming (router.rs:92)
```

---

## Step 1: Add the `RoutingMode` variant

**File:** `model_gateway/src/config/types.rs:293` — `enum RoutingMode` (`#[serde(tag = "type")]`). Add a variant mirroring `Anthropic`:

```rust
#[serde(rename = "myprovider")]
MyProvider { worker_urls: Vec<String> },
```

Adding a variant breaks every exhaustive match — the compiler lists them. Patch each like the `Anthropic` arm already present:
- `types.rs:329` `worker_count`, `types.rs:749` the mode-name `&str`.
- `config/validation.rs:339` (validate URLs; allow empty for dynamic workers) and `:586` (reject service discovery, as Anthropic/Gemini do).
- `config/builder.rs` — optional ergonomic helper like `gemini_mode()` (`builder.rs:94`); Anthropic has none and relies on the generic `mode()` (`builder.rs:99`).

## Step 2: Add the CLI backend + config→mode mapping

**File:** `model_gateway/src/main.rs:55` — `enum Backend`. Add `#[value(name = "myprovider")] Myprovider` and its `Display` arm (`main.rs:73`). Map it to the mode in `to_router_config` (`main.rs:1115`), next to the `Backend::Anthropic` branch (`main.rs:1119`):

```rust
} else if matches!(self.backend, Some(Backend::Myprovider)) {
    RoutingMode::MyProvider { worker_urls: self.worker_urls.clone() }
}
```

Then add a `RoutingMode::MyProvider { worker_urls } =>` arm to the `all_urls` match (`main.rs:1170`). `determine_connection_mode` (`main.rs:838`) returns `Http` unless a URL is `grpc://` — provider routers need HTTP (see Step 4). YAML config users select via `type: myprovider` (the serde tag); see @config-plumbing.md.

## Step 3: Implement the router

**Model file:** `model_gateway/src/routers/anthropic/router.rs`. Create `routers/myprovider/` with a `mod.rs` exporting the router (Anthropic's `mod.rs` re-exports `pub use router::AnthropicRouter;`, keeps `router` private, rest `pub(crate)`).

The router holds `Arc<AppContext>` plus a `RouterContext` of shared infra (`anthropic/context.rs:20`: `mcp_orchestrator`, `mcp_format_registry`, `http_client`, `worker_registry`, `request_timeout`). `new()` returns `Result<Self, String>` and fails fast if a required dependency is absent (`router.rs:55` errors when `mcp_orchestrator` is unset).

Implement `RouterTrait` (`#[async_trait]`) overriding **only** your route method + `as_any` + `router_type`. Shape (`router.rs:86`):

```rust
#[async_trait]
impl RouterTrait for MyProviderRouter {
    fn as_any(&self) -> &dyn Any { self }

    async fn route_messages(            // or route_interactions, etc.
        &self,
        headers: Option<&HeaderMap>,
        tenant_meta: &TenantRequestMeta,
        body: &CreateMessageRequest,    // from openai_protocol::messages (crates/protocols/src/messages.rs:24)
        model_id: &str,
    ) -> Response {
        // 1. (optional) MCP: if header_utils::is_smg_mcp_enabled(headers) && body.has_mcp_toolset()
        //    → mcp_utils::ensure_mcp_servers(...); 502 via routers::error::bad_gateway on failure
        // 2. select a worker for this provider
        // 3. dispatch streaming vs non-streaming
    }

    fn router_type(&self) -> &'static str { "myprovider" }
}
```

Worker selection (`router.rs:155`): `WorkerSelector::new(&registry, &client).select_worker(&SelectWorkerRequest { model_id, headers, provider: Some(ProviderType::Anthropic), ..Default::default() })`. `ProviderType` lives in `crates/protocols/src/worker.rs:265` (`OpenAI | XAI | Anthropic | Gemini | …`) — add a variant there if your provider needs distinct request shaping.

Streaming split (`router.rs:181`): `let is_streaming = body.stream.unwrap_or(false);` then call into separate modules. Anthropic's `streaming::execute` / `non_streaming::execute` take `(&RouterContext, RequestContext)` (`streaming.rs:36`, `non_streaming.rs:28`); `RequestContext` (`context.rs:29`) carries the cloned request, headers, `model_id`, `tenant_request_meta`, connected `mcp_servers`, and the pre-selected `worker`.

**Provider concerns** (split into sibling modules, like Anthropic):
- **SSE:** `sse.rs:57` `build_sse_response` sets `Content-Type: text/event-stream`. Match the vendor's event names exactly.
- **MCP tool interception:** when the request carries MCP toolsets, run an agentic tool loop instead of a plain proxy — Anthropic branches to `execute_mcp_streaming` (`streaming.rs:151`) / the MCP path in `non_streaming.rs`, capping at `mcp_utils::DEFAULT_MAX_ITERATIONS`. Per-server allowed tools come from `mcp::collect_allowed_tools_per_server` (`anthropic/mcp.rs:336`). See @mcp-feature.md.

## Step 4: Wire the factory

**File:** `model_gateway/src/routers/factory.rs`.
1. Add a `RouterId` const in `router_ids` (`factory.rs:34`), e.g. `pub const HTTP_MYPROVIDER: RouterId = RouterId::new("http-myprovider");`.
2. Add a `create_myprovider_router` mirroring `create_anthropic_router` (`factory.rs:182`): `Ok(Box::new(MyProviderRouter::new(ctx.clone())?))`.
3. In `create_router`, add a `RoutingMode::MyProvider { .. } =>` arm under **both** connection modes: the `Grpc` block returns `Err("MyProvider mode requires HTTP connection_mode")` (`factory.rs:71`); the `Http` block calls your factory (`factory.rs:92`).
4. Add the router to `create_igw_routers` (`factory.rs:212`) so multi-router IGW mode serves it.

Import your router in the `super::{...}` block at `factory.rs:5`.

## Step 5: Mount the route (only if it is a new endpoint)

`/v1/messages` and `/v1/interactions` are **already** mounted (`server.rs:903-904`) and dispatch via existing trait methods. For a genuinely new path:
1. Add an axum handler beside `v1_messages` (`server.rs:310`): `State<Arc<AppState>>`, `HeaderMap`, `Extension<TenantRequestMeta>`, a `PreemptionGuard`, and `ValidatedJson(body)`; call `state.router.route_*(...)` inside `cancel.guard(...)`.
2. Register it in the router builder: `.route("/v1/mypath", post(v1_mypath))` (`server.rs:903`).

The body type must be a `crates/protocols` request struct (e.g. `CreateMessageRequest`, `InteractionsRequest`) with `Deserialize` so `ValidatedJson` can parse it.

---

## Verify

```bash
cargo check -p smg          # smg == the model_gateway package
cargo test -p smg routers   # router-scoped tests
```

Then invoke `smg:contribute` for fmt → clippy → test → commit.

**Anti-patterns:** implementing many `RouterTrait` methods (override only what you serve — the rest 501 by design); allowing gRPC for an HTTP-only provider (return the `Err` in the `Grpc` block, `factory.rs:68`); inlining streaming + SSE + MCP in `route_messages` (keep them in sibling modules as Anthropic does); inventing a request struct in the router instead of using/extending `crates/protocols`.
