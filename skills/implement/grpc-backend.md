# Adding a gRPC Backend Client to SMG

gRPC clients live in the `smg-grpc-client` crate (one file per engine) and talk to an inference backend (SGLang, vLLM, TRT-LLM, MLX, TokenSpeed). Each engine wraps its generated tonic client and shares connection/health/tokenizer logic via macros. The router layer (`model_gateway/src/routers/grpc/`) wraps all engines behind the `GrpcClient` enum and owns request building, streaming, and tool/reasoning parsing — the client crate does NOT parse output.

## Inputs to gather first

| Input | Example | Notes |
|-------|---------|-------|
| `ENGINE` | `myengine` | Snake case. File name `{ENGINE}_engine.rs` (or `_scheduler`/`_service`), client `MyengineEngineClient`, runtime key `"myengine"` |
| Proto package | `myengine.grpc.engine` | The `tonic::include_proto!` path. Generated client type lives under `proto::myengine_engine_client::...` |
| RPCs | `generate`, `embed`, `abort` | Engine-specific RPCs beyond the shared `health_check`/`get_model_info`/`get_server_info` |

## Steps

### Step 1: Create the client file

**File:** `crates/grpc_client/src/{ENGINE}_engine.rs`

Model this on `vllm_engine.rs`. Pull the generated client in via a `proto` module, hold it plus a `BoxedTraceInjector`, and call `impl_engine_client_basics!` for the connect/health/info boilerplate.

```rust
use tonic::{transport::Channel, Request};
use tracing::warn;

use crate::BoxedTraceInjector;

pub mod proto {
    #![allow(clippy::all, clippy::absolute_paths, unused_qualifications)]
    tonic::include_proto!("myengine.grpc.engine");
}

#[derive(Clone)]
pub struct MyengineEngineClient {
    client: proto::myengine_engine_client::MyengineEngineClient<Channel>,
    trace_injector: BoxedTraceInjector,
}

impl MyengineEngineClient {
    // connect / connect_with_trace_injector / with_trace_injector +
    // health_check / get_model_info / get_server_info
    crate::impl_engine_client_basics!(
        proto::myengine_engine_client::MyengineEngineClient<Channel>,
        "MyEngine"
    );

    crate::impl_get_tokenizer!();        // get_tokenizer() -> StreamBundle
    crate::impl_subscribe_kv_events!();  // subscribe_kv_events(start_seq)

    pub async fn generate(
        &self,
        req: proto::GenerateRequest,
    ) -> Result<tonic::Streaming<proto::GenerateResponse>, tonic::Status> {
        let mut client = self.client.clone();
        let mut request = Request::new(req);
        // Inject W3C trace context; failures are logged, not fatal.
        if let Err(e) = self.trace_injector.inject(request.metadata_mut()) {
            warn!("Failed to inject trace context: {}", e);
        }
        Ok(client.generate(request).await?.into_inner())
    }
}
```

`impl_engine_client_basics!` requires the macro's `proto::HealthCheckRequest` / `GetModelInfoRequest` / `GetServerInfoRequest` / response types to exist in your `proto` module. `impl_get_tokenizer!`/`impl_subscribe_kv_events!` use `common_proto` types and need the matching RPCs on the generated client.

If `generate` is streaming, prefer the auto-abort wrapper: return `crate::AbortOnDropStream<proto::GenerateResponse, Self>`, `impl AbortOnDropClient for MyengineEngineClient` (its `abort_for_drop` calls your `abort_request`), and build it with `AbortOnDropStream::new(stream, request_id, self.clone())`. The router calls `mark_completed()` on success. See `abort_on_drop.rs`.

### Step 2: Export from the crate root

**File:** `crates/grpc_client/src/lib.rs`

Add `pub mod {ENGINE}_engine;` and re-export: `pub use {ENGINE}_engine::{proto as myengine_proto, MyengineEngineClient};` (mirror the existing vLLM/SGLang lines). Shared infra already lives here: `channel.rs` (`connect_channel`, `normalize_grpc_endpoint`), `abort_on_drop.rs`, `tokenizer_bundle.rs`, and the `TraceInjector` trait (with `NoopTraceInjector` default + `BoxedTraceInjector` alias). Do not add a free trace-injection function — injection goes through the trait.

### Step 3: Wire into the router's GrpcClient enum

**File:** `model_gateway/src/routers/grpc/client.rs`

Add a `Myengine(MyengineEngineClient)` variant, an arm in `connect()` (`"myengine" => Ok(Self::Myengine(MyengineEngineClient::connect(url).await?))`), and arms in `health_check()`, `get_model_info()`, `get_server_info()`, and the `ModelInfo`/`ServerInfo` `to_labels()` matches. Request building, streaming, and tool/reasoning parsing live under `routers/grpc/regular/` and `utils/parsers.rs` — extend those, not the client crate.

### Step 4: Register in runtime detection

**File:** `model_gateway/src/workflow/steps/local/detect_backend.rs`

`detect_grpc_backend` tries `["sglang", "vllm", "trtllm", "tokenspeed", "mlx"]` in order via `do_grpc_health_check`. Add `"myengine"` to that array so `DetectBackendStep` recognizes the backend when no `runtime_type` is configured.

### Step 5: Add metadata discovery

**File:** `model_gateway/src/workflow/steps/local/discover_metadata.rs`

`fetch_grpc_metadata` connects via `GrpcClient::connect`, calls `get_model_info()`/`get_server_info()`, and merges `to_labels()`. `normalize_grpc_keys` renames engine fields to canonical label keys (e.g. `tensor_parallel_size -> tp_size`) and drops transient keys. Ensure your engine surfaces `served_model_name`, `model_path`, `context_length`, etc.; for SGLang-style configs the picked keys live in `SGLANG_GRPC_KEYS` in `client.rs`.

### Step 6: Tests

`vllm_engine.rs` keeps a `#[cfg(test)] mod tests` covering proto construction, defaults, and `connect("invalid://endpoint")` returning `Err`. Backend metadata tests live in `discover_metadata.rs` (the `#[ignore]` ones hit a live server). gRPC routing passes `SelectWorkerInfo { tokens: Option<&[u32]>, .. }` (a struct, not an enum variant) — cover token-aware policies there if relevant.

**Verify:** `cargo test -p smg-grpc-client` and `cargo test -p smg`

## Key Rules

- One file + one client type per engine; never a generic `mybackend.rs`. Reuse the three macros (`impl_engine_client_basics!`, `impl_get_tokenizer!`, `impl_subscribe_kv_events!`) — don't hand-roll connect/health/tokenizer.
- Inject trace context through `self.trace_injector.inject(request.metadata_mut())` on every outbound RPC; log-and-continue on error.
- Connect only via `connect_channel` (handles `grpc://`/`grpcs://` normalization + keep-alive); don't build a raw `Channel`.
- Tool/reasoning parsing is a ROUTER concern (`routers/grpc/...`), not the client crate.
- Targets tonic 0.14, package `smg-grpc-client`.
