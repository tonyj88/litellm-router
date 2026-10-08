# litellm-router

Claude Code skills and agents for working through our LiteLLM gateway. Each skill is self-contained: it has its own folder, its own install steps, and its own README.

The skills assume you run Claude Code against the gateway with API billing, not a Claude subscription. Use full model IDs (`claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`). The bare `sonnet` and `haiku` aliases may resolve to a different model on a gateway.

## Skills

### escalate

Asks a stronger reviewer model for a diagnosis when a cheaper worker model is stuck. The worker keeps its session and applies the fix.

- Install: [escalate/INSTALL.md](escalate/INSTALL.md)
- How it works: [escalate/README.md](escalate/README.md)

### tokenomics

Routes a finished plan across Opus, Sonnet, and Haiku. Main keeps the hard work. Routine, well-specified tasks go to subagents with a clear "done when".

- Install: [tokenomics/INSTALL.md](tokenomics/INSTALL.md)
- Skill and cost model: [tokenomics/skills/tokenomics/](tokenomics/skills/tokenomics/)

## Layout

```
escalate/        /escalate skill, reviewer agent, install notes
tokenomics/      /tokenomics skill, reference docs, subagent definitions, install notes
```

## Install

Each skill's `INSTALL.md` lists the files to copy into `~/.claude`. Install one skill at a time and start a new session after each install, because agents load at session start.
