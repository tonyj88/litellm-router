# PLAN: /escalate skill v0.1

## Done
- [x] `escalate/skills/escalate/SKILL.md`. Accepted when it runs only on `/escalate`, writes a brief of 1,200 words or fewer, calls the reviewer, and handles low-confidence results.
- [x] `escalate/agents/escalation-reviewer.md`. Accepted when it's read-only (Read/Grep/Glob) and the `model:` line is the only setting to change.
- [x] `escalate/INSTALL.md`

## Done (work machine)
- [x] Installed to `~/.claude`, reviewer set to `claude-opus-5-5`. Added a guard so the skill stops when the session model equals the reviewer model (read from the agent file), and the reviewer now reports its model ID so the skill can verify it. Applied after an `/escalate` self-review.

## Next (on the work machine)
- [x] Sonnet session: `/escalate` reviewer call confirmed as `claude-opus-5-5` in the LiteLLM logs.
- [ ] Repeat from a `claude-haiku-5-5` session. If a call uses another model, use the `opus` alias fallback described in INSTALL.md.
- [ ] Forced end-to-end test: the worker gets stuck, runs `/escalate`, applies the result, and the tests pass.
- [ ] Change `model:` to a second model and confirm no other file needed to change.

## Open decisions
- Whether frontmatter `model:` accepts LiteLLM names directly or needs the alias mapping. Answered by the first test. Answered for Sonnet workers: full IDs are accepted and the LiteLLM logs show `claude-opus-5-5` serving the reviewer. Haiku is still to be checked.
- Dropped from the original plan: `config.yaml` (the Agent tool can't pass arbitrary model IDs), Bash for the reviewer, `omitClaudeMd` (not a confirmed field), and in-repo `.ai/escalation` artifacts.
