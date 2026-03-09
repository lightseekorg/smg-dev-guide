# Adding a Tool Parser to SMG

## Collect These 4 Inputs First

Before writing any code, get these from the user:

| Input | Example | How To Decide |
|-------|---------|---------------|
| `PARSER_NAME` | `mymodel` | Snake case. Used as parser name, factory key, test file name |
| `MODEL_PATTERNS` | `["my-model*", "MyModel*"]` | Wildcard patterns matched against model IDs. Check the provider's model catalog for all variant names |
| `FORMAT` | See **Tool Call Format Types** below | The format the model uses to emit tool calls |
| `DELIMITERS` | `<tool_call>` / `</tool_call>` | Start/end tokens wrapping tool calls. See **How to Find Tool Call Format** below |

## How to Find Tool Call Format

### 1. Check existing implementations in vLLM and SGLang

**Fastest path.** These projects track tool call formats across all major models.

**vLLM** — https://github.com/vllm-project/vllm/tree/main/vllm/tool_parsers
- Registry: `__init__.py` → `_TOOL_PARSERS_TO_REGISTER` dict
- 34 parser files covering all major model families
- Each parser defines start/end tokens and format structure

**SGLang** — https://github.com/sgl-project/sglang/tree/main/python/sglang/srt/function_call
- 25 detector files, one per model family
- `function_call_parser.py` is the router

If the model exists in either project, match their format.

### 2. HuggingFace `tokenizer_config.json` → `chat_template`

The Jinja2 chat template shows how tools are formatted in the prompt AND how the model is expected to respond. Look for:
- Tool-related special tokens in `added_tokens_decoder` (e.g., `<tool_call>`, `<|tool_calls_section_begin|>`)
- Template sections handling `message.tool_calls` — shows the output format
- `tool_use` or `function_call` handling in the template

### 3. HuggingFace model card / README

Model cards often document the expected tool call output format with examples.

### 4. Send a test request with tools

```bash
curl -s https://api.provider.com/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -d '{
    "model": "model-id",
    "tools": [{"type":"function","function":{"name":"get_weather","parameters":{"type":"object","properties":{"city":{"type":"string"}}}}}],
    "messages": [{"role":"user","content":"What is the weather in Paris?"}]
  }' | jq '.choices[0].message'
```

Observe the raw format before the API normalizes it.

## Tool Call Format Types

| Type | Description | Parsers Using It | Reuse Strategy |
|------|-------------|------------------|----------------|
| **JSON with tags** | JSON wrapped in model-specific tags | Mistral `[TOOL_CALLS]`, Cohere `<\|START_ACTION\|>`, Step3 `<stepml:function_call>`, KimiK2 `<\|tool_call_begin\|>`, MiniMax `<FUNCTION_CALL>` | Extract between tags, delegate to `helpers::handle_json_tool_streaming()` |
| **XML with parameters** | XML tags with key-value parameter children | QwenCoder `<tool_call><function=..><parameter=..>`, GLM4 `<tool_call>..<arg_key>..<arg_value>` | Custom XML parsing |
| **Raw JSON** | Pure JSON object or array | JsonParser (OpenAI, Claude, Gemini) | Use `JsonParser` directly or register as `json` |
| **Pythonic** | Python function call syntax `[func(arg=val)]` | PythonicParser (Llama 4, DeepSeek R1) | Use `PythonicParser` directly |
| **Unicode tokens** | Full-width Unicode delimiters | DeepSeek V3 `<｜tool▁calls▁begin｜>` | Custom parser with Unicode handling |

**Most new models use "JSON with tags"** — wrap standard JSON in model-specific start/end tokens. This is the easiest to implement because `helpers::handle_json_tool_streaming()` handles 80% of the logic.

### Known Formats

**In SMG today:**

| Model Family | Start Token | End Token | Format | Parser |
|---|---|---|---|---|
| OpenAI, Claude, Gemini | `{` or `[` | `}` or `]` | Raw JSON | `json` |
| Mistral, Mixtral | `[TOOL_CALLS] [` | `]` | JSON array with prefix | `mistral` |
| Qwen 2/2.5/3 | `<tool_call>\n` | `\n</tool_call>` | JSON in XML tags | `qwen` |
| Qwen 3/2.5 Coder | `<tool_call>` | `</tool_call>` | XML with `<function=..><parameter=..>` | `qwen_coder` |
| Llama 3.2 | `<\|python_tag\|>` | JSON boundary | JSON with prefix tag | `llama` |
| Llama 4, DeepSeek R1 | `[` | `]` | Pythonic `func(arg=val)` | `pythonic` |
| DeepSeek V3 | `<｜tool▁calls▁begin｜>` | `<｜tool▁calls▁end｜>` | Unicode tokens + JSON code block | `deepseek` |
| GLM-4.5/4.6 | `<tool_call>` | `</tool_call>` | XML with `<arg_key>`/`<arg_value>` | `glm45_moe` |
| GLM-4.7 | `<tool_call>` | `</tool_call>` | XML (whitespace variant) | `glm47_moe` |
| Step-3 | `<stepml:function_call>` | `</stepml:function_call>` | JSON in XML tags | `step3` |
| Kimi K2 | `<\|tool_call_begin\|>` | `<\|tool_call_end\|>` | JSON in pipe-delimited tags | `kimik2` |
| MiniMax M2 | `<FUNCTION_CALL>` | `</FUNCTION_CALL>` | JSON in all-caps XML tags | `minimax_m2` |
| Cohere Command | `<\|START_ACTION\|>` | `<\|END_ACTION\|>` | JSON with `tool_name`/`parameters` fields | `cohere` |

**In vLLM/SGLang but NOT yet in SMG** (candidates to add):

| Model Family | Format | Source |
|---|---|---|
| Hermes 2 Pro | `<tool_call>` JSON `</tool_call>` (with optional `<scratch_pad>`) | vLLM, SGLang |
| Jamba | `<tool_calls>` JSON array `</tool_calls>` | vLLM |
| xLAM | Multiple: JSON code block, `[TOOL_CALLS]`, or XML | vLLM |
| FunctionGemma | `<start_function_call>call:name{args}<end_function_call>` | vLLM |
| Granite | `<\|tool_call\|>` or `<tool_call>` + JSON array | vLLM, SGLang |
| Phi-4 Mini | JSON format | vLLM |
| Seed OSS | `<seed:tool_call>` with `<function=..><parameter=..>` | vLLM |
| InternLM | Custom format | vLLM, SGLang |
| Hunyuan A13B | Custom format | vLLM |
| OLMo 3 | Pythonic format | vLLM |
| GigaChat 3 | Custom format | vLLM, SGLang |
| DeepSeek V3.1/V3.2 | Variant of DeepSeek V3 Unicode tokens | vLLM, SGLang |

## Step 1: Create parser file

**File:** `crates/tool_parser/src/parsers/{PARSER_NAME}.rs`

For the most common case — **JSON with tags** — generate this template:

```rust
use async_trait::async_trait;
use openai_protocol::common::Tool;
use serde_json::Value;

use crate::{
    errors::ParserResult,
    parsers::helpers,
    partial_json::PartialJson,
    traits::ToolParser,
    types::{FunctionCall, StreamingParseResult, ToolCall, ToolCallItem},
};

const START_TOKEN: &str = "{START_TOKEN}";
const END_TOKEN: &str = "{END_TOKEN}";

pub struct {ParserName}Parser {
    partial_json: PartialJson,
    buffer: String,
    prev_tool_call_arr: Vec<Value>,
    current_tool_id: i32,
    current_tool_name_sent: bool,
    streamed_args_for_tool: Vec<String>,
}

impl {ParserName}Parser {
    pub fn new() -> Self {
        Self {
            partial_json: PartialJson::default(),
            buffer: String::new(),
            prev_tool_call_arr: Vec::new(),
            current_tool_id: -1,
            current_tool_name_sent: false,
            streamed_args_for_tool: Vec::new(),
        }
    }
}

impl Default for {ParserName}Parser {
    fn default() -> Self {
        Self::new()
    }
}

#[async_trait]
impl ToolParser for {ParserName}Parser {
    async fn parse_complete(&self, output: &str) -> ParserResult<(String, Vec<ToolCall>)> {
        // Find content between START_TOKEN and END_TOKEN
        let Some(start) = output.find(START_TOKEN) else {
            return Ok((output.to_string(), vec![]));
        };
        let normal_text = output[..start].to_string();
        let after_start = &output[start + START_TOKEN.len()..];
        let json_str = if let Some(end) = after_start.find(END_TOKEN) {
            &after_start[..end]
        } else {
            after_start
        };

        // Parse JSON and extract tool calls
        let json_str = json_str.trim();
        let value: Value = serde_json::from_str(json_str)?;

        // Adapt based on your model's JSON structure:
        // Most use {"name": "func", "arguments": {...}}
        // Some use {"tool_name": "func", "parameters": {...}}
        let calls = helpers::extract_tool_calls_from_value(&value)?;
        Ok((normal_text, calls))
    }

    async fn parse_incremental(
        &mut self,
        chunk: &str,
        tools: &[Tool],
    ) -> ParserResult<StreamingParseResult> {
        self.buffer.push_str(chunk);

        // Look for start token
        let Some(start_idx) = self.buffer.find(START_TOKEN) else {
            // No tool call yet — emit as normal text
            let normal = self.buffer.clone();
            self.buffer.clear();
            return Ok(StreamingParseResult {
                normal_text: normal,
                calls: vec![],
            });
        };

        // Emit any text before the tool call as normal text
        let normal_text = self.buffer[..start_idx].to_string();
        let json_start = start_idx + START_TOKEN.len();

        // Check for end token
        let json_text = if let Some(end_idx) = self.buffer[json_start..].find(END_TOKEN) {
            &self.buffer[json_start..json_start + end_idx]
        } else {
            &self.buffer[json_start..]
        };

        // Build tool index map from available tools
        let tool_indices: std::collections::HashMap<String, usize> = tools
            .iter()
            .enumerate()
            .filter_map(|(i, t)| t.function.as_ref().map(|f| (f.name.clone(), i)))
            .collect();

        // Delegate to shared JSON streaming helper
        let mut result = helpers::handle_json_tool_streaming(
            json_text,
            0,
            &mut self.partial_json,
            &tool_indices,
            &mut self.buffer,
            &mut self.current_tool_id,
            &mut self.current_tool_name_sent,
            &mut self.streamed_args_for_tool,
            &mut self.prev_tool_call_arr,
        )?;

        result.normal_text = normal_text;
        Ok(result)
    }

    fn has_tool_markers(&self, text: &str) -> bool {
        text.contains(START_TOKEN)
    }

    fn get_unstreamed_tool_args(&self) -> Option<Vec<ToolCallItem>> {
        helpers::get_unstreamed_args(&self.prev_tool_call_arr, &self.streamed_args_for_tool)
    }

    fn reset(&mut self) {
        helpers::reset_parser_state(
            &mut self.buffer,
            &mut self.prev_tool_call_arr,
            &mut self.current_tool_id,
            &mut self.current_tool_name_sent,
            &mut self.streamed_args_for_tool,
        );
    }
}
```

**Verify:** `cargo check -p tool-parser`

## Step 2: Register in module exports

**File:** `crates/tool_parser/src/parsers/mod.rs` — add:
```rust
pub mod {PARSER_NAME};
pub use {PARSER_NAME}::{ParserName}Parser;
```

**File:** `crates/tool_parser/src/lib.rs` — add to the `pub use parsers::{ ... }` block:
```rust
{ParserName}Parser,
```

**Verify:** `cargo check -p tool-parser`

## Step 3: Register in factory

**File:** `crates/tool_parser/src/factory.rs`

In `ParserFactory::new()`:
```rust
registry.register_parser("{PARSER_NAME}", || Box::new({ParserName}Parser::new()));
```

In `ParserFactory::register_default_mappings()`:
```rust
registry.map_model("{model-pattern-1}*", "{PARSER_NAME}");
registry.map_model("{model-pattern-2}*", "{PARSER_NAME}");
```

Pattern matching uses **glob wildcards** (`*` matches any characters).

**Verify:** `cargo check -p tool-parser`

## Step 4: Write tests

**File:** `crates/tool_parser/tests/tool_parser_{PARSER_NAME}.rs`

```rust
mod common;

use common::create_test_tools;
use tool_parser::{ParserName}Parser, ToolParser};

#[tokio::test]
async fn test_parse_single_tool_call() {
    let parser = {ParserName}Parser::new();
    let input = r#"Some text{START_TOKEN}{"name":"get_weather","arguments":{"city":"Paris"}}{END_TOKEN}"#;
    let (normal, calls) = parser.parse_complete(input).await.unwrap();
    assert_eq!(normal, "Some text");
    assert_eq!(calls.len(), 1);
    assert_eq!(calls[0].function.name, "get_weather");
}

#[tokio::test]
async fn test_parse_multiple_tool_calls() {
    let parser = {ParserName}Parser::new();
    // Test with multiple sequential tool calls
    let input = r#"{START_TOKEN}{"name":"func1","arguments":{}}{END_TOKEN}{START_TOKEN}{"name":"func2","arguments":{}}{END_TOKEN}"#;
    let (_, calls) = parser.parse_complete(input).await.unwrap();
    assert_eq!(calls.len(), 2);
}

#[tokio::test]
async fn test_no_tool_calls() {
    let parser = {ParserName}Parser::new();
    let input = "Just normal text, no tool calls here.";
    let (normal, calls) = parser.parse_complete(input).await.unwrap();
    assert_eq!(normal, input);
    assert!(calls.is_empty());
}

#[tokio::test]
async fn test_has_tool_markers() {
    let parser = {ParserName}Parser::new();
    assert!(parser.has_tool_markers("text {START_TOKEN} more"));
    assert!(!parser.has_tool_markers("plain text"));
}

#[tokio::test]
async fn test_streaming_chunks() {
    let mut parser = {ParserName}Parser::new();
    let tools = create_test_tools();

    // Split a tool call across multiple chunks
    let chunks = vec![
        "Normal text",
        "{START_TOKEN}{\"na",
        "me\":\"get_weather\",\"ar",
        "guments\":{\"city\":\"",
        "Paris\"}}{END_TOKEN}",
    ];

    let mut all_normal = String::new();
    let mut all_calls = Vec::new();
    for chunk in chunks {
        let result = parser.parse_incremental(chunk, &tools).await.unwrap();
        all_normal.push_str(&result.normal_text);
        all_calls.extend(result.calls);
    }
    assert_eq!(all_normal, "Normal text");
    assert!(!all_calls.is_empty());
}

#[tokio::test]
async fn test_empty_arguments() {
    let parser = {ParserName}Parser::new();
    let input = r#"{START_TOKEN}{"name":"no_args","arguments":{}}{END_TOKEN}"#;
    let (_, calls) = parser.parse_complete(input).await.unwrap();
    assert_eq!(calls[0].function.arguments, "{}");
}

#[tokio::test]
async fn test_reset() {
    let mut parser = {ParserName}Parser::new();
    let tools = create_test_tools();
    parser.parse_incremental("{START_TOKEN}{\"name", &tools).await.unwrap();
    parser.reset();
    // After reset, parser should handle new input cleanly
    let result = parser.parse_incremental("fresh text", &tools).await.unwrap();
    assert_eq!(result.normal_text, "fresh text");
}
```

**Verify:** `cargo test --test tool_parser_{PARSER_NAME}`

## Step 5: Run full quality gate

Invoke `smg:contribute` to run fmt → clippy → test → bindings → commit format.

## Adapting for Non-Standard JSON Fields

Some models use non-standard field names. Map them in `parse_complete`:

| Model | Name Field | Arguments Field |
|-------|-----------|-----------------|
| Most models | `name` | `arguments` |
| Cohere | `tool_name` | `parameters` |
| Llama 3.2 | `name` | `parameters` |

Use `helpers::normalize_tool_call_fields()` if available, or map manually:
```rust
let name = obj.get("name")
    .or_else(|| obj.get("tool_name"))
    .and_then(|v| v.as_str());
let args = obj.get("arguments")
    .or_else(|| obj.get("parameters"));
```

## When NOT to Use the JSON-with-Tags Template

| Scenario | Example | What To Do Instead |
|----------|---------|-------------------|
| Model uses XML with key-value parameters | QwenCoder `<parameter=key>value</parameter>` | Write custom XML extraction (see `qwen_coder.rs`) |
| Model uses Python function syntax | Llama 4 `[func(arg=val)]` | Use `PythonicParser` or register as `pythonic` |
| Model uses raw JSON (no tags) | OpenAI, Claude | Register as `json` — no new parser needed |
| Model uses Unicode delimiter tokens | DeepSeek V3 full-width chars | Write custom parser with Unicode-aware matching (see `deepseek.rs`) |

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Not handling multiple tool calls | Only first call extracted | Loop over all start/end token pairs or parse JSON array |
| Resetting between chunks instead of requests | Loses buffer state mid-stream | Only call `reset()` between separate API requests |
| Not validating tool names against `tools` list | Invalid tool calls forwarded to client | Skip calls where name doesn't match any provided tool |
| Re-sending full arguments each chunk | Client receives duplicate argument data | Track `streamed_args_for_tool` and send only the delta |
| Missing `get_unstreamed_tool_args()` | Final arguments lost on fast completions | Implement using `helpers::get_unstreamed_args()` |
| Forgetting `Send + Sync` bounds | Parser can't be pooled with `Arc<Mutex<>>` | Avoid `Rc`, `RefCell`, or non-`Send` types in struct fields |
