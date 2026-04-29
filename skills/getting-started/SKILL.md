---
name: getting-started
description: First-run setup for NeoHive. Walks a new user through verifying the MCP server, setting up auth, generating a project-specific topology block in AGENTS.md, migrating existing project memory, and enabling optional helpers. Invoke this once per machine after installing the neohive plugin.
---

# Getting Started with NeoHive

You are onboarding a user who has just installed the NeoHive plugin for Cursor. Your job is to get them from zero to a fully working setup — MCP reachable, memory migrated, helpers configured — without ever leaving them staring at a blank screen.

**Golden rule for this skill: never act silently.** Narrate every step. Every decision has a recommended default. Every write is gated by an explicit confirmation.

## Phase 0 — Tell the user what's about to happen

Open with this exact script (do not paraphrase):

> I'll walk you through setting up NeoHive on this machine. This takes 3–5 minutes and covers:
>   1. Confirming your NeoHive server is reachable
>   2. (Optional) Setting up your auth token
>   3. Generating a project-specific topology block in your AGENTS.md
>   4. Migrating existing project knowledge into NeoHive
>   5. (Optional) Turning on the smart-recall hook
>
> You can stop at any point by saying "stop" or answering "skip" to a step.

Wait for acknowledgement (any affirmative reply, or just continue if they say nothing).

## Phase 1 — Verify MCP reachability

Call `list_hives` immediately. Interpret the outcome:

| Outcome | What to tell the user |
|---|---|
| Returns hives | "Connected. I can see N hives: X, Y, Z." Proceed to Phase 2. |
| Empty list | "Server is reachable but reports no hives. Confirm with your admin — without at least one hive, NeoHive has nowhere to store memories." Pause for user input. |
| Tool unavailable / error | "I can't reach the NeoHive MCP server." Run the diagnostics below. |

### Diagnostics if unreachable

Run these checks and report results in a compact block:

```bash
# 1. Is mcp.json present in the plugin install?
ls ~/.cursor/plugins/local/neohive/mcp.json ~/.cursor/plugins/*/neohive/mcp.json 2>&1 | head -1 || echo "missing"
# 2. Is NEOHIVE_TOKEN set?
[ -n "${NEOHIVE_TOKEN:-}" ] && echo "token set" || echo "token not set"
# 3. Can we reach the server?
grep -oE 'https?://[^"]+' ~/.cursor/plugins/local/neohive/mcp.json ~/.cursor/plugins/*/neohive/mcp.json 2>/dev/null | head -1 | awk -F: '{print $2":"$3}' | xargs -I{} curl -sS -o /dev/null -w "HTTP %{http_code}\n" --max-time 5 "{}" 2>&1 || true
```

Then offer the user: "Fix token now", "I'll fix it later and restart Cursor", "Skip MCP setup for now". If they skip, jump to Phase 6 with a warning that memory features won't work.

## Phase 2 — Auth token (only if needed)

If `list_hives` succeeded, skip this phase. Otherwise ask: "Does your NeoHive server require a bearer token?"

- **Yes — I have one** (recommended): show:
  > Export it before launching Cursor:
  > ```bash
  > export NEOHIVE_TOKEN="your-token-here"
  > ```
  > Add that line to your shell rc (`~/.bashrc`, `~/.zshrc`, or `~/.config/fish/config.fish`) so it persists. Then restart Cursor and rerun this skill.
- **Yes — I need to get one from my admin**: point them at your team's NeoHive admin, then stop.
- **No — it's open**: continue.
- **I'm not sure**: offer to try the call without a token first. If it fails, come back here.

## Phase 3 — Generate project AGENTS.md topology

Now that the MCP is reachable, generate a project-specific topology block in `./AGENTS.md`. This is what makes the model reliable about *which* hive to query and *where* new writes should land — without it, the always-applied rule in `rules/neohive.mdc` is running blind.

Ask the user:

> Generate a project topology block in ./AGENTS.md? (Recommended — improves tool-calling accuracy for everyone on this repo.)

- **Yes** (recommended) — invoke the `generate-agents-md` skill
- **Yes, but let me review the table before writing** — invoke `generate-agents-md` (it has its own review gates)
- **Skip — I'll run generate-agents-md later**

If yes, invoke the `generate-agents-md` skill. The sub-skill handles its own confirmation gates (synthesis review + diff review), so this phase just waits for it to return. When it returns, report: "Topology block written to ./AGENTS.md (N hives mapped)."

If skip, tell the user they can run `generate-agents-md` anytime to add the block, and continue.

## Phase 4 — Migrate existing project memory

Ask the user:

> Want me to scan this project for existing knowledge (AGENTS.md, CLAUDE.md, `.cursor/rules`, `.codex/rules`, `.claude/rules`) and migrate the project-specific parts into NeoHive?

- **Yes** (recommended) — invoke the `migrate-memory` skill
- **Yes, but let me review each memory first** — invoke `migrate-memory` with argument `review=each`
- **Skip — nothing worth migrating**
- **Skip — I'll do this manually later**

Wait for the migrate skill to complete. Report: "Migration done — N memories stored." Then continue.

## Phase 5 — Smart-recall hook (optional, power users)

Ask:

> The default memory lookup passes your prompt verbatim to NeoHive. A smarter version uses a small model to rewrite the query first — usually better results, but costs a few tokens per prompt. Set it up?

- **Not now** (recommended)
- **Yes, set it up** — invoke the `generate-post-submit-hook` skill
- **Tell me more first** — explain in 3–4 sentences (what it adds, what it costs, how to disable), then re-ask

## Phase 6 — Final summary

Print a checklist. Use ✓ / ○ prefixes:

```
✓ MCP server reachable (N hives: ...)
✓ Auth token configured
✓ Project topology block in ./AGENTS.md (N hives mapped)
✓ N project memories migrated
○ Smart-recall hook (skipped — rerun generate-post-submit-hook anytime)
```

Then this exact closing block:

> **You're set. Three things to remember:**
>   1. The `neohive.mdc` rule is always on — the model will call `memory_context` automatically before tasks.
>   2. At the start of a new session, invoke the `start` skill with a short description of what you're working on. That pre-loads extra targeted memory.
>   3. At the end of a session, invoke `revise-vector-memory` so new insights get captured.
>
> When docs feel stale, try `generate-docs`. Rerun `getting-started` anytime to revisit these steps.

## Important rules

- **Never call `memory_store` directly from this skill.** Delegate to `migrate-memory` or `revise-vector-memory`.
- **Never edit the user's shell rc files yourself.** Show the command, let them paste.
- **If the user says "stop" or "skip" at any phase, stop immediately** and print the Phase 6 summary with what's done so far.
- **If any sub-skill fails, surface the error plainly** and offer to skip that phase rather than retrying silently.
