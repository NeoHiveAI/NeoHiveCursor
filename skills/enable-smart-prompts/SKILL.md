---
name: enable-smart-prompts
description: Use when the user says "make NeoHive smarter", "rewrite my prompts before searching memory", "enable smart recall", or during `getting-started` Phase 5. Generates a tailored prompt-rewriting helper that uses a small model (Haiku by default) to rewrite the user's prompt into a good `memory_recall` query, decides when lookup is worthwhile, calls NeoHive, and surfaces only the most relevant results. Replaces the default prompt-passing path with a smarter one.
---

# Enable Smart-Recall Prompts (Cursor)

You help the user install a customized prompt-rewriting helper that intercepts their prompt, uses a small model to formulate a good NeoHive query, calls `memory_recall`, and surfaces the most relevant results back into Cursor's context.

This is a **dynamic setup** — every user has a different hive layout, shell, API key location, and tolerance for latency. You walk them through each choice with a strong recommended default, then write the script.

> Cursor hook registration note: Cursor's public plugin docs describe hooks as "automation scripts triggered by events" but do not fully specify the registration syntax for user-prompt-submit events. This skill writes the script to a conventional location and points the user at the current Cursor docs for wiring. If you know your version's syntax, paste it in and the skill will help finalize.

## Phase 0 — Check prerequisites

Before asking anything, verify:

```bash
command -v claude >/dev/null && echo "claude-cli: OK" || echo "claude-cli: MISSING (install from claude.com/code)"
command -v curl   >/dev/null && echo "curl: OK"       || echo "curl: MISSING"
command -v python3>/dev/null && echo "python3: OK"    || echo "python3: MISSING"
[ -n "${ANTHROPIC_API_KEY:-}" ] && echo "ANTHROPIC_API_KEY: set" || echo "ANTHROPIC_API_KEY: not set (required for headless claude)"
```

If `claude-cli` or `python3` is missing, stop and tell the user to install them. If the API key is missing, tell them to export it first and point at https://console.anthropic.com/.

The template uses `claude -p` for headless model invocation because it's the most ergonomic small-model CLI right now. If you prefer to drive the rewriter through a different CLI (OpenAI CLI, llm, etc.), edit the script after generation — the rewrite-and-filter calls are the only two model invocations.

## Phase 1 — Gather configuration

Ask these in sequence, one at a time:

### 1. Which hive to target

Call `list_hives`. Ask which hive the helper should search on every prompt. Default: all hives (cross-hive RRF), which calls `memory_recall` without a `hive` param.

### 2. Which model drives the query rewriter

Options:

- `claude-haiku-4-5` (Recommended) — fast + cheap
- `claude-sonnet-4-6` — more accurate, slower, ~10x cost
- `claude-opus-4-7` — overkill, only for very noisy hives

### 3. Trigger policy

Options:

- (Recommended) Every prompt longer than 10 chars — skips short clarifications
- Only when prompt contains a keyword the user picks
- Every prompt — no filtering
- Manual only — user triggers via an env flag

If "keyword": ask for the keyword(s).

### 4. Install location

Options:

- (Recommended) `~/.cursor/hooks/neohive-smart-recall.sh` — personal, all projects
- `./.cursor/hooks/neohive-smart-recall.sh` — this project only
- Just show me the script — I'll place it myself

### 5. Disable-flag name

Options:

- (Recommended) `NEOHIVE_SMART_DISABLED`
- `NEOHIVE_HOOK_DISABLED`
- Custom — user types

## Phase 2 — Preview the generated script

Build the script from the template at `${CURSOR_PLUGIN_ROOT}/skills/enable-smart-prompts/template.sh`, substituting the chosen values. Show the final script to the user in a fenced code block. Summarize:

```
Generated helper with:
  • Hive:          <hive-or-all>
  • Model:         <model>
  • Trigger:       <policy>
  • Install path:  <path>
  • Disable flag:  <env-var>
```

Ask: "Install this helper now?"
Options:

- (Recommended) Yes, write it and show me the Cursor registration syntax
- Yes, write it — I'll register it myself
- No — I want to tweak the script first

If "tweak": ask what to change, regenerate, re-preview.
If "no": stop with "Nothing written."

## Phase 3 — Write the script

Create parent directories if needed. Write the script to the chosen location with mode `0755`. Show:

```
Wrote <path> (N bytes, mode 755).
```

## Phase 4 — Register the hook

Cursor's exact hook registration syntax is not fully part of this skill's knowledge. Print the following guidance verbatim:

> Check the current Cursor docs at https://cursor.com/docs/plugins for the hook registration syntax. At time of writing, hooks are documented as "automation scripts triggered by events" but the exact event name and config file for user-prompt-submit aren't always spelled out. Common patterns: a `hooks.json` inside `.cursor-plugin/`, or a project-level `.cursor/hooks.json`. Add an entry that points at `<install-path>` for the "user prompt submitted" event.
>
> If you already know the right syntax, paste it and I'll write the registration file for you.

## Phase 5 — Verification

Tell the user:

> Restart Cursor (or reload plugins per your Cursor docs) for the hook to take effect.
>
> Test it: start a new session and ask about something you know is in your hive. You should see a block starting with "NeoHive smart context:" before the model's reply.
>
> Disable temporarily: `export <DISABLE_FLAG>=1` in your shell.
> Disable permanently: remove the hook registration, or delete the script at `<install-path>`.

## Important rules

- **Never overwrite an existing helper at the target path without confirmation.** If the file exists, show its contents and ask whether to replace.
- **Never put the API key in the generated script.** The script reads `$ANTHROPIC_API_KEY` at runtime.
- **Never hardcode the hive UUID in the script.** It discovers the MCP URL the same way the default plugin path does (via `mcp.json` / `.mcp.json` / `~/.cursor/mcp.json`).
- **Always set a `--max-time` on every `curl` and `claude -p` call.** A slow helper blocks every prompt.
- **Gracefully exit 0 on any failure.** A broken helper must never block the user's prompt from reaching Cursor.
