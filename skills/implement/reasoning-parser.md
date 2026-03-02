# Adding a Reasoning Parser to SMG

10+ model families supported. Most delegate to `BaseReasoningParser` — only add custom logic if needed.

## Key Decision: `initial_in_reasoning`

| Value | Behavior | Models |
|-------|----------|--------|
| `true` | Everything is reasoning until end token | DeepSeek-R1, Step3, Nemotron |
| `false` | Requires explicit start token | Qwen3, GLM-4, Kimi, MiniMax, Cohere |

## Steps

### Step 1: Create parser file

**File:** `reasoning_parser/src/parsers/mymodel.rs`

```rust
pub struct MyModelParser {
    base: BaseReasoningParser,
}

impl MyModelParser {
    pub fn new() -> Self {
        Self {
            base: BaseReasoningParser::new(ParserConfig {
                think_start_token: "<think>".to_string(),
                think_end_token: "</think>".to_string(),
                initial_in_reasoning: false,  // Choose based on model behavior
                stream_reasoning: true,
                max_buffer_size: 65536,
            }).with_model_type("mymodel".to_string()),
        }
    }
}

impl ReasoningParser for MyModelParser {
    fn detect_and_parse_reasoning(&mut self, text: &str) -> Result<ParserResult, ParseError> {
        self.base.detect_and_parse_reasoning(text)
    }
    fn parse_reasoning_streaming_incremental(&mut self, text: &str) -> Result<ParserResult, ParseError> {
        self.base.parse_reasoning_streaming_incremental(text)
    }
    fn reset(&mut self) { self.base.reset() }
    fn model_type(&self) -> &str { self.base.model_type() }
    fn is_in_reasoning(&self) -> bool { self.base.is_in_reasoning() }
}
```

Only add custom logic (override base methods) for non-standard token formats like Kimi (Unicode `◁think▷`) or MiniMax (auto-prepends `<think>`).

**Verify:** `cargo build`

### Step 2: Register in factory

```rust
registry.register_parser("mymodel", || Box::new(MyModelParser::new()));
registry.register_pattern("my-model", "mymodel");  // Case-insensitive substring
```

**Verify:** `cargo build`

### Step 3: Export

**Files:** `reasoning_parser/src/parsers/mod.rs` and `reasoning_parser/src/lib.rs`

### Step 4: Write tests

Required test cases:
- Streaming with partial tokens (e.g. `</` that doesn't complete `</think>`)
- Buffer overflow (>64KB reasoning block)
- Normal text / reasoning text separation
- Reset between requests (NOT between chunks)
- `initial_in_reasoning` behavior matches expectations

**Verify:** `cargo test`

## Critical: Partial Token Bug

When `</` appears but doesn't complete `</think>`, the system must not lose the `</` prefix. The `BaseReasoningParser` handles this correctly — verify in tests by sending `</` followed by non-`think>` text.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Wrong `initial_in_reasoning` | All output classified as wrong type |
| Resetting between chunks instead of requests | Loses buffer state mid-stream |
| Not delegating to `BaseReasoningParser` | Reimplements solved edge cases |
