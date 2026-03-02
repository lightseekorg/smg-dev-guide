---
name: map
description: Use when you need to understand the SMG codebase structure, find which crate owns a subsystem, or understand how crates depend on each other before making changes
---

# SMG Codebase Map

## What Is SMG?

High-performance Rust gateway for LLM inference backends. Routes requests to workers running vLLM, SGLang, TensorRT-LLM with 8 routing policies, KV cache optimization, K8s service discovery, WASM plugins, MCP tool execution, and mesh HA.

## Crate Map

| Crate | Role | Key Types |
|-------|------|-----------|
| `model_gateway` | Main binary. HTTP/gRPC handlers, routing engine, service discovery, observability, CLI | `RouterConfig`, `ServerConfig`, `CliArgs` |
| `protocols` | OpenAI-compatible types shared by ALL consumers (config, bindings, API). Sacred — no impl-specific fields. | `WorkerSpec`, `ModelCard`, `WorkerModels`, `ChatCompletionRequest/Response` |
| `kv_index` | KV cache-aware routing. Radix trees (String for HTTP, Token for gRPC), positional indexer | `StringTree`, `TokenTree`, `RadixTree` trait, `PositionalIndexer` |
| `auth` | API key (SHA-256 hashed), JWT/OIDC, role-based access (Admin/User), audit logging | `JwtConfig`, `ApiKeyEntry`, `Principal`, `Role` |
| `mesh` | HA cluster via SWIM gossip. CRDT KV store, partition detection, consistent hashing | `ClusterState`, `WorkerState`, `NodeStatus` |
| `wasm` | WebAssembly plugin system. WIT interface, middleware hooks (OnRequest/OnResponse), LRU cache | `WasmModule`, `Action` (Continue/Reject/Modify) |
| `mcp` | MCP protocol client. Tool discovery, execution, approval workflows, response format translation | `McpConfig`, `McpOrchestrator`, `ToolAnnotations` |
| `grpc_client` | gRPC client for backends. Macros for dedup, streaming, trace injection | `SglangGrpcClient`, `VllmGrpcClient` |
| `data_connector` | Pluggable storage: PostgreSQL, Oracle, Redis, in-memory. Hook system for interception | `StorageBackend` trait, `StorageHook` |
| `tool_parser` | 13+ tool call parsers (JSON, Mistral, Qwen, DeepSeek, Pythonic, etc.). Streaming with incremental JSON | `ToolParser` trait, `ParserFactory`, `StreamingParseResult` |
| `reasoning_parser` | Reasoning extraction from 10+ model families (DeepSeek-R1, Qwen3, Kimi, Cohere). Streaming | `ReasoningParser` trait, `ParserFactory`, `ParserResult` |
| `tokenizer` | LLM tokenization, chat templates | `Tokenizer` |
| `multimodal` | Image/audio processing. Vision processors (LLaVA, LLaVA-Next), media fetching | `ImageFrame`, `MultiModalInputs`, `ChatContentPart` |
| `workflow` | Step-based async workflow engine (wfaas) | `StepExecutor`, `WorkflowContext` |
| `bindings/python` | PyO3 bindings. `Router` class with ~80 constructor params, enum mapping | `Router`, `PolicyType` |
| `bindings/golang` | Go SDK via FFI (cgo). OpenAI-style API, streaming, tool calling | `Client`, `ChatCompletionRequest` |
| `clients/rust` | Rust client library | |

## Layering Rule

```
protocols (shared types — ALL consumers)
    ↑
model_gateway (implementation — ONE consumer writes each field)
    ↑
bindings/* (language SDKs — wrap model_gateway + protocols)
```

**Iron law**: If only one crate writes a field, it doesn't belong in `protocols/`. K8s-specific, runtime-specific, or gateway-specific fields stay in `model_gateway`.

## Config Propagation (3-Stage)

```
CLI args (main.rs CliArgs) + YAML file (RouterConfig)
    ↓ merge (CLI overrides file)
DiscoveryConfig / RouterConfig (config/types.rs) — serde-friendly, user-facing
    ↓ convert in main.rs (TWO paths: to_router_config + to_server_config)
ServiceDiscoveryConfig / ServerConfig — typed, runtime
```

**Both conversion paths** in main.rs must stay in sync. Miss one = CLI flag or config file silently ignored.

## Request Flow

```
Client → HTTP/gRPC handler → Auth middleware → WASM OnRequest
  → Routing policy selects worker → Proxy to backend
  → Stream response → Tool/reasoning parsing → WASM OnResponse → Client
```

## Worker Lifecycle (5-Step Workflow)

```
K8s Pod → PodInfo::from_pod() → handle_pod_event() → Job::AddWorker
  Step 1: Detect Runtime (sglang/vllm/trt)
  Step 2: Discover Connection Mode (HTTP/gRPC)
  Step 3: Discover DP Info (rank/size)
  Step 4: Discover Metadata → flattens into labels HashMap
  Step 5: Create Worker → merge labels, resolve model_id, build ModelCard
```

## The Label Pipeline

Central integration pattern. All worker metadata flows as key-value labels:
- **Source**: Backend HTTP endpoints (flattened JSON → HashMap)
- **Override**: WorkerSpec.labels from config (takes precedence)
- **Consumed**: create_worker.rs reads labels to build ModelCard
- **To inject metadata**: add as label — pipeline handles merging

## Essential Commands

```bash
cargo +nightly fmt --all                                      # Format
cargo clippy --all-targets --all-features -- -D warnings      # Lint
cargo test                                                     # Test
make python-dev                                                # Python bindings
make pre-commit                                                # All checks
```

## Next Steps

- **Implementing?** Use `smg:implement` — detects the subsystem and loads step-by-step recipes with verification.
- **Preparing to ship?** Use `smg:contribute` — enforces quality gates before PR.
- **Reviewing a PR?** Use `smg:review-pr` — systematic checklist mapped to changed subsystems.
