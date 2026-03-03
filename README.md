# SMG Dev Guide

AI-powered development guide for the [Shepherd Model Gateway](https://github.com/lightseekorg/smg) — 4 process-enforcing skills that change what your AI coding agent **does**, not just what it **knows**.

Works with **Google Antigravity**, **Gemini CLI**, **Claude Code**, **Codex**, and **Cursor**.

## Install

### Google Antigravity

**Project-level (Recommended)**: Antigravity natively supports the skills through its workflow system. Simply open this repository (or copy the `.agents/` and `skills/` directories into your SMG project root) in the Antigravity IDE and it will automatically discover them.

**Global**: To make these skills and workflows available across all projects, copy or symlink them into your global Antigravity `~/.gemini/antigravity/` directory:

```bash
git clone https://github.com/lightseekorg/smg-dev-guide.git ~/.gemini/antigravity/repos/smg-dev-guide
mkdir -p ~/.gemini/antigravity/workflows ~/.gemini/antigravity/skills
ln -s ~/.gemini/antigravity/repos/smg-dev-guide/.agents/workflows/* ~/.gemini/antigravity/workflows/
ln -s ~/.gemini/antigravity/repos/smg-dev-guide/skills/* ~/.gemini/antigravity/skills/
```

### Gemini CLI

The Gemini CLI natively supports Agent Skills. You can install these skills directly from the repository using the CLI's built-in package manager.

**Global Installation (Available in all projects)**:
```bash
gemini skills install https://github.com/lightseekorg/smg-dev-guide.git
```

**Workspace Installation (Only in current project)**:
```bash
gemini skills install https://github.com/lightseekorg/smg-dev-guide.git --scope workspace
```

### Claude Code

```bash
claude marketplace add lightseekorg/smg-dev-guide
```

### Codex

Copy or symlink the skills into your user skills directory:

```bash
git clone https://github.com/lightseekorg/smg-dev-guide.git ~/.agents/repos/smg-dev-guide
ln -s ~/.agents/repos/smg-dev-guide/.agents/skills/* ~/.agents/skills/
```

Or clone into the SMG repo and skills are discovered automatically from `.agents/skills/`.

### Cursor

Install as a Cursor plugin via `.cursor-plugin/plugin.json`.

## Usage

4 skills, each enforcing a specific developer action:

| Skill | Action | What It Does |
|-------|--------|-------------|
| `map` | Orient | Crate map, layering rules, config propagation, request flow, label pipeline |
| `implement` | Build | Detects subsystem, loads recipe, creates tasks, enforces step-by-step execution with verification |
| `review-pr` | Review | Maps changed files to checklist sections, creates review tasks per subsystem, cites file:line |
| `contribute` | Ship | 5-step quality gate (fmt → clippy → test → bindings → commit format) with enforcement |

**Google Antigravity** — invoke workflows using slash commands in the chat:
```
/map                         → discover codebase structure and ownership
/implement-feature           → guides you through building a feature
/review-pr                   → checks your work against anti-patterns
/verify-pr                   → runs the full 5-step quality gate
```

**Gemini CLI** — skills trigger automatically based on your prompt:
```
"Where does the label pipeline live?"      → Activates map skill
"Am I ready to submit?"                    → Activates contribute skill
```
*Note: You can view installed skills by running `gemini skills list`.*

**Claude Code** — use the `/smg` command:
```
/smg where does the label pipeline live     → smg:map
/smg add a --timeout flag                   → smg:implement
/smg review PR #562                         → smg:review-pr
/smg am I ready to submit                   → smg:contribute
```

**Codex** — skills trigger automatically based on your prompt, or invoke explicitly via `$smg-implement`, `$smg-contribute`, etc.

## How It Works

Unlike passive reference docs, these skills **enforce workflows**:

- **Iron Laws** prevent skipping critical steps (e.g. "NO PR WITHOUT PASSING ALL QUALITY GATES")
- **Hard Gates** block progression without prerequisites (e.g. must identify touched subsystems before reviewing)
- **Rationalization Tables** counter common excuses for cutting corners
- **Skill Chaining** ensures `implement` → `contribute` → `review-pr` flow
- **15 Implementation Recipes** provide step-by-step guidance with exact file paths, code patterns, and verification commands for every subsystem

## Implementation Recipes

`implement` auto-detects what you're building and loads the right recipe:

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

## Directory Structure

```
.agents/workflows/       # Antigravity workflow definitions
.claude-plugin/          # Claude Code marketplace manifest
.cursor-plugin/          # Cursor plugin manifest
.agents/skills/          # Codex skill discovery (symlinks to skills/)
skills/                  # Skill source files
  map/SKILL.md
  contribute/SKILL.md
  review-pr/SKILL.md
  implement/SKILL.md     # + 15 recipe files
commands/                # Claude Code /smg command router
  smg.md
```

## License

Apache-2.0
