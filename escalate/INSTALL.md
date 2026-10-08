# Installing /escalate

1. Copy the files into your user Claude folder:

   ```bash
   mkdir -p ~/.claude/skills/escalate ~/.claude/agents
   cp escalate/skills/escalate/SKILL.md ~/.claude/skills/escalate/
   cp escalate/agents/escalation-reviewer.md ~/.claude/agents/
   ```

The reviewer is `claude-opus-5-5`, already set in the agent file. You can run `/escalate` from any session whose model differs from the reviewer, such as `claude-sonnet-5-5` or `claude-haiku-5-5`. To use a different reviewer, change the `model:` line in `~/.claude/agents/escalation-reviewer.md`, spelled exactly as it appears in your `/model` list.

## First test: does the frontmatter model route?

Selecting a model with `/model` doesn't prove that a subagent's `model:` field accepts the same name. Test this before anything else.

1. Start a session with the worker model and run `/escalate` on a small made-up problem.
2. Check the LiteLLM request logs to see which model served the reviewer call.
3. If it was the reviewer model, you're done. If it errored or used another model, set `ANTHROPIC_DEFAULT_OPUS_MODEL=<reviewer-name>` in your environment and change the line to `model: opus`. That makes every `opus` request in that environment go to the reviewer model, so only do this if you don't use `opus` for anything else. With this fallback, the skill's session-model check treats any Opus session as matching the reviewer. That is intended.

## Changing the reviewer

Edit the `model:` line in `escalation-reviewer.md`. Nothing else needs to change. If your project has its own `.claude/agents/escalation-reviewer.md`, it overrides the user-level file, so edit that one instead.
