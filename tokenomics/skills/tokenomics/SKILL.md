---
name: tokenomics
description: After a plan is drafted, decide whether delegation pays, map each task to the main session's model or a cheaper tier (Sonnet, Haiku) by difficulty and token shape, bundle and delegate to the matching subagent without breaking the main session's prompt cache, then review results in the main session against acceptance criteria. Use at the end of any planning session or when asked to "route the plan" or run /tokenomics.
---

# Tokenomics (LiteLLM build)

Version: 1.4.0 (2026-10-08). Forked from tstack 1.3.0 for API access through the
LiteLLM gateway (`ANTHROPIC_BASE_URL=https://<your-litellm-gateway>`). This build
assumes API billing, not a Claude subscription.

Turn a finished plan into routed, delegated work, then review it. Routing is
relative to whatever model the main session runs (call it **Main**): Main keeps
the hard work, and only tiers *below* Main are worth delegating to.

## Provider config

Model IDs are the LiteLLM aliases. Agent files pin these full IDs. Don't use
bare `sonnet` or `haiku` aliases, because on this gateway they may resolve to
something else.

| Tier | Model ID | Agent | Effort |
|---|---|---|---|
| Opus (Main, usually) | `claude-opus-5-5` | none (it is Main) | session default |
| Sonnet | `claude-sonnet-5-5` | `worker` | medium |
| Haiku medium | `claude-haiku-5-5` | `scout`, `Explore` | medium |
| Haiku low | `claude-haiku-5-5` | `grunt` | low |

Main is chosen per session with `/model`. Don't change it mid-session.
`settings.json` sets the default only.

Tier ladder, strongest first: Opus > Sonnet > Haiku.
- Main on Opus: delegate to Sonnet (`worker`) and Haiku (`scout`, `Explore`, `grunt`).
- Main on Sonnet: Sonnet-tier work stays inline; only Haiku-tier work is delegated.
- Main on Haiku: run everything inline.

Cheaper tiers save money on **output tokens and new input** (file reads, tool
output), not on re-reading context: Main re-reads its warm cache at the cache-read
rate. Every subagent starts cold, so a subagent's cost is its cold prefix, the
brief Main writes, and the report Main reads. The cost model is in
`reference/economics.md`. Its prices are list prices, so treat them as relative
cost, not the gateway's actual bill.

## 0. Delegation gate
Each tier has its own gate. "New input" means files and tool output not already
in Main's context.
- **Sonnet:** delegate only when the task is expected to **write more than
  about 3K tokens or read more than about 6K tokens of new input**. Below both,
  the cold start costs more than it saves.
- **Haiku:** delegate when the brief is clearly shorter than what Main would
  write doing the task inline, or when the task reads new input Main doesn't
  need to keep (searches, bulk reads). Main's brief and review are the real cost.

Execute the whole plan inline in the main session (lower `/effort` if the work
is routine) and skip steps 2–5 when **any** of these is true:
- No task passes its tier's gate.
- One dependent chain where each step needs the last one's reasoning.
- The files involved are already in context.
- The "done when" can't be checked objectively (a cheap tier risks paying twice).
- No tier exists below Main.

State the decision in one line, e.g. *"Inline: 2 edits, ~1K written, files already loaded."*

Inline still means using the right skill. If a dedicated skill covers the work
(`code-review` for a review, `security-review` for a security pass), invoke it
in the main session instead of doing the work freehand.

## 1. Break the plan into tasks
Each task must be independently executable and include:
- **ID**: T1, T2, …
- **Description**: one or two sentences
- **Inputs**: file paths, line ranges, or outputs of other tasks it needs
- **Touches**: files it will modify
- **Done when**: concrete, checkable acceptance criteria (tests pass, file exists, output matches spec)
- **Depends on**: task IDs, if any
- **Shape**: `read-heavy` (> ~6K new input), `write-heavy` (> ~3K output), `reasoning` (small I/O, hard), or `touch-up` (small change to already-loaded context)
- **Size**: rough tokens read / written, e.g. `20K / 2K`

If a task can't be given concrete acceptance criteria, it is still ambiguous and stays with Main.

## 2. Route each task

| Tier | Route here when the task… | Examples |
|---|---|---|
| **Main** (main session) | needs judgment, design choices, ambiguity resolution, planning, cross-cutting reasoning, or security-sensitive decisions; is a `reasoning`/`touch-up` task; or doesn't pass its tier's gate | architecture, planning, ambiguous tasks, hard reviews, tricky debugging, API design, small fixes in loaded files |
| **Sonnet** (`worker`) | is well specified, needs real reasoning or non-trivial code, and is `read-heavy` or `write-heavy` | implementing a feature to spec, multi-file refactors, debugging without a reproducer, larger test suites |
| **Haiku medium** (`scout`) | is routine and well specified, and a test or command proves the result | small code changes with a test, bug fixes with a failing test or reproducer, tests written to a spec |
| **Haiku medium** (`Explore`) | is a read-only search with a clear question | codebase searches, repo exploration, "where is X used" |
| **Haiku low** (`grunt`) | is mechanical, follows a pattern given in the brief, and has an objective "done when" | classification, extraction, summarizing, processing search results, formatting, renames, boilerplate, docstrings, simple transforms |

Routing rules:
- Never route a task to a tier at or above Main; that tier is Main itself, inline.
- When in doubt between two tiers, choose the cheaper one **only if** the "done when" is fully objective; otherwise choose the stronger one.
- Haiku low vs medium: `grunt` when the brief gives the pattern to apply; `scout` when the agent has to work out where or how (find the call sites, make a failing test pass) or needs Bash.
- Debugging goes to `scout` only when the brief includes a failing test or a reproducer. Without one, "done when" isn't objective: route to Sonnet.
- Any task touching auth, crypto, data deletion, or migrations goes to Sonnet at minimum (Main if Main is Sonnet) and is always flagged for review by Main.
- Split a mixed task rather than routing the whole thing up a tier.
- Keep each Haiku package under about **60K tokens of reading** by Main's count. This cap avoids the >100K-token price tier, which applies if the gateway bills Haiku the way Anthropic does. It is a conservative default until the proxy's billing is confirmed.
- Don't give Haiku open-ended reasoning: a failed run costs a redo one tier up.

## 3. Bundle into packages
Group same-tier tasks that share inputs into **one** subagent call (P1, P2, …),
so the cold start is paid once. A package can pass the Sonnet gate even when
its tasks don't individually. Haiku packages stay under the size cap in step 2.
Run packages in parallel only when they are independent (no dependency between
them and no overlap in **Touches**) and latency matters; each parallel spawn is
another cold start.

## 4. Output the routing table
Present it before delegating:

| ID | Task | Shape | Size (read/write) | Tier | Package | Why | Done when | Touches | Depends on |
|---|---|---|---|---|---|---|---|---|---|

Then add a one-line estimate of the share of expected **tokens** (not tasks) on each tier.

Stop and wait for the user to approve or adjust the table when any task is
flagged (auth, crypto, deletion, migrations) or more than half the expected
tokens go to subagents, unless the user already said to proceed without
confirmation. Otherwise continue straight to delegation.

## 5. Delegate (cache-safe)
- Never switch the main session's model mid-conversation. That cold-starts the whole conversation cache. Pick the model at session start with `/model`. Cheaper tiers run only as subagents.
- Main-tier tasks run in the main session; Sonnet packages go to `worker`, Haiku medium packages to `scout`, Haiku low packages to `grunt`, and read-only searches to `Explore`. Respect dependencies.
- The agents are installed in `~/.claude/agents/` with full model IDs. If one is missing, use the general-purpose agent with an explicit model override set to the full ID (`claude-sonnet-5-5` or `claude-haiku-5-5`), set its effort (`medium` for `scout` and `Explore` work, `low` for `grunt` work), and put the agent's report contract (≤ 10 lines, no diffs or file contents) in the brief. Never use the bare `sonnet`/`haiku` aliases on this gateway.
- Briefs carry **pointers, not pastes**: paths, line ranges, each task's description and "done when", constraints. Never paste file contents or the whole plan; Main's output is the most expensive token you can spend. If the work needs a skill (`deslop`, `technical-writing`), name it in the brief: Haiku 5.5 often skips a skill it isn't told to load.
- Expect compact reports (the agents are told ≤ 10 lines). Everything returned is re-read on every later main-session turn.
- **Cache TTL:** on this gateway both Main and subagents get the **5-minute** cache. A subagent or Main idle past 5 minutes pays the cache write again. While a subagent runs, Main should do its own independent work rather than wait idle. Set `promptCacheTtl` or `subagentPromptCacheTtl` only after confirming the proxy honors 1-hour cache writes.

## 6. Review (Main), proportionate
For each completed task:
- Objective "done when" (tests, lint, build, file exists): run the check yourself and accept on pass. Don't re-read the diff.
- Read the diff only for flagged tasks (auth, crypto, deletion, migrations), spot-checks of Haiku output, and integration points between tasks.
- Mark it **pass**, **fix** (small correction done by Main), or **escalate** (redo one tier up, with the original brief plus the failure reason). The order is Haiku low, Haiku medium, Sonnet, Main.
- A task escalates at most once. If it fails again, Main does it inline.
- A Haiku run that ends in a safety refusal is re-run on Sonnet with the same brief. Haiku 5.5 has no server-side fallback, and a refusal says nothing about the task's difficulty, so it doesn't count as the task's escalation. Log it as `refusal` in notes.

## 7. Log
Append to `~/.claude/routing-log.md` (create it with the header line below if
it doesn't exist), one line per task; inline runs get one line for the whole
plan. Use a different path only when the user names one.

`date | project | task ID | shape | main model | tier | package | result (pass/fix/escalate/inline) | tokens in/out | notes`

`tier` is one of `main`, `sonnet`, `haiku-med`, or `haiku-low`. `main model` is the full
model ID of the session (for example `claude-opus-5-5`). `project` is the repository or
working-directory name. Record token usage when the subagent result reports it (one
figure per package is fine); otherwise write `n/a`. The session transcripts under
`~/.claude/projects/` hold per-model usage for later cost analysis.

Escalations are the tuning signal: if a (tier, shape) pair keeps escalating, route it higher next time. If inline runs were consistently trivial, raise the Sonnet thresholds.

## Gotchas
- Delegating to Codex, working in git worktrees, a Haiku run that refused or stopped early, or an alias that resolves to the wrong model: read `reference/gotchas.md` first.
