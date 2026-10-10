---
name: onboard-repo
description: Bring a repo onto the shared agent practices (AGENTS.md, docs/agents config, triage labels, project board, lint over prose) and open a PR.
disable-model-invocation: true
---

# Onboard a repo

Brings one repo onto the shared practices and opens a single PR. Everything that lives on GitHub rather than in the diff (labels, board, issues) is done directly and listed in the PR body.

**Board:** the GitHub Project named `Personal`, owned by the repo owner. If the user names a different board, use that everywhere this skill says `Personal`.

## 1. Explore, then confirm

Read before changing anything:

- `git remote -v`, the default branch, and `git worktree list` (another session may own the main checkout).
- `CLAUDE.md`, `AGENTS.md`, `.claude/CLAUDE.md`, and every reference to them (`git grep -n CLAUDE.md`).
- Where documentation is indexed: `README.md`, `CLAUDE.md`, `docs/`. Note every index, full or partial.
- Ad-hoc backlogs: `TODO.md` (check `git check-ignore`; it is often a local, untracked file), plan files, "open questions" or "future work" sections in docs.
- Lint and hook config: `pyproject.toml` / `ruff.toml`, `.pre-commit-config.yaml`, and whether the hook is installed (`.git/hooks/pre-commit`).
- Open issues (`gh issue list --state open --limit 200`) and which are already on the board (`gh project item-list <n> --owner <owner> --limit 200`).
- Existing labels (`gh label list`).

Present a short summary of what you found and the plan, then wait for the user's go-ahead. List every judgement call you expect to make (see step 9).

## 2. Work in a worktree

Create `<repo>/worktrees/onboard-agent-practices` on a new branch from the default branch's remote head. Add `worktrees/` to `.gitignore` if missing.

## 3. AGENTS.md

- `git mv CLAUDE.md AGENTS.md`. Keep a `CLAUDE.md` only if it has genuinely Claude-specific lines, and then it holds `@AGENTS.md` plus those lines.
- If neither file exists, create `AGENTS.md` with a one-paragraph map of the repo, a pointer to the README index, and only the rules specific to this repo that an agent could not infer from the code. Creating it, and a README if missing, are judgement calls for step 1.
- Update every reference found in step 1. Fix references that were already wrong while you are on the line, and point them at the doc that actually holds the content.
- One documentation index, in `README.md`. Merge any other index into it so no listed doc is lost, and replace the other copy with a one-line pointer to `README.md#documentation`.
- Remove instructions that only tell agents to edit-protect the file or to avoid creating docs; the global instructions cover documentation policy.

## 4. Agent skill configuration

Locate the `setup-matt-pocock-skills` skill folder (the directory holding its `SKILL.md` in your skills directory). Copy three of its templates into `docs/agents/`, then edit them so they describe this repo only:

- `issue-tracker-github.md` saved as `issue-tracker.md`:
  - Create issues with `gh issue create --project "Personal" ...`, and say never to omit `--project`.
  - Add: open a draft PR whose body contains `Closes #<number>` as soon as the branch has its first commit. Linking moves the issue to In Progress on the board; merging moves it to Done.
  - In "Pull requests as a triage surface", keep only the flag line, as a statement (`**PRs as a request surface: no.** External PRs are not triaged as feature requests.`), and delete the setup parenthetical and the instructions for the `yes` case.
  - Keep the wayfinding section; the `wayfinder` skill reads it.
- `triage-labels.md`: delete the closing line telling the reader to edit the column.
- `domain.md`: keep the single-context layout only (`GLOSSARY.md` and `docs/adr/` at the root). Delete the multi-context text and the example file trees.

Add this block to `AGENTS.md`, or update it in place if one exists:

```markdown
## Agent skills

### Issue tracker

GitHub Issues on `<owner>/<repo>`, added to the `Personal` project board. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `GLOSSARY.md` and `docs/adr/` at the repo root, created when first needed. See `docs/agents/domain.md`.
```

Create any of the five labels the repo lacks with `gh label create --force`.

Remove any competing issue workflow in other docs (for example "use the /create-issue skill") and point it at `docs/agents/issue-tracker.md`.

## 5. Prose rules to tooling

For every "always" or "never" rule in `AGENTS.md` and the development docs, check whether a linter, type checker or hook can enforce it.

- Already enforced (for example pre-commit runs the formatter): delete the prose.
- Enforceable with few violations (roughly 10 or fewer): enable the rule, fix the violations, read the surrounding code to confirm each fix keeps behaviour correct, and replace the prose with one line naming the rule.
- Enforceable with many violations: leave the prose, and open an issue with `--project "Personal"` giving the rule, the per-code violation counts (`ruff check --select <RULE> --statistics`), the scope decisions, and a "done when".
- Not mechanically checkable: keep it as prose.

A repo with no linters still gets this pass; record its outcome in the PR body even when nothing changes.

### Docs checks

Copy the shared docs checks from the `agent-config` repo's `docs-checks/` directory:

- `vale/Temporal/` to `.vale/styles/Temporal/`, and `vale.ini` to `.vale.ini`. In the `agent-config` repo itself, copy no styles: point `.vale.ini`'s `StylesPath` at `docs-checks/vale`.
- Append the hooks in `pre-commit-hooks.yaml` to the repo's `.pre-commit-config.yaml`, creating it (with `repos:`) if missing.
- If the repo has no CI job running pre-commit, add `lint.yml` under `.github/workflows/`.

Run `pre-commit run --all-files`. Fix every Vale error and lychee failure in human-facing docs (errors are near-certain narration; rewrite the sentence to state the current state). Leave Vale warnings for a docs-gardening pass. If a file outside the hook's scope trips a check, narrow the hook's `files` pattern rather than editing that file. Document the hooks, their scope, and `pre-commit install` wherever the repo describes its development tooling; if it describes none, add a "Development" section to `README.md`. Install the hook locally with `uvx pre-commit install`.

## 6. Backlog to issues

Delegate this to a subagent if one is available. For each backlog item:

- If an open issue already covers it, record the mapping and create nothing.
- Otherwise create one issue with `--project "Personal"`, a concise imperative title, and the item's full original content (links, issue references, sub-steps) kept verbatim. Note dead references (for example a plan file that no longer exists) rather than dropping them.

Then remove the migrated source: delete tracked files in the PR; for an untracked or ignored file, ask the user before deleting, because it cannot be recovered. Leave "open questions" sections in docs for a docs-gardening pass unless the user asks otherwise.

## 7. Board

- `gh project link <n> --owner <owner> --repo <repo>`.
- Add every open issue not already on the board with `gh project item-add`.

## 8. Review, then PR

Commit with the pre-commit hook running. Before opening the PR, have an agent that did not write the change review `git diff origin/<default>...HEAD` against this skill's steps: references, links and anchors, behaviour of any code fixes, accuracy of new docs against the repo's scripts and config (the source of truth when no prior docs exist), contradictions between `AGENTS.md` and `docs/agents/`, and leftover template text. Fix what it finds in a follow-up commit, judging each finding on its merits (it may not know which skills are installed).

Open the PR. Its body lists the diff changes, the work done outside the diff (labels, board, issue numbers), and the judgement calls. Never merge.

## 9. Judgement calls to surface

Do not decide these silently. Raise them in step 1 and in the PR body:

- Keeping a `CLAUDE.md` alongside `AGENTS.md`.
- Which index survives when two disagree.
- Any lint rule enabled, and any left as an issue instead.
- Backlog items you judged to overlap an existing issue.
- Deleting any untracked file.
