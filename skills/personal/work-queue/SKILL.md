---
name: work-queue
description: Work through a repo's ready-for-agent issues one after another via issue-to-pr, optionally merging each ready PR. Usage `/work-queue [merge] [issues=<n>,<n> | label=<label>]`.
disable-model-invocation: true
---

# Work queue

Drains a repo's `ready-for-agent` issues, one at a time, in this session's repo. Invoking it is the user's authorisation to run `issue-to-pr` on every candidate and, with `merge`, to merge what it marks ready. Without `merge`, merge nothing.

Arguments:

- `merge`: merge PRs that `issue-to-pr` marks ready.
- `issues=<n>,<n>`: only these issues are candidates.
- `label=<label>`: only issues that also carry this label are candidates.

The triage labels and issue tracker are in `docs/agents/` on the default branch (`git show origin/<default>:docs/agents/triage-labels.md`; the local checkout may be on another branch). Use its names wherever this skill says `ready-for-agent` or `needs-info`. If the files are missing, stop and say the repo needs onboarding.

Eligibility belongs to the `issue-to-pr` gate alone. Hand it every candidate and let it accept or refuse.

## 1. List candidates

Open issues labelled `ready-for-agent`, narrowed by `issues=` and `label=`, lowest number first, minus every issue already tried this run. Keep the tried list for the whole run, with each issue's outcome.

Done when the list is built. If it is empty, go to "Summary".

## 2. Work the first candidate

Dispatch a subagent whose instruction is: "Call the Skill tool with `issue-to-pr`, passing `<n> autonomous`." One subagent at a time. Add `<n>` to the tried list.

Wait for it to finish. Then read the outcome from the tracker alone, ignoring the subagent's message:

- **ready**: a PR with head branch `issue-<n>-*` is open and not a draft.
- **parked**: the issue carries `needs-info`. The latest issue comment holds the question or the reason it stopped.
- **refused**: the issue still carries `ready-for-agent` and has no ready PR.

Done when the issue is classified into one outcome.

## 3. Merge (with `merge` only)

For a **ready** PR whose last commit has green CI (`gh pr checks <pr>`; a repo with no checks counts as green):

1. Merge it with an allowed method (`gh repo view --json squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed`), squash where several are allowed. Record **merged**.
2. Wait for the post-merge workflow runs on the default branch: `gh run list --commit <merge sha>` until runs appear (none within five minutes means the repo has no post-merge workflows), then `gh run watch <id>` for each.
3. If any run fails, record the failing run's link and stop the loop (see "Summary").

Record any other **ready** PR as ready (unmerged).

Done when the PR is merged and every post-merge run is green, or it is recorded as unmerged.

## 4. Repeat

Go to step 1. A merge can unblock dependents, so the list is rebuilt each time.

## Summary

Stop when no candidates remain or a post-merge run fails. Report the tried issues in order, each with its outcome and link:

- merged (PR link)
- ready, unmerged (PR link)
- parked (link to the question or reason comment)
- refused by the gate

Then state why the loop stopped: queue empty, or the failing post-merge run's link.
