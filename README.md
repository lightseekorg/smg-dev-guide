# SMG Claude Plugin

Claude Code plugin for the [Shepherd Model Gateway](https://github.com/lightseekorg/smg) — 4 process-enforcing skills that change what Claude **does**, not just what it **knows**.

## Install

```bash
claude plugins add github:lightseekorg/smg-claude-plugin
```

## Usage

Single entry point — `/smg` routes based on what you're doing:

```
/smg where does the label pipeline live     # → smg:map (orientation)
/smg add a --timeout flag                   # → smg:implement (step-by-step recipe)
/smg review PR #562                         # → smg:review-pr (systematic checklist)
/smg am I ready to submit                   # → smg:contribute (quality gates)
```

## Skills

| Skill | Action | What It Does |
|-------|--------|-------------|
| `smg:map` | Orient | Crate map, layering rules, config propagation, request flow, label pipeline |
| `smg:implement` | Build | Detects subsystem, loads recipe, creates tasks, enforces step-by-step execution with verification |
| `smg:review-pr` | Review | Maps changed files to checklist sections, creates review tasks per subsystem, cites file:line |
| `smg:contribute` | Ship | 5-step quality gate (fmt → clippy → test → bindings → commit format) with enforcement |

## How It Works

Unlike passive reference docs, these skills **enforce workflows**:

- **Iron Laws** prevent skipping critical steps (e.g. "NO PR WITHOUT PASSING ALL QUALITY GATES")
- **Hard Gates** block progression without prerequisites (e.g. must identify touched subsystems before reviewing)
- **Rationalization Tables** counter common excuses for cutting corners
- **Skill Chaining** ensures `implement` → `contribute` → `review-pr` flow
- **15 Implementation Recipes** provide step-by-step guidance with exact file paths, code patterns, and verification commands for every subsystem

## Implementation Recipes

`smg:implement` auto-detects what you're building and loads the right recipe:

| Recipe | Subsystem |
|--------|-----------|
| config-plumbing | CLI flags, config fields, the critical two-path rule |
| routing-policy | Load balancing, dual HTTP/gRPC mode |
| tool-parser | Tool/function call formats (13+) |
| reasoning-parser | Reasoning extraction (10+ model families) |
| bindings-update | Python PyO3 + Go FFI |
| discovery-feature | K8s discovery, label pipeline |
| grpc-backend | gRPC client, trace injection |
| storage-backend | Data connectors, hooks |
| wasm-plugin | WASM middleware, WIT interface |
| auth-feature | API keys, JWT/OIDC |
| observability-feature | Metrics, tracing, logging |
| mesh-feature | SWIM gossip, CRDT stores |
| mcp-feature | MCP protocol, tool execution |
| kv-index-feature | Radix trees, cache-aware routing |
| multimodal-feature | Vision processors, media pipeline |

## License

Apache-2.0
