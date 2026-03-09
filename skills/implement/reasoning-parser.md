# Adding a Reasoning Parser to SMG

## Collect These 4 Inputs First

Before writing any code, get these from the user:

| Input | Example | How To Decide |
|-------|---------|---------------|
| `MODEL_NAME` | `mymodel` | Snake case. Used as parser name, model_type, factory key |
| `PATTERNS` | `["my-model", "my-model-v2", "mymodel"]` | Case-insensitive substrings matched against model IDs. Check the provider's model catalog for all variant names |
| `TOKENS` | `<think>` / `</think>` | See **How to Find Reasoning Tokens** below |
| `INITIAL_IN_REASONING` | `true` or `false` | See **How to Determine initial_in_reasoning** below |

## How to Find Reasoning Tokens

There is no single canonical source. Check these locations in order:

### 1. Check existing implementations in vLLM and SGLang

**Fastest path.** These projects track reasoning tokens across all major models. If the model already has a parser there, just copy the tokens.

**vLLM** — https://github.com/vllm-project/vllm/tree/main/vllm/reasoning
- Registry: `__init__.py` → `_REASONING_PARSERS_TO_REGISTER` dict
- Each parser file defines `start_token` / `end_token` properties
- 17 models: deepseek_r1, deepseek_v3, qwen3, kimi_k2, glm45, step3, step3p5, minimax_m2, mistral, granite, ernie45, gptoss, hunyuan_a13b, olmo3, holo2, seed_oss, nano_v3

**SGLang** — https://github.com/sgl-project/sglang/blob/main/python/sglang/srt/parser/reasoning_parser.py
- Single file with all parsers and a `DetectorMap` dict mapping model names to detector classes
- Each detector class defines `think_start_token` / `think_end_token`

If the model exists in either project, use their tokens directly — they've already been validated against real model output.

### 2. HuggingFace `tokenizer_config.json` → `added_tokens_decoder`

Most reliable primary source. Download from `https://huggingface.co/{org}/{model}/raw/main/tokenizer_config.json` and search for thinking-related tokens.

**Example — Qwen3** (tokens 151667 and 151668):
```json
"151667": { "content": "<think>", "special": false },
"151668": { "content": "</think>", "special": false }
```

### 2. HuggingFace `tokenizer_config.json` → `chat_template`

The Jinja2 chat template often shows how reasoning tokens are handled during formatting.

**Example — DeepSeek-R1** (template strips reasoning from assistant messages):
```jinja2
{% if '</think>' in content %}
  {% set content = content.split('</think>')[-1] %}
{% endif %}
```

The `add_generation_prompt` section may also reveal the start token — DeepSeek-R1 appends `<think>\n` when generating.

**Example — Qwen3** (template wraps reasoning_content):
```jinja2
'<think>\n' + reasoning_content + '\n</think>\n\n' + content
```

### 3. HuggingFace model card / README

Some providers document the token format in their model card.

**Example — Cohere Command-A Reasoning:**
```
<|START_THINKING|>
[reasoning]
<|END_THINKING|>
```

Cohere also supports `reasoning=True` parameter in `apply_chat_template()`.

### 4. Provider API docs

Check the provider's API documentation for reasoning/thinking output format.

**Example — Together.ai, OpenRouter, etc.** often document thinking output as a separate `reasoning` field with its own delimiter format.

### 5. Send a test request and observe

Last resort. Send a reasoning-capable prompt through the provider's API and inspect the raw response tokens.

```bash
curl -s https://api.provider.com/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -d '{"model":"model-id","messages":[{"role":"user","content":"Think step by step: what is 17*23?"}]}' \
  | jq '.choices[0].message'
```

Look for the delimiters wrapping the chain-of-thought content.

### Known Token Formats

**In SMG today:**

| Model Family | Start Token | End Token | Notes |
|---|---|---|---|
| DeepSeek-R1 | `<think>` | `</think>` | `initial_in_reasoning=true` |
| Qwen3 | `<think>` | `</think>` | `initial_in_reasoning=false` |
| QwenThinking | `<think>` | `</think>` | `initial_in_reasoning=true` |
| Kimi | `◁think▷` | `◁/think▷` | Unicode tokens |
| GLM-4.5 | `<think>` | `</think>` | `initial_in_reasoning=false` |
| Step3 | `<think>` | `</think>` | `initial_in_reasoning=true` |
| NanoV3 (Nemotron) | `<think>` | `</think>` | `initial_in_reasoning=true` |
| MiniMax M2 | `<think>` | `</think>` | Model doesn't emit start token — SMG prepends it |
| Cohere Command | `<\|START_THINKING\|>` | `<\|END_THINKING\|>` | `initial_in_reasoning=false` |

**In vLLM/SGLang but NOT yet in SMG** (candidates to add):

| Model Family | Start Token | End Token | Notes | Source |
|---|---|---|---|---|
| Mistral | `[THINK]` | `[/THINK]` | Uses `mistral_common` special tokens | vLLM |
| Granite | regex: `Here's my thought process:` | regex: `Here's my response:` | Regex-based, not token-based | vLLM |
| Ernie 4.5 | `<think>` | `</think>` + `<response>`/`</response>` | Extra response delimiters | vLLM |
| GptOss | `<\|channel\|>analysis<\|message\|>` | `<\|end\|>` | Multi-token sequence | vLLM, SGLang |
| Hunyuan A13B | `<think>` | `</think>` | Standard tokens | vLLM |
| OLMo 3 | `<think>` | `</think>` | Standard tokens | vLLM |
| Seed OSS | `<think>` | `</think>` | Standard tokens | vLLM |
| Holo2 | `<think>` | `</think>` | Standard tokens | vLLM |

Most models use `<think>`/`</think>`. If the model you're adding uses those, you can usually just confirm via one of the sources above. For non-standard tokens (Mistral, Granite, GptOss), check the vLLM/SGLang implementations for the exact format.

## How to Determine `initial_in_reasoning`

This flag controls whether the parser assumes the first token is reasoning or normal text.

**Check the chat template's `add_generation_prompt` section:**
- If it appends the start token (e.g. `<think>\n`) → `initial_in_reasoning: true`
- If it appends nothing or just a role tag → `initial_in_reasoning: false`

**Or send a test request** and observe:
- If the very first output token is reasoning content (before any start token) → `true`
- If the first output is normal text or starts with explicit start token → `false`

| Value | Behavior | Models |
|-------|----------|--------|
| `true` | Everything is reasoning until end token appears | DeepSeek-R1, Step3, Nemotron, QwenThinking |
| `false` | Normal text until explicit start token | Qwen3, GLM-4.5, Kimi, MiniMax, Cohere |

## Step 1: Create parser file

**File:** `crates/reasoning_parser/src/parsers/{MODEL_NAME}.rs`

Generate this file, substituting the 4 inputs:

```rust
use crate::{
    parsers::BaseReasoningParser,
    traits::{ParseError, ParserConfig, ParserResult, ReasoningParser},
};

pub struct {ModelName}Parser {
    base: BaseReasoningParser,
}

impl {ModelName}Parser {
    pub fn new() -> Self {
        let config = ParserConfig {
            think_start_token: "{START_TOKEN}".to_string(),
            think_end_token: "{END_TOKEN}".to_string(),
            initial_in_reasoning: {INITIAL_IN_REASONING},
            stream_reasoning: true,
            max_buffer_size: 65536,
        };
        Self {
            base: BaseReasoningParser::new(config)
                .with_model_type("{MODEL_NAME}".to_string()),
        }
    }
}

impl Default for {ModelName}Parser {
    fn default() -> Self {
        Self::new()
    }
}

impl ReasoningParser for {ModelName}Parser {
    fn detect_and_parse_reasoning(&mut self, text: &str) -> Result<ParserResult, ParseError> {
        self.base.detect_and_parse_reasoning(text)
    }

    fn parse_reasoning_streaming_incremental(
        &mut self,
        text: &str,
    ) -> Result<ParserResult, ParseError> {
        self.base.parse_reasoning_streaming_incremental(text)
    }

    fn reset(&mut self) {
        self.base.reset();
    }

    fn model_type(&self) -> &str {
        self.base.model_type()
    }

    fn is_in_reasoning(&self) -> bool {
        self.base.is_in_reasoning()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_model_type() {
        let parser = {ModelName}Parser::new();
        assert_eq!(parser.model_type(), "{MODEL_NAME}");
    }

    #[test]
    fn test_reasoning_extraction() {
        let mut parser = {ModelName}Parser::new();
        let result = parser
            .detect_and_parse_reasoning("{START_TOKEN}thinking here{END_TOKEN}normal text")
            .unwrap();
        assert_eq!(result.reasoning_text, "thinking here");
        assert_eq!(result.normal_text, "normal text");
    }

    #[test]
    fn test_streaming_partial_token() {
        let mut parser = {ModelName}Parser::new();
        // Send partial end token — must not lose characters
        let r1 = parser
            .parse_reasoning_streaming_incremental("{START_TOKEN}partial</")
            .unwrap();
        let r2 = parser
            .parse_reasoning_streaming_incremental("not-a-token")
            .unwrap();
        // The "</not-a-token" should appear in reasoning (not lost)
        let combined = format!("{}{}", r1.reasoning_text, r2.reasoning_text);
        assert!(combined.contains("partial"));
        assert!(combined.contains("</not-a-token"));
    }

    #[test]
    fn test_streaming_multiple_chunks() {
        let mut parser = {ModelName}Parser::new();
        let _r1 = parser
            .parse_reasoning_streaming_incremental("{START_TOKEN}chunk1")
            .unwrap();
        let _r2 = parser
            .parse_reasoning_streaming_incremental("chunk2{END_TOKEN}")
            .unwrap();
        let r3 = parser
            .parse_reasoning_streaming_incremental("normal output")
            .unwrap();
        assert_eq!(r3.normal_text, "normal output");
    }

    #[test]
    fn test_reset() {
        let mut parser = {ModelName}Parser::new();
        parser
            .parse_reasoning_streaming_incremental("{START_TOKEN}thinking")
            .unwrap();
        assert!(parser.is_in_reasoning());
        parser.reset();
        assert!(!parser.is_in_reasoning());
    }

    #[test]
    fn test_initial_state() {
        let parser = {ModelName}Parser::new();
        // If initial_in_reasoning=true: parser starts in reasoning mode
        // If initial_in_reasoning=false: parser starts in normal mode
        assert_eq!(parser.is_in_reasoning(), {INITIAL_IN_REASONING});
    }

    #[test]
    fn test_empty_reasoning_block() {
        let mut parser = {ModelName}Parser::new();
        let result = parser
            .detect_and_parse_reasoning("{START_TOKEN}{END_TOKEN}normal")
            .unwrap();
        assert_eq!(result.normal_text, "normal");
    }
}
```

**Verify:** `cargo check -p reasoning-parser`

## Step 2: Register in module exports

**File:** `crates/reasoning_parser/src/parsers/mod.rs` — add:
```rust
pub mod {MODEL_NAME};
pub use {MODEL_NAME}::{ModelName}Parser;
```

**File:** `crates/reasoning_parser/src/lib.rs` — add to the `pub use parsers::{ ... }` block:
```rust
{ModelName}Parser,
```

**Verify:** `cargo check -p reasoning-parser`

## Step 3: Register in factory

**File:** `crates/reasoning_parser/src/factory.rs` — in `ParserFactory::new()`, add:

```rust
// Parser registration
registry.register_parser("{MODEL_NAME}", || Box::new({ModelName}Parser::new()));

// Pattern mappings — one per model ID variant
registry.register_pattern("{pattern-1}", "{MODEL_NAME}");
registry.register_pattern("{pattern-2}", "{MODEL_NAME}");
```

Pattern matching is **case-insensitive substring**: `model_id.to_lowercase().contains(pattern)`.

**Verify:** `cargo check -p reasoning-parser`

## Step 4: Run tests

```bash
cargo test -p reasoning-parser
```

All 7 tests in the new file plus all existing tests must pass.

## Step 5: Run full quality gate

Invoke `smg:contribute` to run fmt → clippy → test → bindings → commit format.

## When You Need Custom Logic

Most parsers are pure `BaseReasoningParser` wrappers. Only override methods when:

| Scenario | Example | What To Do |
|----------|---------|------------|
| Model always starts with reasoning but doesn't emit start token | MiniMax M2 | Prepend start token on first chunk (track `is_first_chunk` state) |
| Non-standard delimiters | Kimi (`◁think▷`), Cohere (`<\|START_THINKING\|>`) | Just change token strings in `ParserConfig` |
| Model needs preprocessing | Hypothetical | Override trait methods, call `self.base` after preprocessing |

For MiniMax-style prepending, add an `is_first_chunk: bool` field and override `parse_reasoning_streaming_incremental`:

```rust
fn parse_reasoning_streaming_incremental(&mut self, text: &str) -> Result<ParserResult, ParseError> {
    let modified = if self.is_first_chunk {
        self.is_first_chunk = false;
        format!("{START_TOKEN}{text}")
    } else {
        text.to_string()
    };
    self.base.parse_reasoning_streaming_incremental(&modified)
}
```

Don't forget to reset `is_first_chunk = true` in `reset()`.

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Wrong `initial_in_reasoning` | All output classified as wrong type | Test with real model output to observe whether first token is reasoning |
| Missing pattern variant | Model ID doesn't match, falls back to base | Check provider's model catalog for ALL variant names (aliases, versioned IDs) |
| Resetting between chunks instead of requests | Loses buffer state mid-stream | Only call `reset()` between separate API requests |
| Not delegating to `BaseReasoningParser` | Reimplements solved edge cases (partial tokens, buffer overflow) | Always compose `BaseReasoningParser` unless you have a very good reason |
| Forgetting `parsers/mod.rs` export | Compiles but factory can't find the type | Always update both `mod.rs` and `lib.rs` |
