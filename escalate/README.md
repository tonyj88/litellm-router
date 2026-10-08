# How /escalate works

`/escalate` lets a cheap worker model get help from a stronger model on one hard problem, without giving up the worker's session. The worker keeps writing the code. The stronger model only diagnoses the problem.

To install it, see [INSTALL.md](INSTALL.md).

## Why not just switch models?

Running `/model` mid-session puts the expensive model on every remaining turn, and it reads the whole conversation history each time. Starting a fresh strong session loses everything the worker learned. `/escalate` sits between the two. The strong model sees a short summary once, answers, and leaves.

## The flow

1. The worker gets stuck. For example, two different fixes failed, or it can't find the root cause. It suggests `/escalate` to you, but it can't run it by itself. The skill sets `disable-model-invocation: true`, so only you can start a paid escalation.
2. You type `/escalate`.
3. The worker writes a brief of 1,200 words or fewer: the goal, the exact errors, the relevant files, what it tried, and one specific question. It doesn't send the conversation transcript.
4. The worker starts the `escalation-reviewer` subagent with the brief. The subagent begins with an empty context, so it sees the brief and nothing else from the session.
5. The reviewer checks the brief against the code and replies in a fixed format: root cause, confidence, what earlier attempts missed, the recommended approach, the files involved, and how to verify the fix.
6. The worker gets the reply in its own session, with its context still intact. It checks the reply against what it already knows, then applies the fix and runs the verification steps.

The worker saves the brief and the reply to `~/.claude/escalations/<repo>/<timestamp>/` as `brief.md` and `result.md`, so you can audit an escalation later. The files stay outside the repo and never end up in a PR.

## Which model is the reviewer?

The `model:` line in `~/.claude/agents/escalation-reviewer.md` sets it, and it ships as `claude-opus-5-5`. To change reviewers, edit that one line. The worker is whatever model the session runs on, typically `claude-sonnet-5-5` or `claude-haiku-5-5`. The skill never sets it. If the session is already running the reviewer model, the skill stops before writing a brief. The reviewer also reports its own model ID in the result, and the skill rejects the result if it doesn't match the agent file.

The setting lives in the agent file, not a config file, for a practical reason: when the Agent tool starts a subagent, its model override accepts only `sonnet`, `opus`, `haiku`, or `fable`. It can't pass a LiteLLM model name. Agent frontmatter can.

## Guardrails

- **The reviewer can't edit files.** Its only tools are `Read`, `Grep`, and `Glob`.
- **There's no silent fallback.** If the reviewer model fails, the skill reports the error and stops. It doesn't try another model and doesn't answer the question itself.
- **Low confidence goes back to you.** If the reviewer reports `Confidence: Low`, the worker doesn't act on the recommendation. It tells you what's still unknown, and you decide what to do next.
- **There's one hop.** The flow is worker, then reviewer, then worker. A reviewer can't escalate to a stronger reviewer.
- **The brief can be wrong.** A cheaper model wrote it, so the reviewer is told to check the evidence against the code before trusting the worker's theories.
