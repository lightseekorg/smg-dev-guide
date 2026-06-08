# Adding Observability Features to SMG

Three pillars: Prometheus metrics (40+), OpenTelemetry tracing, structured logging via `tracing` crate.

## Adding Metrics

### Step 1: Describe at startup

```rust
describe_counter!("my_metric_total", "Description of what this counts");
describe_histogram!("my_latency_ms", "Description of what this measures");
```

### Step 2: Record on hot path with string interning

```rust
use crate::observability::metrics::intern_string;

// Dynamic labels MUST use intern_string to avoid allocation per request
let model = intern_string(&model_id);
counter!("my_metric_total", "model" => model).increment(1);
histogram!("my_latency_ms").record(elapsed_ms as f64);
```

**Anti-pattern:** Using raw strings for labels on hot paths — unbounded allocations, label cardinality explosion.

Static values for common cases (zero allocation):
```rust
status_code_to_static_str(200)  // → Some("200")
bool_to_static_str(true)        // → "true"
```

### Step 3: Verify exposure

Metrics exposed via Prometheus `/metrics` endpoint (served on port **29000** by default, set via `--prometheus-port`).

**Verify:** `curl localhost:29000/metrics | grep my_metric`

## Adding Tracing

The codebase records fields on the current span manually — it does NOT use `#[instrument]`, and `tracing::Span` has no `set_attribute` (that is OTel-only). See `model_gateway/src/middleware/logging.rs`.

```rust
use tracing::Span;

let span = Span::current();
// Fields must be declared (as Empty) when the span is created, then filled in:
span.record("model_id", req.model_id.as_str());
span.record("worker_url", url.as_str());
```

- OTel context is bridged in `model_gateway/src/observability/otel_trace.rs`; trace context propagates through gRPC metadata (see @grpc-backend.md)
- Real-time metrics are also pushed over a WebSocket (`observability/metrics_ws/`)

## Logging Rules

**Use `tracing`, NEVER `println!`/`eprintln!`** (clippy warns).

```rust
use tracing::{debug, info, warn, error};
info!(worker_url = %url, model_id = %id, "Worker registered");
warn!(error = %e, "Health check failed");
```

Module-specific levels:
```bash
RUST_LOG=smg::routing=trace cargo run
```

## Key Rules

- `intern_string()` for all dynamic label values
- `tracing` crate only, no `println!`
- Structured fields on spans, not string interpolation
- GaugeHistogram for in-flight tracking (see `gauge_histogram.rs`)
