# Tokenomics: delegation gotchas (LiteLLM build)

## Model routing through the LiteLLM gateway

- **Use full model IDs, never bare aliases.** Agent files pin `claude-sonnet-5-5` and `claude-haiku-5-5`. On a custom base URL, the bare `sonnet` or `haiku` alias may map to `ANTHROPIC_DEFAULT_*_MODEL` or to the legacy `ANTHROPIC_SMALL_FAST_MODEL`, which can be a non-Claude model. Verify the transcript after any change (see below).
- **Verify each alias once.** Spawn `worker`, `scout`, `grunt`, and `Explore` with a trivial "reply OK" task. Check the `model` field in each subagent transcript under `~/.claude/projects/`. It must match the pinned ID.
- **Don't set `CLAUDE_CODE_SUBAGENT_MODEL`.** It overrides every agent's `model:` line, so all tiers would collapse to one model.
- **Main is chosen per session with `/model`.** `settings.json` `model` only sets the default. Mid-session switches cold-start the cache.
- **Agents load at session start.** An agent file added or edited mid-session isn't picked up until a new session starts.
- **The built-in `Explore` wins if the override isn't installed.** `~/.claude/agents/explore.md` must exist. Without it, `Explore` runs on Main's model, which is expensive for cheap searches.

## Haiku 5.5 subagents

From the move to Haiku 5.5 (2026-10-07) and its migration guide in the claude-api skill.

- **Refusals have no fallback.** Haiku 5.5 runs safety classifiers that can decline a benign request (categories include `cyber` and `general_harms`), and unlike the larger models it has no server-side fallback. Re-run the package on Sonnet with the same brief; it doesn't count as the task's escalation.
- **Set effort explicitly.** Haiku 5.5 defaults to medium effort. An agent file without `effort:` runs at medium, so `grunt` needs `effort: low` to be the cheap tier.
- **Low effort can stop early.** With a long agent system prompt at low effort, Haiku 5.5 sometimes hands the task back before it's done. If `grunt` reports unfinished work without a reason, re-run it on `scout`; that counts as the escalation.
- **Haiku 5.5 often skips skills.** It tends to do the task itself instead of loading the skill whose description matches. In the 2026-10-07 trigger eval, it loaded the right skill on 37 of 76 prompts, against 69 of 76 for Haiku 4.5 with the same descriptions. If a Haiku package needs a skill, name it in the brief.

## Cache and billing on the gateway

- **Cache TTL is 5 minutes for both Main and subagents** unless the proxy is confirmed to honor 1-hour writes. Don't rely on the subscription's 1-hour main-session cache.
- **Billing is set by LiteLLM.** The price tables in `economics.md` are list prices used for relative cost. Check the gateway's spend logs for real figures.

## Codex and git worktrees

Problems hit while delegating to Codex in git worktrees and merging the
resulting stacked PRs. Read this before delegating to Codex or using worktrees.

- **Codex in the background waits on stdin.** A backgrounded `codex exec` with no TTY prints "Reading additional input from stdin..." and does nothing. Append `< /dev/null` to the command.
- **Codex can't commit in a git worktree.** Its `workspace-write` sandbox can't reach the git metadata, which lives outside the worktree. Tell it not to commit, and commit yourself after review.
- **Codex leaves locked pytest folders.** Cache and temp folders created in its sandbox get ACLs that block deletion, so `git worktree remove` fails. Have it run `python -m pytest -p no:cacheprovider`. If folders are already locked, the user runs `takeown /f <dir> /r /d y` from an elevated shell, then deletes them.
- **Deleting the base of stacked PRs closes them.** `gh pr merge --delete-branch` on a PR that other PRs target closes those PRs instead of retargeting them, and a closed PR whose base is gone can't be reopened. Merge the base PR without `--delete-branch`. Then retarget each stacked PR with `gh pr edit <n> --base main`, merge `main` into its branch (a plain merge, not a rebase and force-push), and delete the base branch afterwards.
