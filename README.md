# litellm-router

Claude Code skills and agents for working through our LiteLLM gateway. Each skill is self-contained: it has its own folder, its own install steps, and its own README.

The skills assume you run Claude Code against the gateway with API billing, not a Claude subscription. Use full model IDs (`claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`). The bare `sonnet` and `haiku` aliases may resolve to a different model on a gateway.

## Which skill do you need?

Both skills move work to a cheaper or stronger model. They differ in when they run and what they send.

| | `/tokenomics` | `/escalate` |
|---|---|---|
| When it runs | After you draft a plan | When a worker gets stuck on one problem |
| Where you run it | An Opus session | A Sonnet or Haiku session |
| What it does | Splits the plan into tasks and sends each task to the cheapest tier that can do it | Asks the Opus reviewer for a diagnosis. The worker applies the fix |
| Who does the work | Subagents (`worker`, `scout`, `grunt`, `Explore`) | The worker, in its own session |
| Who it's for | Work you want done cheaply, in bulk | One hard problem that a cheaper model can't solve |

Think of it this way. `/tokenomics` decides who does the work, and `/escalate` gets a second opinion when the worker is stuck.

## Skills

### tokenomics

Routes a finished plan across Opus, Sonnet, and Haiku. Main keeps the hard work. Routine, well-specified tasks go to subagents with a clear "done when".

Run it from an Opus session, after the plan is drafted. Opus is the main session's model, so it stays on the planning and the judgment calls. The skill then routes each task:

- `worker` (Sonnet) takes well-specified work that needs real reasoning or non-trivial code, such as a feature built to a spec or a multi-file refactor.
- `scout` (Haiku) takes routine changes that a test or command can prove, such as a bug fix with a failing test.
- `grunt` (Haiku, low effort) takes mechanical work that follows a pattern you give it, such as renames or boilerplate.
- `Explore` (Haiku) takes read-only searches, such as "where is this function called?".

The repo has no Opus agent on purpose. Opus is Main, and sending work to a second Opus would cost more than running it inline. Every agent sits below Main. Work that needs Opus stays in the main session.

If the main session is Sonnet or Haiku, tokenomics has less to do. A Sonnet session delegates only Haiku-tier work. A Haiku session runs everything inline. Tokenomics never sends work up a tier.

- Install: [tokenomics/INSTALL.md](tokenomics/INSTALL.md)
- Skill and cost model: [tokenomics/skills/tokenomics/](tokenomics/skills/tokenomics/)

### escalate

Asks a stronger reviewer model for a diagnosis when a cheaper worker model is stuck. The worker keeps its session and applies the fix.

Run it from a Sonnet or Haiku session when the worker has failed on one problem more than once and can't find the cause. The reviewer is `escalation-reviewer`, pinned to `claude-opus-5-5`. It reads a short brief and returns a diagnosis after checking it against the code. The worker then applies the fix in its own session. The reviewer can't edit files.

Use `/escalate` for a single hard question. Don't use it to hand off a whole task. For that, start a new Opus session with the plan.

- Install: [escalate/INSTALL.md](escalate/INSTALL.md)
- How it works: [escalate/README.md](escalate/README.md)

### Using both together

1. Start an Opus session and draft the plan.
2. Run `/tokenomics` to route the plan. The routed tasks run as subagents.
3. If a worker gets stuck in a Sonnet or Haiku session, run `/escalate` to get Opus's diagnosis.

Don't switch the main session's model with `/model` to get Opus mid-task. That reloads the whole conversation at the new model's cache price. `/escalate` gets Opus's help without changing the main session's model.

## Layout

```
escalate/        /escalate skill, reviewer agent, install notes
tokenomics/      /tokenomics skill, reference docs, subagent definitions, install notes
```

## Install

Each skill's `INSTALL.md` lists the files to copy into `~/.claude`. Install one skill at a time and start a new session after each install, because agents load at session start.
