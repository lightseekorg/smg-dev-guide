# Adding MCP Features to SMG

Model Context Protocol client for external tool servers. Manages discovery, execution, approval, and response translation.

## Architecture

```
crates/mcp/src/
  core/
    orchestrator.rs  → Tool execution, routing, validation (101KB)
    session.rs       → Server bindings, tool sessions
    config.rs        → McpConfig, trust levels
    pool.rs          → Connection pooling, reconnection
    oauth.rs         → OAuth flows
  approval/          → Approval system for destructive tools
  inventory/         → Tool storage, indexing, aliases
  transform/         → Response format conversion (OpenAI, Claude)
```

## Steps

### Adding a New Transport

Implement `rmcp::Transport` trait for the new connection type.

### Adding a Response Format

**Directory:** `crates/mcp/src/transform/`

Convert MCP tool results to API-compatible format (OpenAI function calling, Claude tool use, custom).

### Adding a Policy Rule

Extend `PolicyRule` with new condition patterns. Policy evaluation order:
1. Explicit tool policies (`"server:tool"`)
2. Server policies with trust levels
3. Default policy fallback

### Adding Custom Approval

Implement approval acceptance interface for destructive tool workflows:
1. Tool call detected as destructive (via annotations or policy)
2. Approval request created
3. Admin approves/denies via API
4. Execution proceeds or rejected

## Trust Levels

```rust
Trusted     // Allow all tools
Standard    // Allow non-destructive
Untrusted   // Require approval
Sandboxed   // Restricted execution
```

## Tool Annotations

```rust
ToolAnnotations { read_only, destructive, idempotent, open_world }
```

## Testing

Use `#[serial_test]` for approval workflow tests (shared state).

**Verify:** `cargo test -p smg-mcp`
