---
name: issue-to-pr
description: Take one named ready-for-agent issue to a reviewed, CI-green pull request. Use when the user or a calling skill gives a specific issue number to implement through to a PR, or to resume one parked earlier.
---

# Issue to PR

Takes one issue from the tracker to a pull request that is ready for a human to merge. Never merge.

## Decisions

A decision is the user's only when nothing an agent can reach settles it: not the issue and its comments, ADRs, docs, the code, nor a second agent asked to weigh the evidence. Settle everything else yourself and record it as a judgement call in the PR body.

How a user decision is handled depends on the mode, given as an argument after the issue number:

- **Default:** bring it back to the user. Ask, with the options and your recommendation, wait for the answer, record it on the PR, and continue. If you are a subagent, return the question to your caller and stop until you are resumed with the answer.
- **`autonomous`** (only when the user or a calling skill asks for it): never wait. **Park** the issue: stop as in "Stopping", with the question, its options and your recommendation as the reason. The human answers on the issue and swaps the label back; re-running the skill on the issue resumes the work (see step 1).

The issue tracker, triage labels and PR conventions are in `docs/agents/` on the default branch (`git show origin/<default>:docs/agents/issue-tracker.md`; the local checkout may be on another branch). If they are missing, stop and say the repo needs onboarding.

For a plan of several dependent tickets on one integration branch, use `implement-spec` instead.

## 1. Check the gate, then claim

`gh issue view <n> --json state,labels,assignees,blockedBy,closedByPullRequestsReferences,body,comments`. Continue only if all hold:

- The issue is open and labelled `ready-for-agent` (or the tracker's mapping for it).
- It is unassigned, or assigned to you.
- No blocker is open: every `blockedBy` node and every "Blocked by #m" in the body is closed.
- No linked pull request is open, except a draft on this skill's own branch `issue-<n>-*` (a resume). Check each `closedByPullRequestsReferences` entry with `gh pr view <url> --json state,isDraft,headRefName`.
- It states acceptance criteria or a testable outcome.

If any fails, stop and report which, changing nothing on the tracker. Otherwise claim it with `gh issue edit <n> --add-assignee @me`, then re-read the assignees; if someone else is also assigned, unassign yourself and stop, changing nothing else on the tracker.

**Resume** when `git ls-remote --heads origin 'issue-<n>-*'` names a branch: it is this skill's own stopped work. Keep that branch name in step 2, and read the issue comments: an answer to the question it was parked on settles that decision.

## 2. Branch and draft PR

- `git fetch origin`, then `git worktree add --no-track -b issue-<n>-<slug> <repo>/worktrees/issue-<n>-<slug> origin/<default>`, and push with `git push -u origin issue-<n>-<slug>` (a tracking branch would push to the default branch). When resuming, after `git fetch origin` run `git worktree add -B <branch> <repo>/worktrees/<branch> origin/<branch>` instead (this resets any stale local copy) and skip the push; skip opening a draft PR if one is open.
- If `.pre-commit-config.yaml` exists and `git rev-parse --git-path hooks/pre-commit` names no file, run `uvx pre-commit install`.
- Make the first commit as soon as there is real work, push, and open a **draft** PR the way `docs/agents/issue-tracker.md` says, so the board shows the issue in progress. A change that fits in one commit opens the draft with it. If draft PRs are unavailable, open a normal PR titled `[WIP] ...`.

## 3. Implement

Read the code around the change before writing any. Call the Skill tool with `tdd` where the change has a testable seam. Choose the seams yourself from the acceptance criteria (this overrides `tdd`'s step of confirming them with the user), and list them in the PR body as judgement calls. Format and lint only the files you changed (the hooks do this on commit). Run single test files as you go, and the repo's default test run (what CI runs, with the environment its workflow sets and dummy values for its secrets) once before step 4; never tests that need live services or real credentials. If a test fails, run it on `origin/<default>`: a failure there too is pre-existing, so note it in the PR body and continue.

Then **exercise** the change the way a user would reach it: run the CLI command, call the endpoint, render the page. For a config or docs change, observe its effect instead (the collected tests, the rendered page). Passing tests are not evidence the feature works. Use dummy credentials only, and run nothing with outward side effects (deploys, writes to shared databases, messages).

Done when every acceptance criterion is met and exercised.

## 4. CI, review, fix

Up to three review rounds:

1. Push and wait for CI. Wait until `gh pr checks <pr>` lists checks (if none appear within five minutes, treat the repo as having no CI), then `gh pr checks <pr> --watch`. On failure, find the run with `gh run list --branch <branch>`, read `gh run view <id> --log-failed`, fix and push; after three failed fixes in one round, stop. If the repo has no CI, run the full suite and `pre-commit run --all-files` locally instead and say so in the PR body.
2. Call the Skill tool with `code-review`, with `origin/<default>` as the fixed point, issue #n as the spec, and the worktree path. Tell it the reviewers are read-only (no checkout, edit or commit in the worktree), and give them the findings already answered in the PR body so they are not raised again. Its reviewers run as subagents that did not write the change. Wait for both reports before going on: dispatch them so you block on their results, and never end your turn while one is running.
3. Fix every hard Standards violation and every Spec finding (missing, partial, wrong or unrequested). Fix each judgement-call finding or answer it in the PR body. A finding that needs a user decision (see "Decisions"; for example a deliberate deviation from the issue) is handled as that section says. A finding that a decision already recorded on the issue or PR settles is answered in the PR body by citing that decision. Run another round only if this round's fixes changed behaviour.

Done when CI is green and no hard Standards violation or Spec finding is open, or three rounds are spent.

## 5. Hand over

Rewrite the PR body (call the Skill tool with `pr` if available), keeping the issue link line from step 2: what changed and why, how you exercised it, the seams you chose, review findings you answered rather than fixed, and every other judgement call you made without asking. Once everything is pushed, remove the worktree (`--force` is fine then; it only discards ignored files such as `.venv`).

Mark the PR ready (`gh pr ready <pr>`) only if CI is green on the last pushed commit, no hard Standards violation, Spec finding or user decision is open, and the review ran in subagents. Otherwise leave it as a draft and stop as in "Stopping", with what is open as the reason.

Stop there. The human merges.

## Stopping

When you stop early, or are made to report before finishing:

- Comment the reason on the issue, and swap its `ready-for-agent` label for `needs-info` (or the tracker's mappings), so the issue leaves the agent queue until a human clears it.
- If a draft PR exists, link that comment at the top of its body.
- Remove the worktree, and keep the assignment, so the issue shows who stopped and why.

End your final message with the same reason.
