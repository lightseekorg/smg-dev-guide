# Adding Multimodal Features to SMG

Image/audio processing for vision-capable LLMs. Fetches media, preprocesses, tracks through pipeline.

## Pipeline

```
User message with image URL/data
  → Media connector fetches (URL, data URL, local file, inline bytes)
  → ImageFrame created (DynamicImage + raw bytes + metadata)
  → Vision processor (model-specific: LLaVA, LLaVA-Next)
  → MultiModalInputs (token IDs, mm_kwargs, placeholders, cache salt)
  → Included in prompt for backend
```

## Steps

### Adding a New Modality

1. Extend `Modality` enum and `ChatContentPart` in `crates/multimodal/src/`
2. Add fetch method to media connector
3. Implement processing pipeline
4. Track with UUID for deduplication

### Adding a Vision Processor

**Directory:** `crates/multimodal/src/vision/`

1. Implement processor trait (image → model-specific tensor format)
2. Handle resizing, normalization, placeholder insertion
3. Add per-model spec module in `crates/multimodal/src/registry/` (e.g. `mymodel.rs`)
4. Register in the registry's `mod.rs`
5. Add NPZ array comparison tests for output validation

### Adding a Media Source

Add fetch method to media connector:
- Domain whitelisting for URLs
- Timeout handling (default 10s)
- Content hashing for deduplication

## Content Types

```rust
pub enum ChatContentPart {
    Text { text: String },
    ImageUrl { url, detail: Option<ImageDetail>, uuid },
    ImageData { data: Vec<u8>, mime_type, uuid, detail },
    ImageEmbeds { payload: Value, uuid },
}
```

## Key Rules

- Track all media with UUIDs for deduplication and caching
- Domain whitelisting on URL fetches
- Timeout handling on all external fetches
- Use fixture images for tests, NPZ comparison for processor validation
