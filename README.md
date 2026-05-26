# NeoHiveCursor

A Cursor plugin for the [NeoHive](https://github.com/NeoHiveAi) cognitive memory system. Install this plugin to wire Cursor into any NeoHive MCP server — persistent semantic memory across sessions, an always-apply rule that enforces memory tool usage, and skills for session setup, memory migration, documentation design, and post-session learning extraction.

## Quick Start

### 1. Install the plugin

From the Cursor app, install from the marketplace or point at this repository. See the [official Cursor plugin docs](https://cursor.com/docs/plugins) for the current install flow.

For local development without publishing, symlink or copy this directory into `~/.cursor/plugins/local/`:

```bash
ln -s "$(pwd)" ~/.cursor/plugins/local/neohive
```

### 2. Run the guided setup

Invoke the `getting-started` skill. It walks through verifying the MCP server, setting up auth, generating a project-specific topology rule at `.cursor/rules/neohive-topology.mdc`, migrating any existing project memory (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.codex/rules`, `.claude/rules`) into NeoHive, and optionally enabling the smart-recall helper. ~3–5 minutes.

### 3. (Optional) Environment overrides

If your NeoHive server requires auth, export a bearer token before launching Cursor:

```bash
export NEOHIVE_TOKEN="your-token-here"
```

Suppress the in-response hint emitted by `memory_recall` / `memory_context`:

```bash
export NEOHIVE_MCP_HINTS=0
```

## Available Components

| Type | Name | Purpose |
|------|------|---------|
| Rule | `rules/neohive.mdc` | Always-apply rule: when to call `memory_context` / `memory_recall` / `memory_store`, recall-first guidance for codebase exploration, prose-encoded guidance for delegated exploration (the Cursor analogue of Claude's `explore-neohive` subagent + PreToolUse hook). |
| Skill | `getting-started` | First-run setup orchestrator: verifies MCP, sets up auth, generates topology rule, migrates memory, surfaces next steps. |
| Skill | `load-context` | Pre-load relevant NeoHive memories via `memory_context`. Run at the start of every session. |
| Skill | `generate-cursor-rules` | Survey connected hives and write a project-specific topology rule at `.cursor/rules/neohive-topology.mdc`. Re-runnable when hives change. |
| Skill | `migrate-memory` | Scan local memory files (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.codex/rules`, `.claude/rules`) and migrate project-scoped entries into NeoHive. |
| Skill | `design-codebase-docs` | Design a documentation gold standard through a guided dialogue, save to NeoHive, validate with 2–3 sample pages, then hand off to a fresh session. |
| Skill | `enable-smart-prompts` | Generate a tailored prompt-rewriting helper that rewrites prompts with a small model before querying NeoHive. Cursor hook-wiring is platform-dependent — see the skill for current guidance. |
| Skill | `capture-session-learnings` | End-of-session extraction of learnings, corrections, and insights into NeoHive. |

The slugs `start`, `revise-vector-memory`, `generate-docs`, and `generate-post-submit-hook` remain as deprecated aliases that redirect to the new names; they will be removed in a future minor release.

The plugin does **not** ship a pre-configured MCP server — register your own NeoHive gateway through Cursor's MCP configuration. One server keyed with `neohive` is enough; the skills resolve it automatically.

## Repository Layout

```
NeoHiveCursor/
├── .cursor-plugin/
│   └── plugin.json                       # Plugin manifest
├── rules/
│   └── neohive.mdc                       # Always-apply memory-usage rule
├── skills/
│   ├── getting-started/SKILL.md
│   ├── load-context/SKILL.md
│   ├── capture-session-learnings/SKILL.md
│   ├── generate-cursor-rules/SKILL.md
│   ├── migrate-memory/SKILL.md
│   ├── design-codebase-docs/SKILL.md
│   ├── enable-smart-prompts/
│   │   ├── SKILL.md
│   │   └── template.sh
│   ├── start/SKILL.md                    # deprecated alias
│   ├── revise-vector-memory/SKILL.md     # deprecated alias
│   ├── generate-docs/SKILL.md            # deprecated alias
│   └── generate-post-submit-hook/SKILL.md # deprecated alias
└── README.md
```

## Cursor vs Claude — Feature Notes

Cursor's plugin surface is leaner than Claude Code's in two specific ways. Two Claude-only mechanisms are encoded as prose in `rules/neohive.mdc` rather than as runtime hooks:

- **`explore-neohive` subagent (Claude)** → Cursor has no per-agent tool allowlist, so the rules file instructs you to apply the same "recall first, traverse second" pattern manually and to include that directive when delegating exploration to another agent or task runner.
- **`PreToolUse` hook on Glob/Grep (Claude)** → Cursor has no equivalent hook surface, so the rules file prompts you to mentally check "would `memory_recall` answer this?" before running broad filesystem searches in indexed projects.

The `enable-smart-prompts` skill installs a standalone shell helper. Wiring it into the prompt path depends on which Cursor version you use and what hook syntax it exposes for user-prompt-submit events — see the skill for current guidance.

## Publishing

Cursor requires open-source plugins and a manual review step. Load the plugin locally from `~/.cursor/plugins/local/neohive` to test, then submit at [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish).

## Versioning

Bump `version` in `.cursor-plugin/plugin.json` on every change — without a bump, installed users may not see updates. Restart Cursor after edits to a local plugin so it picks up changes.
