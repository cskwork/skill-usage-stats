---
name: skill-cleanup
description: Archive (not delete) Claude Code and Codex skills that are zero-use or stale, based on a generated skill-stats.html report, keeping every file recoverable. Triggers — "clean up skills", "archive unused skills", "스킬 정리", "안 쓰는 스킬 정리".
---

# Skill Cleanup

Companion to `skill-usage-stats`. Reads the same HTML report and **archives** skills marked DELETE (or matching user-chosen criteria) without deleting any files.

## How archiving works

For each chosen skill directory:

1. Move it under `~/.claude/skills-archive/<source>/<skill-name>/`  (or `~/.codex/skills-archive/...`)
2. Inside the archived directory, rename `SKILL.md` → `SKILL.md.archived` so the harness no longer treats it as an active skill (the harness only auto-discovers `SKILL.md`).
3. Write a `.archived-meta.json` next to it recording original path, archive date, and the reason from the stats report.

Single-file prompts (`~/.codex/prompts/<name>.md`) get a wrapper directory instead: `~/.codex/skills-archive/codex-global-prompt/<name>/<name>.md.archived` plus the same `.archived-meta.json`.

Reverse: `restore <name>` moves it back, renaming `SKILL.md.archived` → `SKILL.md` (or `<name>.md.archived` → `<name>.md` for a prompt).

## How to run

```bash
# 1. Generate fresh stats first
python3 ~/.claude/skills/skill-usage-stats/analyze.py --no-open

# 2. Dry-run (default) — print what would be archived
python3 ~/.claude/skills/skill-cleanup/archive.py

# 3. Apply
python3 ~/.claude/skills/skill-cleanup/archive.py --apply

# Useful flags
#   --verdict DELETE       default; DELETE = never used and installed >60d, or last used >180d
#   --verdict DELETE,REVIEW
#   --days 90              also archive count>0 but stale >N days
#   --source claude-global only archive from one source

# Restore
python3 ~/.claude/skills/skill-cleanup/archive.py restore <name>
python3 ~/.claude/skills/skill-cleanup/archive.py list-archive
```

## Safety

- Dry-run by default. A broad cleanup request produces a review; apply only the user-authorized selection. Recorded zero use is not proof of universal non-use.
- Archives preserve contents and metadata; check restore destinations for conflicts before claiming reversibility.
- Skips any skill not in the stats report (so you can't accidentally archive an unscanned skill).
- Skips symlinked skill directories, reporting each as `symlink (not archiving)` — a symlinked catalog stays untouched.
- Refuses to operate on plugin cache paths (`~/.claude/plugins/cache/...`) — those are managed by the plugin marketplace and a rename there would just be re-installed.
