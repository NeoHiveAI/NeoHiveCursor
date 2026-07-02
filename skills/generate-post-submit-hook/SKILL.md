---
name: generate-post-submit-hook
description: Generate a tailored prompt-interception hook that uses a small model (e.g. Haiku) to intelligently query NeoHive on every user prompt. The raw MCP connection returns literal recall hits for whatever text is passed in; this generator produces a smarter wrapper that rewrites queries, decides when lookup is worthwhile, and formats results. Use when the user asks for smarter memory lookup, a prompt-interception script, or during getting-started Phase 4.
---

# Generate a Smart Post-Submit Hook

You are helping the user install a customized hook that intercepts their prompt, uses a small model to formulate a good NeoHive query, calls `memory_recall`, and injects relevant results back into Cursor's context.

This is a **dynamic setup** — every user has a different hive layout, shell, API key location, and tolerance for latency. You walk them through each choice with a strong recommended default, then write the script.

**Note on Cursor hook registration:** Cursor's public docs describe hooks as "automation scripts triggered by events" but do not specify the exact registration syntax. This skill focuses on generating the script itself and placing it at a conventional location; it then points the user at the current Cursor docs for registration. If registration details have been published since, adapt accordingly.

## Phase 0 — Check prerequisites

Before asking anything, verify:

```bash
command -v claude >/dev/null && echo "claude-cli: OK" || echo "claude-cli: MISSING (install from claude.com/code)"
command -v curl   >/dev/null && echo "curl: OK"       || echo "curl: MISSING"
command -v python3>/dev/null && echo "python3: OK"    || echo "python3: MISSING"
[ -n "${ANTHROPIC_API_KEY:-}" ] && echo "ANTHROPIC_API_KEY: set" || echo "ANTHROPIC_API_KEY: not set (required for the headless rewriter)"
```

If `claude-cli` or `python3` is missing, stop and tell the user to install them. If the API key is missing, tell them to export it first and point at <https://console.anthropic.com/>.

You do not need to enforce Claude CLI specifically — the template below uses it because it's the most ergonomic headless interface for Claude models. If the user prefers to drive the rewriter through a different CLI (OpenAI CLI, llm, etc.), offer to adapt.

## Phase 1 — Gather configuration

Ask these in sequence, one question at a time:

### 1. Which hive to target

Call `list_hives`. Ask:

> Which hive should the hook search on every prompt?

- **All hives (cross-hive RRF)** (recommended) — calls `memory_recall` without a `hive` param
- One entry per hive returned by `list_hives`

### 2. Which model drives the rewriter

> Which model should rewrite your prompt into a NeoHive query and filter results?

- `claude-haiku-4-5` (recommended) — fast + cheap
- `claude-sonnet-4-6` — more accurate, slower, ~10× cost
- `claude-opus-4-7` — overkill, only for very noisy hives

### 3. Trigger policy

> When should the hook fire?

- **Every prompt longer than 10 chars** (recommended) — skips short clarifications
- **Only when prompt contains a keyword I pick** — ask follow-up for keyword list
- **Every prompt** — no filtering
- **Manual only** — only fires when `NEOHIVE_SMART_RUN=1` is set

### 4. Install location

> Where should the hook live?

- `~/.cursor/hooks/neohive-smart-recall.sh` (recommended) — personal, all projects
- `./.cursor/hooks/neohive-smart-recall.sh` — this project only
- **Just show me the script** — user will place it manually

### 5. Disable-flag name

> What env var should disable the hook when set to 1?

- `NEOHIVE_SMART_DISABLED` (recommended)
- `NEOHIVE_HOOK_DISABLED` — same flag your default NeoHive path uses; one switch turns both off
- Custom — user types it

## Phase 2 — Preview the generated script

Build the script from `template.sh` in this skill directory — substitute the chosen values. Show the final script in a fenced code block. Summarize at the top:

```
Generated hook with:
  • Hive:          <hive-or-all>
  • Model:         <model>
  • Trigger:       <policy>
  • Install path:  <path>
  • Disable flag:  <env-var>
```

Ask one last question:

> Install this hook now?

- **Yes, write it and register it** (recommended) — write the script, then show the user the Cursor hook registration syntax to run
- **Yes, write it — I'll register it myself**
- **No — I want to tweak the script first**

If "tweak": ask what they want to change, regenerate, re-preview.
If "no" at any point: stop with "Nothing written."

## Phase 3 — Write the script

Create parent directories if needed. Write the script to the chosen location with mode `0755`. Show:

```
Wrote <path> (N bytes, mode 755).
```

## Phase 4 — Register the hook

Cursor's exact hook registration syntax is not part of this skill's knowledge. Print the following guidance verbatim:

> Check the current Cursor docs at <https://cursor.com/docs/plugins> for the hook registration syntax. At time of writing, hooks are documented as "automation scripts triggered by events" but the exact event name and config file aren't spelled out. Common patterns used by similar tools: a `hooks.json` file inside `.cursor-plugin/` or a project-level `.cursor/hooks.json`. Add an entry that points at `<install-path>` for the "user prompt submitted" event.

If the user says they know the right syntax, let them paste it and offer to write the registration file for them.

## Phase 5 — Verification

Tell the user:

> Restart Cursor (or reload plugins per your Cursor docs) for the hook to take effect.
>
> Test it: start a new session and ask about something you know is in your hive. You should see a block starting with "NeoHive smart context:" before the model's reply.
>
> Disable temporarily: `export <DISABLE_FLAG>=1` in your shell.
> Disable permanently: remove the hook registration, or delete the script at `<install-path>`.

## Important rules

- **Never overwrite an existing hook at the target path without confirmation.** If the file exists, show its contents and ask whether to replace.
- **Never put the API key in the generated script.** The script reads `$ANTHROPIC_API_KEY` at runtime.
- **Never hardcode the hive UUID in the script.** It discovers the MCP URL the same way the default plugin connection does (via `mcp.json` / `.mcp.json` / `~/.claude.json`).
- **Always set a `--max-time` on every `curl` and `claude -p` call.** A slow hook blocks every prompt.
- **Gracefully exit 0 on any failure.** A broken hook must never block the user's prompt.
