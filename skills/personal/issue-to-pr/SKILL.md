---
name: issue-to-pr
description: Take one ready-for-agent issue to a reviewed, CI-green pull request without supervision.
disable-model-invocation: true
---

# Issue to PR

Takes one issue from the tracker to a pull request that is ready for a human to merge. Runs unattended: where a step would ask the user, stop instead (see "Stopping"). Never merge.

The issue tracker, triage labels and PR conventions are in `docs/agents/` on the default branch (`git show origin/<default>:docs/agents/issue-tracker.md`; the local checkout may be on another branch). If they are missing, stop and say the repo needs onboarding.

For a plan of several dependent tickets on one integration branch, use `implement-spec` instead.

## 1. Check the gate, then claim

`gh issue view <n> --json state,labels,assignees,blockedBy,closedByPullRequestsReferences,body,comments`. Continue only if all hold:

- The issue is open and labelled `ready-for-agent` (or the tracker's mapping for it).
- It is unassigned, or assigned to you.
- No blocker is open: every `blockedBy` node and every "Blocked by #m" in the body is closed.
- No linked pull request is open (`closedByPullRequestsReferences`).
- It states acceptance criteria or a testable outcome.

If any fails, stop and report which, changing nothing on the tracker. Otherwise claim it with `gh issue edit <n> --add-assignee @me`, then re-read the assignees; if someone else is also assigned, stop.

## 2. Branch and draft PR

- `git fetch origin`, then `git worktree add --no-track -b issue-<n>-<slug> <repo>/worktrees/issue-<n>-<slug> origin/<default>`, and push with `git push -u origin issue-<n>-<slug>` (a tracking branch would push to the default branch).
- If `.pre-commit-config.yaml` exists and `git rev-parse --git-path hooks/pre-commit` names no file, run `uvx pre-commit install`.
- Make the first commit as soon as there is real work, push, and open a **draft** PR the way `docs/agents/issue-tracker.md` says, so the board shows the issue in progress. A change that fits in one commit opens the draft with it. If draft PRs are unavailable, open a normal PR titled `[WIP] ...`.

## 3. Implement

Read the code around the change before writing any. Call the Skill tool with `tdd` where the change has a testable seam. Choose the seams yourself from the acceptance criteria (this overrides `tdd`'s step of confirming them with the user), and list them in the PR body as judgement calls. Run single test files as you go, and the repo's default test run (what CI runs, with the environment its workflow sets and dummy values for its secrets) once before step 4; never tests that need live services or real credentials. If a test fails, run it on `origin/<default>`: a failure there too is pre-existing, so note it in the PR body and continue.

Then **exercise** the change the way a user would reach it: run the CLI command, call the endpoint, render the page. For a config or docs change, observe its effect instead (the collected tests, the rendered page). Passing tests are not evidence the feature works. Use dummy credentials only, and run nothing with outward side effects (deploys, writes to shared databases, messages).

Done when every acceptance criterion is met and exercised.

## 4. CI, review, fix

Up to three review rounds:

1. Push and wait for CI. Wait until `gh pr checks <pr>` lists checks (if none appear within five minutes, treat the repo as having no CI), then `gh pr checks <pr> --watch`. On failure, find the run with `gh run list --branch <branch>`, read `gh run view <id> --log-failed`, fix and push; after three failed fixes in one round, stop. If the repo has no CI, run the full suite and `pre-commit run --all-files` locally instead and say so in the PR body.
2. Call the Skill tool with `code-review`, with `origin/<default>` as the fixed point, issue #n as the spec, and the worktree path. Tell it the reviewers are read-only: no checkout, edit or commit in the worktree. Its reviewers run as subagents that did not write the change. Wait for both reports before going on: dispatch them so you block on their results (in Claude Code, `run_in_background: false`), and never end your turn while one is running.
3. Fix every hard Standards violation and every Spec finding (missing, partial, wrong or unrequested). Fix each judgement-call finding or answer it in the PR body. A finding that needs a maintainer decision (for example a deliberate deviation from the issue) is not fixed by you: list it as a question, and it keeps the PR a draft. Run another round only if this round's fixes changed behaviour.

Done when CI is green and no hard Standards violation or Spec finding is open, or three rounds are spent.

## 5. Hand over

Rewrite the PR body (call the Skill tool with `pr` if available), keeping the issue link line from step 2: what changed and why, how you exercised it, the seams you chose, review findings left open, and every other judgement call you made without asking. Once everything is pushed, remove the worktree (`--force` is fine then; it only discards ignored files such as `.venv`).

Mark the PR ready (`gh pr ready <pr>`) only if CI is green on the last pushed commit, no hard Standards violation or Spec finding is open, and the review ran in subagents. Otherwise leave it as a draft, list what is open, and comment the same on the issue.

Stop there. The human merges.

## Stopping

When you stop early, or are made to report before finishing: if a draft PR exists, put the reason at the top of its body; comment the reason on the issue; remove the worktree; and keep the assignment, so the issue shows who stopped and why. End your final message with the same reason.
