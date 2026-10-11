---
name: issue-to-pr
description: Take one ready-for-agent issue to a reviewed, CI-green pull request without supervision.
disable-model-invocation: true
---

# Issue to PR

Takes one issue from the tracker to a pull request that is ready for a human to merge. Runs unattended: where a step would ask the user, stop instead (see "Stopping"). Never merge.

The issue tracker, triage labels and PR conventions are in `docs/agents/`. If they are missing, stop and say the repo needs onboarding.

For a plan of several dependent tickets on one integration branch, use `implement-spec` instead.

## 1. Check the gate, then claim

`gh issue view <n> --json state,labels,assignees,blockedBy,closedByPullRequestsReferences,body,comments`. Continue only if all hold:

- The issue is open and labelled `ready-for-agent` (or the tracker's mapping for it).
- It is unassigned, or assigned to you.
- No blocker is open: every `blockedBy` node and every "Blocked by #m" in the body is closed.
- No linked pull request is open (`closedByPullRequestsReferences`).
- It states acceptance criteria or a testable outcome.

Then claim it: `gh issue edit <n> --add-assignee @me`.

## 2. Branch and draft PR

- `git fetch origin`, then create a worktree at `<repo>/worktrees/issue-<n>-<slug>` on a new branch `issue-<n>-<slug>` from `origin/<default>`.
- If `.pre-commit-config.yaml` exists and `.git/hooks/pre-commit` does not, run `uvx pre-commit install`.
- Make the first commit as soon as there is real work, push, and open a **draft** PR the way `docs/agents/issue-tracker.md` says, so the board shows the issue in progress. If draft PRs are unavailable, open a normal PR titled `[WIP] ...`.

## 3. Implement

Read the code around the change before writing any. Call the Skill tool with `tdd` where the change has a testable seam. Choose the seams yourself from the acceptance criteria instead of confirming them with the user, and list them in the PR body as judgement calls. Run single test files as you go, and the full suite once before step 4.

Then **exercise** the change the way a user would reach it: run the CLI command, call the endpoint, render the page. Passing tests are not evidence the feature works. Use dummy credentials only, and run nothing with outward side effects (deploys, writes to shared databases, messages).

Done when every acceptance criterion is met and exercised.

## 4. CI, review, fix

Up to three review rounds:

1. Push and wait for CI. Wait until `gh pr checks <pr>` lists checks, then `gh pr checks <pr> --watch`. On failure, find the run with `gh run list --branch <branch>`, read `gh run view <id> --log-failed`, fix and push; after three failed fixes in one round, stop. If the repo has no CI, run the full suite and `pre-commit run --all-files` locally instead and say so in the PR body.
2. Call the Skill tool with `code-review`, with `origin/<default>` as the fixed point and issue #n as the spec. Its reviewers run as subagents that did not write the change.
3. Fix every hard Standards violation and every Spec finding (missing, partial, wrong or unrequested). Fix each judgement-call finding or answer it in the PR body. If the round found nothing to fix, stop repeating.

Done when CI is green and no hard Standards violation or Spec finding is open, or three rounds are spent.

## 5. Hand over

Update the PR body: what changed and why, how you exercised it, the seams you chose, review findings left open, and every other judgement call you made without asking. Remove the worktree.

Mark the PR ready (`gh pr ready <pr>`) only if CI is green, no hard Standards violation or Spec finding is open, and the review ran in subagents. Otherwise leave it as a draft, list what is open, and comment the same on the issue.

Stop there. The human merges.

## Stopping

When you stop early: if a draft PR exists, put the reason at the top of its body; comment the reason on the issue; remove the worktree; and keep the assignment, so the issue shows who stopped and why. End your final message with the same reason.
