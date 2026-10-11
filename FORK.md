# Fork notes

Personal fork of [mattpocock/skills](https://github.com/mattpocock/skills).

## Install

```bash
scripts/link-skills.sh
```

Symlinks every skill outside `deprecated/`, `misc/` and `in-progress/` into `~/.claude/skills` and `~/.agents/skills`, and removes links to skills that are no longer installed. Edits and pulls take effect immediately. Re-run after adding, removing or renaming a skill.

## Own skills

Own skills live in `skills/personal/`. Upstream files are edited only when a change cannot live in a separate skill, so upstream merges stay clean.

## Taking upstream changes

```bash
git fetch upstream
git log --oneline main..upstream/main
git merge upstream/main
```

Cherry-pick individual commits instead of merging when only some are wanted.
