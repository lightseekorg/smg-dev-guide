# Adding a Tool Parser to SMG

13+ formats supported. Most are JSON-based and can reuse `handle_json_tool_streaming()`.

## Steps

### Step 1: Create parser file

**File:** `tool_parser/src/parsers/myformat.rs`

```rust
pub struct MyFormatParser {
    buffer: String,
    partial_json: PartialJson,
    current_tool_id: i32,
    streamed_args_for_tool: Vec<String>,
}

#[async_trait]
impl ToolParser for MyFormatParser {
    async fn parse_complete(&self, output: &str) -> ParserResult<(String, Vec<ToolCall>)> { ... }
    async fn parse_incremental(&mut self, chunk: &str, tools: &[Tool]) -> ParserResult<StreamingParseResult> { ... }
    fn has_tool_markers(&self, text: &str) -> bool { ... }
    fn reset(&mut self) { ... }  // Clear buffer, tool_id, streamed_args
}
```

**Verify:** `cargo build`

### Step 2: Implement streaming (two-stage pattern)

Stage 1 — Name detected:
```rust
StreamingParseResult { items: vec![ToolCallItem { name: Some("func"), parameters: "" }] }
```

Stage 2 — Arguments delta:
```rust
// Calculate delta = current_args[previous_length..]
StreamingParseResult { items: vec![ToolCallItem { name: None, parameters: delta }] }
```

For JSON-based formats, use `helpers::handle_json_tool_streaming()` — handles 80% of the logic.

**Anti-pattern:** Not tracking `streamed_args_for_tool` — re-sends entire argument string each chunk.

### Step 3: Register in factory

**File:** `tool_parser/src/` (factory/registry)

```rust
registry.register_parser("myformat", || Box::new(MyFormatParser::new()));
registry.map_model("my-model*", "myformat");  // Wildcard pattern matching
```

**Verify:** `cargo build`

### Step 4: Export

**Files:** `tool_parser/src/parsers/mod.rs` and `tool_parser/src/lib.rs`

Add `pub mod myformat;` and re-export.

### Step 5: Write tests

**File:** `tool_parser/tests/tool_parser_myformat.rs`

Required test cases:
- Single tool call
- Multiple tool calls
- Empty arguments
- Streaming with chunks split at different boundaries
- Invalid tool name (should skip, not error)
- Mixed normal text and tool calls
- Unicode and special characters

**Verify:** `cargo test --test tool_parser_myformat`

## Key Rules

- Validate tool names against the `tools` list — skip invalid calls
- Reset state between **requests**, NOT between **chunks**
- Use `partial_json.parse_value()` for incomplete JSON
- Parsers are pooled with `Arc<Mutex<>>` — must be `Send + Sync`
