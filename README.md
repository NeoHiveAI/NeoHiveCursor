# NeoHiveCursor

A Cursor plugin for the [NeoHive](https://github.com/NeoHiveAi) cognitive memory system. Install this plugin to wire Cursor into any NeoHive MCP server — persistent semantic memory across sessions, a built-in rule that enforces memory tool usage, and skills for session setup, memory migration, documentation design, and post-session learning extraction.

## Quick Start

### 1. Install the plugin

From the Cursor app, install from the marketplace or point at this repository. See the [official Cursor plugin docs](https://cursor.com/docs/plugins) for the current install flow.

For local development without publishing, symlink or copy this directory into `~/.cursor/plugins/local/`:

```bash
ln -s "$(pwd)" ~/.cursor/plugins/local/neohive
```

### 2. Run the guided setup

Invoke the `getting-started` skill. It walks through verifying the MCP server, setting up auth, migrating any existing project memory (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `.codex/rules`, `.claude/rules`) into NeoHive, and optionally generating a smart-recall hook. ~3–5 minutes.

### 3. (Optional) Environment overrides

If your NeoHive server requires auth, export a bearer token before launching Cursor:

```bash
export NEOHIVE_TOKEN="your-token-here"
```

## Available Components

| Type | Name | Purpose |
|------|------|---------|
| Rule | `neohive.mdc` | Always-apply rule instructing the model to call `memory_context` before any task |
| Skill | `getting-started` | First-run setup orchestrator |
| Skill | `start` | Pre-load relevant NeoHive memories via `memory_context` |
| Skill | `migrate-memory` | Scan local memory files and migrate project-scoped entries into NeoHive |
| Skill | `generate-docs` | Design a documentation gold standard, save to NeoHive, validate with sample pages |
| Skill | `generate-post-submit-hook` | Generate a tailored smart-recall hook that rewrites prompts with a small model before querying NeoHive |
| Skill | `revise-vector-memory` | End-of-session extraction of learnings into NeoHive |
| MCP | `securisource-neohive` | HTTP MCP server for the Securisource NeoHive |
| MCP | `snyk-neohive` | HTTP MCP server for the Snyk NeoHive |

## Repository Layout

```
NeoHiveCursor/
├── .cursor-plugin/
│   └── plugin.json             # Plugin manifest
├── mcp.json                    # HTTP MCP server registration (no leading dot per Cursor spec)
├── rules/
│   └── neohive.mdc             # Always-apply memory usage rule
├── skills/
│   ├── getting-started/SKILL.md
│   ├── start/SKILL.md
│   ├── migrate-memory/SKILL.md
│   ├── generate-docs/SKILL.md
│   ├── generate-post-submit-hook/
│   │   ├── SKILL.md
│   │   └── template.sh
│   └── revise-vector-memory/SKILL.md
└── README.md
```

## Publishing

Cursor requires open-source plugins and a manual review step. Load the plugin locally from `~/.cursor/plugins/local/neohive` to test, then submit at [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish).
