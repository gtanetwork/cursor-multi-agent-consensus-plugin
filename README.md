# Multi-agent Consensus

A Cursor plugin that turns **plan-mode and debug-mode** into a **real collaboration** between multiple frontier models — not a competition.

The lead model (whatever you're chatting with — Opus, GPT, Gemini, etc.) drafts a plan or a debugging hypothesis, hands it to two or more read-only "planning buddy" subagents pinned to *different* models, collects their critiques, and synthesizes a single refined output that incorporates every perspective. You review the consensus output and approve it before any code is written or any fix is applied.

Useful for both:

- **Planning** non-trivial features — catches risky assumptions, missing edge cases, and weak test strategy *before* a single line is written.
- **Debugging** non-obvious bugs — challenges the lead model's first-guess root cause, surfaces alternative hypotheses, and stress-tests the proposed fix's blast radius *before* you ship a fix that papers over the wrong problem.

> Idea inspired by the [PAL / Zen MCP `consensus` tool](https://github.com/BeehiveInnovations/pal-mcp-server/blob/main/docs/tools/consensus.md). The architecture and naming are different (this is a Cursor-native plugin, not an MCP server), but the philosophy of multi-model perspective-gathering is theirs.

## What it does

When you trigger `/consensus` (or are reminded by the bundled `consensus-reminder` rule), the workflow is the same in both modes:

1. **Lead model drafts the artifact** — your active chat model produces either a technical plan (plan-mode) or a structured root-cause hypothesis with a proposed fix (debug-mode).
2. **Two planning-buddy subagents review it in parallel:**
   - `planning-buddy-gemini` (pinned to a Gemini model)
   - `planning-buddy-gpt` (pinned to a GPT model)
   Both are read-only, both receive the full artifact and the original user prompt, and both critique it independently using the same evaluation framework.
3. **Lead model synthesizes consensus** — resolves disagreements, challenges weak suggestions, and produces a single refined output with a Risks & Mitigations section and a list of assumptions that were validated or revised. For debugging, the refined output also lists alternative hypotheses to rule out and the verification step that will confirm or falsify the proposed root cause.
4. **You approve.** Implementation (or the fix) does not happen until you say so.

Critically, this is not "pick the best of N independent attempts." All three models work on the **same artifact** and feed back into each other through the lead model. The lead model retains decision authority.

## Alternative approaches considered

### Native Cursor parallel agents

Cursor 3.0 ships [`/best-of-n` and parallel-agent worktrees](https://cursor.com/changelog/05-07-26): the same task is run in parallel across multiple models, each in an isolated git worktree, and you pick the winning attempt.

This is selection, not collaboration. The agents never see each other's work, never debate trade-offs, and never refine a shared artifact. Useful for "which implementation do I like more," not for "what does the room think of this plan."

### Zen / PAL MCP `consensus` tool

PAL's [`consensus` tool](https://github.com/BeehiveInnovations/pal-mcp-server/blob/main/docs/tools/consensus.md) is the gold standard for multi-model perspective-gathering: stance steering (for/against/neutral), custom stance prompts, focus areas, ethical guardrails, and synthesis.

The blocker for Cursor users on a team plan: PAL talks to model providers directly and **requires you to bring your own API keys** for each model. You cannot reuse the models your Cursor team plan already pays for.

### This plugin

A Cursor-native take on the same idea, built entirely out of plugin-native primitives — subagents, a slash command, a skill, and a reminder rule — so it uses **the models Cursor already gives you** with no extra API keys.

## Architecture

```text
                         User
                          │
                          ▼
              ┌───────────────────────┐
              │   Lead model (chat)   │
              │  e.g. Claude Opus 4   │
              └─────────┬─────────────┘
                        │  /consensus
                        │  (drafts plan)
                        ▼
        ┌───────────────┴────────────────┐
        ▼                                ▼
┌──────────────────┐            ┌──────────────────┐
│ planning-buddy-  │            │ planning-buddy-  │
│ gemini           │            │ gpt              │
│ (read-only,      │            │ (read-only,      │
│ Gemini 3.1 Pro)  │            │ GPT-5.5)         │
└────────┬─────────┘            └────────┬─────────┘
         │  critiques + risks            │
         └───────────────┬───────────────┘
                         ▼
              ┌───────────────────────┐
              │   Lead model again    │
              │ synthesizes consensus │
              └─────────┬─────────────┘
                        ▼
              Refined plan for user approval
```

The two subagents are deliberately **identical except for the model they pin to**. The diversity of perspective comes from the model swap, not from differently-prompted personas.

## Components

| File | Role |
|---|---|
| `commands/consensus.md` | The `/consensus` slash command — orchestrates the workflow. |
| `skills/consensus/SKILL.md` | Same workflow as the slash command, exposed as a skill so it shows up in the Plugins UI and can be triggered explicitly. Auto-invocation by the lead model is intentionally disabled (`disable-model-invocation: true`) to keep consensus runs deliberate. |
| `agents/planning-buddy-gemini.md` | Read-only reviewer pinned to Gemini. |
| `agents/planning-buddy-gpt.md` | Read-only reviewer pinned to GPT. |
| `rules/consensus-reminder.mdc` | Agent-requestable rule that nudges the lead agent to use `/consensus` for non-trivial plans *and* non-obvious debugging hypotheses. |

## Workflow at the keyboard

**Plan-mode:**

1. Open Cursor in plan mode (or just describe a non-trivial change).
2. Type `/consensus` — or let the bundled rule remind the agent to use it.
3. Wait for both planning buddies to return (they run in parallel).
4. Read the consensus output: refined plan, risks, validated/revised assumptions.
5. Approve the plan.
6. Switch to implementation — the agent now codes against the agreed plan.

**Debug-mode:**

1. Open Cursor in debug mode (or describe the bug + symptoms + repro).
2. Let the agent form an initial root-cause hypothesis and proposed fix.
3. Type `/consensus` — the lead model packages the hypothesis and hands it to the planning buddies.
4. Read the consensus output: refined hypothesis, alternative root causes worth ruling out, the verification step, and the fix's blast radius.
5. Approve the refined hypothesis (or send the agent back to investigate the alternatives first).
6. Apply the fix — the agent implements against the agreed direction.

## Requirements

- **Cursor 3.0 or newer.** The plugin system, subagents, and `agents/`/`commands/`/`skills/`/`rules/` auto-discovery all require Cursor 3.0+.
- **Cursor model access for both planning-buddy models.** Out of the box, `planning-buddy-gemini` pins to `gemini-3.1-pro` and `planning-buddy-gpt` pins to `gpt-5.5`. Your Cursor plan must expose those slugs. If either model is unavailable, edit the `model:` field in the corresponding `agents/planning-buddy-*.md` file (see [Customization](#customization)).
- **No external API keys.** Subagents run on Cursor's model access — the plugin does not call provider APIs directly.

## Installation

### From the Cursor marketplace (recommended, once published)

This plugin will be listed on the [Cursor marketplace](https://cursor.com/marketplace). Install it from inside Cursor:

`Settings → Plugins → Browse marketplace → search "Multi-agent consensus" → Install`.

### Local install (development / pre-marketplace)

Cursor's local-plugin loader does **not follow symlinks** — install via real directory copy.

**macOS / Linux**, from the repository root:

```bash
mkdir -p ~/.cursor/plugins/local
rm -rf ~/.cursor/plugins/local/multi-agent-consensus
rsync -a --delete --exclude='.git' "$PWD/" ~/.cursor/plugins/local/multi-agent-consensus/
```

**Windows PowerShell**, from the repository root:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.cursor\plugins\local" | Out-Null
Remove-Item -Recurse -Force "$env:USERPROFILE\.cursor\plugins\local\multi-agent-consensus" -ErrorAction SilentlyContinue
Copy-Item -Recurse -Force "$PWD" "$env:USERPROFILE\.cursor\plugins\local\multi-agent-consensus"
Remove-Item -Recurse -Force "$env:USERPROFILE\.cursor\plugins\local\multi-agent-consensus\.git" -ErrorAction SilentlyContinue
```

Then **fully reload Cursor**: `Cmd+Shift+P → "Developer: Reload Window"`.

### Verify the install

After reload, in Cursor's plugins UI you should see **Multi-agent consensus** [Local] with:

- Skills: `consensus`
- Subagents: `planning-buddy-gemini`, `planning-buddy-gpt`
- Rules: `consensus-reminder`
- Commands: `consensus`

If the plugin doesn't appear, check the Cursor plugin loader log for a line reading `loadUserLocalPlugins completed in Xms (0 plugins loaded)`. If you see it, you most likely installed via symlink — replace with a real directory copy.

Log location:

- **macOS**: `~/Library/Application Support/Cursor/logs/<latest>/window1/exthost/anysphere.cursor-agent-exec/Cursor Plugins.log`
- **Linux**: `~/.config/Cursor/logs/<latest>/window1/exthost/anysphere.cursor-agent-exec/Cursor Plugins.log`
- **Windows**: `%APPDATA%\Cursor\logs\<latest>\window1\exthost\anysphere.cursor-agent-exec\Cursor Plugins.log`

## Customization

### Swap the planning-buddy models

Each `agents/planning-buddy-*.md` pins a model via YAML frontmatter:

```yaml
---
name: planning-buddy-gemini
model: gemini-3.1-pro
description: ...
readonly: true
---
```

Edit the `model:` field to any model your Cursor build exposes. To add a third (or fourth) reviewer, drop a new `agents/planning-buddy-<name>.md` file in — the orchestrator discovers all `planning-buddy-*` agents automatically and launches every one of them in parallel.

### Customize the workflow

Edit `commands/consensus.md` and `skills/consensus/SKILL.md`. Keep the two in sync — they describe the same workflow in two different forms (slash command vs. skill).

## Notes & limitations

- **Cursor-native, not VS Code.** This is a Cursor plugin (manifest at `.cursor-plugin/plugin.json`). It does not load in VS Code or its forks.
- **No external API keys required.** The plugin only uses subagents, which run on your existing Cursor model access.
- **Read-only by design.** Both planning buddies have `readonly: true`. They critique; they do not write code or files.
- **Latency cost.** Two extra model calls run in parallel before you see the refined plan. Real-world overhead is roughly half a minute to a minute and a half, depending on plan size and which models the buddies pin to.

## Credits

- Multi-model consensus pattern: [BeehiveInnovations / pal-mcp-server `consensus` tool](https://github.com/BeehiveInnovations/pal-mcp-server/blob/main/docs/tools/consensus.md).
- Plugin format and validator: [`cursor/plugin-template`](https://github.com/cursor/plugin-template).

## License

See [`LICENSE`](LICENSE).
