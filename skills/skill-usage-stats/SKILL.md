---
name: skill-usage-stats
description: Generate an HTML dashboard of skill usage across Claude Code and Codex (global + project .claude/skills, ~/.claude/plugins, ~/.codex/skills, ~/.codex/prompts). Triggers — "skill usage stats", "which skills do I actually use", "audit my skills", "스킬 사용 통계".
---

# Skill Usage Stats

Scans local Claude Code (`~/.claude/projects/**/*.jsonl`) and Codex (`~/.codex/sessions/**/*.jsonl`) transcripts, joins against the installed skill/prompt catalog, and writes an interactive HTML dashboard to `~/.claude/skill-stats.html`.

## Files in this skill

- `analyze.py` — scans transcripts, writes the HTML dashboard, opens browser.
- `summary.py` — terminal-only views (top/zero/stale/overview) from the same HTML.

## How to run

Generate the HTML dashboard (default: scan all transcripts, write to `~/.claude/skill-stats.html`, open browser):

```bash
python3 ~/.claude/skills/skill-usage-stats/analyze.py
```

`analyze.py` flags:
- `--days N` — only count invocations in the last N days (default: all time).
- `--project <substr>` — restrict to transcripts whose cwd matches.
- `--out <path>` — output HTML path (default: `~/.claude/skill-stats.html`).
- `--no-open` — generate but don't open the browser.

Terminal views (after the HTML exists):

```bash
python3 ~/.claude/skills/skill-usage-stats/summary.py overview         # per-source totals
python3 ~/.claude/skills/skill-usage-stats/summary.py top -n 20        # top 20 most-used
python3 ~/.claude/skills/skill-usage-stats/summary.py zero             # 0-use skills (cleanup list)
python3 ~/.claude/skills/skill-usage-stats/summary.py stale --days 90  # >90d unused or never
```

`summary.py` accepts `--source claude-global|claude-project|codex-global-skill|codex-global-prompt` and `--html <path>`.

## Detection rules

Counts explicit calls and observed load-like path references in the supported transcript formats. File reads/edits can be inspection rather than invocation; missing events, retention gaps, or unsupported harness formats can undercount. Treat counts as observed signals, not universal usage telemetry.

- **Claude Code — explicit `Skill` tool**: `tool_use` with `name="Skill"`, `input.skill=<name>`.
- **Claude Code — slash command**: `<command-name>/name</command-name>` tag in user messages.
- **Claude Code — auto-trigger via `Read`**: `tool_use` with `name="Read"`, `file_path` ending in `/skills/<name>/SKILL.md` (this is how `using-superpowers`, `using-symphony`, etc. actually get activated).
- **Claude Code — auto-trigger via `Bash`**: `tool_use` with `name="Bash"`, command containing `/skills/<name>/SKILL.md` (e.g. `cat`, `sed`).
- **Codex — slash command**: user `input_text` whose first non-blank line is `/<name>`.
- **Codex — auto-trigger via shell**: `function_call` events with `name` in `{exec_command, shell, apply_patch}` whose arguments contain `/skills/<name>/SKILL.md` (Codex reads skills via shell, not a Read tool).

Each `tool_use` / `function_call` event credits the skill once. Source is decided by the path containing `.claude` vs `.codex`.

**Catalog sources** (where the SKILL.md lives on disk):
- `claude-global`: `~/.claude/skills/*/SKILL.md`
- `claude-project`: `<repo>/.claude/skills/*/SKILL.md` (discovered from transcript cwds)
- `claude-plugin`: `~/.claude/plugins/**/skills/*/SKILL.md` (marketplaces + cache)
- `codex-global-skill`: `~/.codex/skills/*/SKILL.md`
- `codex-global-prompt`: `~/.codex/prompts/*.md`

## Dashboard

Columns: `name | verdict | sources | count | sessions | last used | days since | size (KB) | description`. Sortable; filter by source, verdict, zero-use, or stale. Zero-use and >90d rows are highlighted.

`verdict` carries a heuristic reason for review, not authorization to archive:
- `KEEP` — used within 90 days, or never used but installed ≤30 days ago.
- `REVIEW` — never used and installed 31–60 days ago, or last used 91–180 days ago.
- `DELETE` — never used and installed >60 days ago, or last used >180 days ago.
