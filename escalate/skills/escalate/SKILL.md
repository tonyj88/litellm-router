---
name: escalate
description: Hand a stuck problem to the escalation-reviewer subagent as a compact brief, then continue the task with its recommendation. Run only when the user types /escalate.
disable-model-invocation: true
---

# /escalate

You are the worker. You keep ownership of the task. Do not change the session model. A stronger reviewer model will diagnose one problem and send back a recommendation. You stay responsible for implementing it.

## When to recommend /escalate

Suggest it to the user (don't run it yourself) when one of these is true:

- Two materially different approaches have failed.
- Targeted investigation hasn't found the root cause.
- You're repeating the same diagnosis or fix.
- Evidence contradicts an important assumption in `plan.md`.
- The fix needs a consequential architecture, security or design decision.

Don't suggest it for a single failed command or test, a typo, a straightforward bug, needing to read more files, or after only one attempt.

## Step 0: Check the session model

Read the `model:` line of the `escalation-reviewer` agent file (`.claude/agents/escalation-reviewer.md` in the project if it exists, otherwise `~/.claude/agents/escalation-reviewer.md`). That is the reviewer model.

Compare it with this session's model ID, as stated in your system prompt. Ignore suffixes such as `[1m]`. If the reviewer line is an alias (`opus`, `sonnet`, `haiku`), treat it as matching any session model of that family.

If they match, stop and tell the user:

```
This session is already running the reviewer model (<id>), so /escalate would ask the same model.
Gather more evidence, or start a fresh session instead.
```

Don't write a brief or start the reviewer.

If you can't determine your own model ID, or can't read the agent file, continue to Step 1 and mention the uncertainty in the brief's Current State.

## Step 1: Write the brief

Write the brief from what you already know. Don't paste the conversation transcript. Keep it under 1,200 words. If space is tight, keep these first, in this order: evidence, failed attempts, code locations, the question.

```markdown
# Escalation Brief

## Objective
## Success Criteria
## Current State
## Evidence
Exact errors, test output, logs. Quote them; don't paraphrase.
## Relevant Code
Paths and function or symbol names.
## Attempts
### Attempt 1
What was tried, what happened, why it seems insufficient.
### Attempt 2
## Plan Assumption Conflict
Only if relevant: what plan.md assumed vs. what you found.
## Current Hypotheses
## Exact Question for Reviewer
```

Save it to `~/.claude/escalations/<repo-folder-name>/<YYYYMMDD-HHMMSS>/brief.md`. If saving fails, say so and continue. Saving is for auditing and isn't required.

## Step 2: Call the reviewer

Start the `escalation-reviewer` subagent. Pass the full brief as the prompt. Don't pass a `model` argument. The reviewer's model comes only from its agent file.

If the subagent can't be started, or its model is rejected, stop and report:

```
Configured reviewer model could not be used: <error>
```

Don't retry with another model and don't answer the question yourself.

The result begins with a `Reviewer Model` section. If it doesn't match the agent file's `model:` line (using the alias rule from Step 0), report `Configured reviewer model could not be used: reviewer reported <id>` and don't act on the result.

Save the reviewer's reply as `result.md` next to `brief.md`.

## Step 3: Continue

1. Read the result and check it against the evidence you already have.
2. If it conflicts with stronger evidence you have, say so and explain why you're not following it.
3. If `Confidence` is `Low`, don't act on it. Tell the user:

   ```
   Reviewer confidence is low.
   The unresolved question is: ...
   Additional evidence needed: ...
   ```

   Then let the user choose: gather more evidence, run /escalate again later, or start a stronger session. Don't escalate again on your own.
4. Otherwise, implement the recommendation, run its verification steps, and continue the original task. Don't restart the investigation.
