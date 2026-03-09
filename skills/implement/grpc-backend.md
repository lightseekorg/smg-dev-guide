# Adding a gRPC Backend Client to SMG

gRPC clients connect to LLM backends (SGLang, vLLM, TRT). Use shared macros for common logic.

## Steps

### Step 1: Create client file

**File:** `crates/grpc_client/src/mybackend.rs`

Implement connection, health check, and inference methods. Use shared macros:
```rust
impl_tokenizer_methods!(MyBackendGrpcClient);
// Add other shared macros as applicable
```

### Step 2: Add trace injection

Every RPC call must propagate distributed tracing:
```rust
let mut request = tonic::Request::new(payload);
inject_trace_context(request.metadata_mut());
```

### Step 3: Register in runtime detection

**File:** `model_gateway/src/core/steps/worker/local/detect_runtime.rs`

Add detection logic so the 5-step worker lifecycle recognizes this backend.

### Step 4: Add metadata discovery

**File:** `model_gateway/src/core/steps/worker/local/discover_metadata.rs`

Map backend-specific fields to standard label keys:
```rust
// Backend returns "model_name" → map to "served_model_name" label
labels.insert("served_model_name", backend_info.model_name);
```

### Step 5: Handle streaming

```rust
let mut stream = client.chat_completion_stream(request).await?;
while let Some(chunk) = stream.message().await? {
    // Apply tool_parser / reasoning_parser if needed
}
```

### Step 6: Write tests

- Connection and health check
- Inference (non-streaming and streaming)
- Trace context propagation
- Metadata discovery maps to correct labels

**Verify:** `cargo test`

## Key Rules

- Use macros for shared logic (tokenizer, KV events) — don't duplicate
- Always inject trace context on RPC calls
- Map backend metadata to standard label keys (the label pipeline consumes them)
- gRPC routing uses `SelectWorkerInfo::Grpc { token_ids }` — ensure routing tests cover this
