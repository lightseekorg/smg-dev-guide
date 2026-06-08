# Adding a Tenant-Aware Feature to SMG

Every serving request carries a canonical `TenantKey` (`Arc<str>` newtype, `model_gateway/src/tenant.rs`). To make a feature per-tenant you **read** the key that resolution already computed — never re-derive it from headers in your handler. To add a new identity source you extend the resolution flow, not consumers.

A `TenantKey` is the `Display` of a `TenantIdentity` variant with a prefix: `auth:<id>`, `header:<id>`, `ip:<addr>`, or `anonymous` (`TenantIdentity::into_key`, tenant.rs:22-31). `Explicit(key)` is passed through verbatim (admin target path).

## Resolution pipeline

```
auth_middleware (auth.rs:58-62)
    ↓ inserts DataPlaneCaller (authenticated_from_sha256) into request extensions
route_request_meta_middleware (tenant_resolution.rs:121-139)
    ↓ resolve_raw_tenant_key, in priority order (tenant_resolution.rs:58-73):
    ↓   1. DataPlaneCaller extension → its TenantKey (auth wins)
    ↓   2. trusted header (only if config.trust_tenant_header) → header:<id>
    ↓   3. ConnectInfo<SocketAddr> → ip:<addr>
    ↓   4. else → anonymous
    ↓ resolve_tenant_key: optional TenantAliasStore remaps key → canonical (skips expired/missing)
    ↓ inserts RouteRequestMeta into extensions
handler: Extension<TenantRequestMeta> → passes &TenantRequestMeta into RouterTrait methods
```

Layers in `server.rs` (~build_app, lines ~921-933) apply **bottom-up**: `auth_middleware` is added after `route_request_meta_middleware`, so auth runs **first** and its `DataPlaneCaller` is present when resolution runs. `TenantRequestMeta` is a type alias for `RouteRequestMeta` (`middleware/mod.rs:40`).

`route_request_meta_middleware` and `ordinary_tenant_resolution_middleware` are the **same code** — the latter is a thin async wrapper kept for backward-compatible naming (tenant_resolution.rs:143-149). Use `route_request_meta_middleware`.

## Task A: consume the tenant key in a feature

### Step 1: Read the key where you already have the request

In a **middleware**, pull it from extensions (model on `priority_admission_middleware`, admission.rs:100-104):
```rust
use crate::middleware::RouteRequestMeta;
use crate::tenant::TenantKey;

let tenant = req
    .extensions()
    .get::<RouteRequestMeta>()
    .map(|m| m.tenant_key().clone())
    .unwrap_or_else(|| TenantKey::new("anonymous"));
```
In a **handler**, take it as an extractor (server.rs:181, the `generate` handler) and forward it:
```rust
Extension(tenant_meta): Extension<middleware::TenantRequestMeta>,
// ...
state.router.route_generate(Some(&headers), &tenant_meta, &body, &body.model)
```
In a **RouterTrait method**, you already receive `tenant_meta: &TenantRequestMeta`; read `tenant_meta.tenant_key().as_str()` (router_manager.rs:611 passes it to skill resolution).

**Anti-pattern:** reading `x-smg-tenant-id` (or any auth header) directly in your handler. That bypasses the auth→alias precedence and the `trust_tenant_header` gate; the resolved key may differ from the raw header.

### Step 2: Key your per-tenant state on `TenantKey`

`TenantKey` is `Hash + Eq + Clone`, so use it directly as a map key. Model on `StaticTenantPolicyResolver` (`middleware/scheduler/policy.rs:30-77`): a `HashMap<TenantKey, T>` with a default fallback for unknown keys, built once at startup from config and looked up synchronously via a `Resolver` trait.

```rust
fn limit_for(&self, tenant: &TenantKey) -> Quota {
    self.per_tenant.get(tenant).copied().unwrap_or(self.default)
}
```

If you need to attach **derived per-request data** (not just read the key), append it to the meta with `RouteRequestMeta::with_extension` and read it back with `.extension::<T>()` (tenant.rs:117-132). Real use: router_manager.rs:618-624 attaches a resolved skill manifest onto the per-request meta. Do not add new public fields to `RouteRequestMeta`.

**Verify:** `cargo check -p smg`

### Step 3: Tests

Build a `RouteRequestMeta::new(TenantKey::from("auth:acme"))` (router_manager.rs:972-973 shows the test helper) and assert your feature reads the right key. For middleware, drive it with `from_fn_with_state` + `oneshot` and assert on the `auth:` / `header:` / `ip:` / `anonymous` key (tenant_resolution.rs tests, e.g. lines 213-268).

**Verify:** `cargo test -p smg`

## Task B: add a new `TenantIdentity` source

Only when a new principal type must produce keys. Two coordinated edits:

### Step 1: Add the variant + its key mapping

**File:** `model_gateway/src/tenant.rs` — add a variant to `TenantIdentity` (tenant.rs:11-18) and a matching arm in `into_key` (tenant.rs:22-31) with a **new unique prefix** (e.g. `apikey:`):
```rust
ApiKey(Arc<str>) => format!("apikey:{id}"),
```
Keep prefixes disjoint so two sources can never collide on one key string. `canonical_tenant_key(identity)` (tenant.rs:150) is just `identity.into_key()`.

### Step 2: Emit it during resolution

**File:** `model_gateway/src/middleware/tenant_resolution.rs` — add a branch in `resolve_raw_tenant_key` (tenant_resolution.rs:58-73), placed at the right **priority** (the function returns on the first match; `DataPlaneCaller` must stay first). Construct via `canonical_tenant_key(TenantIdentity::ApiKey(Arc::from(value)))`.

Header-sourced identities must stay behind the `state.trust_tenant_header` gate (tenant_resolution.rs:62) — an untrusted client can spoof a header. Auth-derived identities should instead be inserted upstream as a `DataPlaneCaller` in `auth_middleware` (auth.rs:58-62, via `DataPlaneCaller::authenticated_from_sha256` / `DataPlaneCaller::new(TenantKey)`).

### Step 3: Config (if gated/named)

A new trusted source usually needs a `TenantResolutionConfig` field (`config/types.rs:206-209`, `#[serde(default)]`). Plumb it per @config-plumbing.md and wire it through `TenantResolutionState::from_config` (tenant_resolution.rs:32-40). Validation lives in `config/validation.rs::validate_tenant_resolution`.

**Verify:** `cargo test -p smg`

## Step: contribute

Invoke `smg:contribute` to run fmt -> clippy -> test -> bindings -> commit.

## Critical Rules

- Read the resolved `TenantKey`; never re-parse `x-smg-tenant-id` (`DEFAULT_TENANT_HEADER_NAME`, tenant.rs:8) in a handler. Header trust is gated by `trust_tenant_header` and overridden by auth + alias remap.
- Precedence in `resolve_raw_tenant_key` is fixed: `DataPlaneCaller` (auth) > trusted header > client IP > anonymous. Adding a source means choosing where it sits, not appending blindly.
- Consumers must tolerate **any** key, including `anonymous` — always have a default-tenant fallback (`policy.rs:75` `unwrap_or(self.default)`; admission.rs:104 `unwrap_or_else(|| TenantKey::new("anonymous"))`). Resolution never fails to produce a key; it only 500s if the alias store errors (tenant_resolution.rs:128-135).
- `route_request_meta_middleware` == `ordinary_tenant_resolution_middleware` (alias). Don't apply both.
- New `TenantIdentity` variant ⇒ new `into_key` arm with a **disjoint prefix**. Forgetting the arm fails to compile (non-exhaustive match); a duplicate prefix silently merges tenants.
- Extend per-request data via `RouteRequestMeta::with_extension`/`.extension::<T>()`, not new public fields. The struct's `PartialEq` ignores extensions (tenant.rs:135-139).
- Admin/control-plane paths build the meta directly from an explicit id (`resolve_admin_target_tenant_id` → `TenantIdentity::Explicit`, tenant.rs:166-183); they do not flow through `resolve_raw_tenant_key`.
