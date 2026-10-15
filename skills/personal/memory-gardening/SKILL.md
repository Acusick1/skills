---
name: memory-gardening
description: Prune and restructure Claude Code's persistent memory: delete stale or derivable memories, merge duplicates, and promote shared rules into AGENTS.md.
disable-model-invocation: true
---

# Memory gardening

Brings a memory directory back to the format your memory instructions define: one fact per file, every fact still true, nothing the repo already records, and a `MEMORY.md` that is only an index.

**Scope:** the memory directory named in your system prompt. If the user says `all`, garden every `~/.claude/projects/*/memory/` in turn, running steps 2 to 5 for each one and checking each memory against the repo that the directory name encodes (`-home-andrew-code-odds` is `/home/andrew/code/odds`).

## 1. Back up

The memory directory is not under version control, so a deletion is permanent. Copy the whole directory into your scratchpad before you change anything, and report the backup path at the end.

## 2. Read everything

Read `MEMORY.md` and every other file in the directory. `MEMORY.md` often holds content of its own (status logs, findings, whole sections); treat each fact in it as a memory in its own right.

## 3. Give every memory a verdict

Check each memory against the repo and give it exactly one verdict. The step is done when every file and every fact in `MEMORY.md` has a verdict and its evidence.

- **Stale:** it names a file, function, flag, issue or decision that no longer exists or no longer holds. Verify with `grep`, `git log`, or `gh issue view`; a mention is stale only after you looked. Verdict: delete, or rewrite to the current truth when the underlying fact still matters.
- **Derivable:** the repo already records it in code, config, `AGENTS.md`/`CLAUDE.md`, an ADR, a docs page, git history, or an issue or PR. Merged-PR logs and "completed #123" status lines are git history. Verdict: delete. If the memory holds the one non-obvious part (the reason, the gotcha), rewrite it down to that part.
- **Duplicate:** two memories carry the same meaning. Verdict: merge into one file and delete the other.
- **Contradiction:** two memories, or a memory and the repo, disagree. Verdict: ask the user which holds; never pick one silently.
- **Shared:** a rule that applies to everyone working in the repo, not just this user. Verdict: promote into the repo's `AGENTS.md` (or a lint rule or hook, when it can be checked mechanically) and delete the memory.
- **Global:** a preference of this user's that holds across every project. Verdict: promote into the user's global instructions file (the one `~/.claude/CLAUDE.md` loads) and delete the memory from every project that carries it.
- **Malformed:** true and useful, but not in the defined format (several facts in one file, missing frontmatter, missing **Why:**/**How to apply:**, a relative date, content sitting in `MEMORY.md`). Verdict: rewrite. A relative date with no anchor is undatable; use the file's modification time, and say so.
- **Keep:** none of the above.

## 4. Confirm the plan

Show the user one table: memory, verdict, evidence, and the resulting action. List promotions with the exact text and target file. Wait for approval; the user may overrule any row.

## 5. Apply

- Delete, merge and rewrite memory files as approved.
- Make promotions to a repo through that repo's normal change flow (branch or worktree, commit, PR), never by editing a checkout another session may own. Edit the global instructions file in place.
- Rebuild `MEMORY.md` from the surviving files: one line per memory in the defined index format, no other content.

The step is done when every file in the directory has exactly one line in `MEMORY.md` and every line in `MEMORY.md` resolves to a file.

## 6. Report

Counts per verdict, the promotions made (with PR links), the questions still open, and the backup path.
