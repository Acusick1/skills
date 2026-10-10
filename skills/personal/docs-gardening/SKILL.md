---
name: docs-gardening
description: Review or rewrite a repo's human-facing Markdown docs against the loaded documentation rules. Use when reviewing documentation changes in a diff or PR, when acting as the non-author reviewer of a change that touches docs or behaviour, or when the user asks to garden, tidy, refresh or audit docs.
---

# Docs gardening

The process for applying the documentation rules in the loaded instructions (`AGENTS.md` and global instructions) to human-facing Markdown docs. This skill adds no rules of its own. If no documentation rules are loaded, ask the user for them; if you cannot ask, report that and stop.

Out of scope: agent instruction files (`AGENTS.md`, `CLAUDE.md`, `docs/agents/`, skills), ADR bodies under `docs/adr/`, and short-lived docs (implementation plans, progress trackers). For agent instruction files, call the Skill tool with "writing-for-agents".

## Pick the mode

- **Review**: a diff, branch or PR is given, or you are the non-author reviewer of a change. Report findings; edit nothing.
- **Garden**: the user asks for docs to be refreshed, tidied or audited. Rewrite in place and open a PR.
- **Otherwise** (you noticed a stale or contradictory doc while doing something else): name the problem in one sentence and offer a Garden pass. Start nothing.

## 1. Gather

- **Review:** the base is the PR's base branch, else the merge-base with the default branch. Read `git diff <base>` (which includes uncommitted work), covering both the changed docs and the changed code, since a behaviour change can make an unchanged doc wrong.
- **Garden:** the pages the user named, or every in-scope page reachable from `README.md` plus the rest of `docs/`.
- For each page, also read the pages it links to and the pages that link to it. Duplication and contradiction only show up across pages.
- For a large scope, delegate claim verification to subagents, one per group of related pages, before rewriting anything. Each returns the page's checkable claims marked verified or wrong, with the evidence.

## 2. Mechanical checks

Run whatever the repo configures for prose and links (for example Vale, a link checker, a docs CI job). If none is configured, skip this step. In Review, report what they find; in Garden, fix it. Do not repeat their findings as judgement calls.

## 3. Judge each page

Read the whole page, start to finish, before judging any part. Then check it against every loaded documentation rule, one rule at a time, recording each violation with the rule it breaks.

When checking that a page matches the code, verify every path, command, flag, config key and default it states, using `ls`, `grep`, the source, or a command's `--help`. Never run a command with side effects to check a doc. If `--help` fails because required configuration is missing, set dummy values for the required environment variables; never use real credentials.

Where a fact's authoritative home is code, config, or an agent prompt file, the page links to it rather than restating it.

## 4a. Review: report

List findings, most severe first. Each names the page, the section heading, the rule broken, and the fix. A behaviour change in the diff with no matching reference-page update is always a finding. If nothing is wrong, say so.

## 4b. Garden: rewrite

Work in a worktree on a new branch.

- Rewrite each page in place. Prefer restructuring over appending.
- A page whose sections serve different Diátaxis purposes stays whole: splitting creates new pages, which are opt-in. Recommend the split in the PR body.
- Before committing a restructured page, check that every number, date and cited result in the old text either remains or appears in the commit's removal list. Dated evidence is the first thing a restructure loses.
- One commit per source page, including the edits its moved content causes in destination pages. The commit body lists every piece of removed content and where it went: a linked page, a backlog item for the PR body, or "deleted because wrong" with the evidence.
- Never add rationale. List hard-to-reverse decisions that have no ADR as ADR candidates for the PR body.
- Backlog content removed from pages goes into the PR body as a list. Create issues for it only after the user approves.
- Re-run the mechanical checks. Then dispatch a subagent that did not write the changes to run this skill in Review mode against the branch. Fix what it finds in one follow-up commit whose body lists any further removals; do not rewrite the page commits.
- If the gardened pages now contradict an out-of-scope file (for example `AGENTS.md`), list each contradiction in the PR body and offer a writing-for-agents pass.
- Open the PR. Its body summarises each page's changes and lists the backlog items, the ADR candidates, recommended splits, and anything you were unsure of. Never merge.
