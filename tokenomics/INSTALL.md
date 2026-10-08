# Installing /tokenomics (LiteLLM build)

1. Copy the files into your user Claude folder:

   ```bash
   mkdir -p ~/.claude/skills/tokenomics/reference ~/.claude/agents
   cp tokenomics/skills/tokenomics/SKILL.md ~/.claude/skills/tokenomics/
   cp tokenomics/skills/tokenomics/reference/*.md ~/.claude/skills/tokenomics/reference/
   cp tokenomics/agents/*.md ~/.claude/agents/
   ```

2. Point Claude Code at your LiteLLM gateway in `~/.claude/settings.json`. Replace `<your-litellm-gateway>` in `SKILL.md` and `reference/economics.md` with the same URL.

3. Pick the main model per session with `/model`: `claude-opus-5-5`, `claude-sonnet-5-5`, or `claude-haiku-5-5`. Don't use the bare `sonnet`/`haiku` aliases on a gateway.

## Check the routing before relying on it

Start a session, then run one trivial delegation to each agent (`worker`, `scout`, `grunt`, `Explore`) with the task "reply OK". Open each subagent transcript under `~/.claude/projects/` and confirm the `model` field matches the pinned ID in its agent file. Don't use the bare `sonnet`/`haiku` aliases on a gateway. If any transcript shows another model, stop and fix the agent file before using `/tokenomics`.

Agents load at session start, so start a new session after installing or editing them.

## Limits

Prices in `reference/economics.md` are list prices used for relative cost. Your gateway's billing is set by its LiteLLM admin, so check its spend logs for real figures.
