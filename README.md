# skill-usage-stats

Two cross-platform (macOS / Linux / Windows) skills that answer two simple questions about your Claude Code and Codex installations:

1. **Which skills do I actually use?**
2. **Which ones can I safely clean up?**

Both work entirely offline against your local transcript history — no telemetry, no network calls.

---

## Why this exists

After installing dozens of marketplace skills, plugins, and personal scripts, the catalog gets noisy fast. Auto-triggered skills load through `Read` or shell access to `SKILL.md` (not always through the explicit `Skill` tool), so a simple grep undercounts. This pair:

- Scans every Claude Code transcript (`~/.claude/projects/**/*.jsonl`) and Codex session (`~/.codex/sessions/**/*.jsonl`).
- Reconciles invocations against the on-disk skill catalog: global, project, plugin (marketplaces + cache), Codex skills, Codex prompts.
- Renders an interactive HTML dashboard you can sort, filter, and export from.
- Provides a reversible archive workflow — never deletes, just renames `SKILL.md` so the harness stops surfacing it.

---

## What gets counted

| Source | Pattern detected |
|---|---|
| Claude Code explicit | `tool_use` with `name="Skill"`, `input.skill=<name>` |
| Claude Code slash command | `<command-name>/name</command-name>` in user messages only |
| Claude Code auto-trigger | `Read` of `*/skills/<name>/SKILL.md` |
| Claude Code shell auto-trigger | `Bash` command containing `*/skills/<name>/SKILL.md` |
| Codex slash command | User `input_text` whose first line is `/name` |
| Codex auto-trigger | `function_call(exec_command/shell)` arguments containing `*/skills/<name>/SKILL.md` |

Each event credits the skill once. Source is decided from the `SKILL.md` path containing `.claude` vs `.codex`. Same-named skills across global / project / plugin are merged into one row with summed count and a list of sources.

---

## Install

```bash
git clone https://github.com/cskwork/skill-usage-stats.git
cd skill-usage-stats

# macOS / Linux
cp -R skills/skill-usage-stats ~/.claude/skills/
cp -R skills/skill-cleanup     ~/.claude/skills/

# Windows (PowerShell)
Copy-Item -Recurse skills/skill-usage-stats $HOME\.claude\skills\
Copy-Item -Recurse skills/skill-cleanup     $HOME\.claude\skills\
```

Both skills auto-trigger from natural-language phrases (Korean / English), or you can invoke them directly via `python3` (see below).

---

## skill-usage-stats — usage

```bash
# Generate the HTML dashboard (default: scan all transcripts, open in browser)
python3 ~/.claude/skills/skill-usage-stats/analyze.py

# Flags
#   --days N          only count invocations in the last N days
#   --project <substr> restrict to transcripts whose cwd matches
#   --out <path>      output HTML path (default: ~/.claude/skill-stats.html)
#   --no-open         generate but don't open the browser

# Terminal-only views (after the HTML exists)
python3 ~/.claude/skills/skill-usage-stats/summary.py overview
python3 ~/.claude/skills/skill-usage-stats/summary.py top -n 20
python3 ~/.claude/skills/skill-usage-stats/summary.py zero
python3 ~/.claude/skills/skill-usage-stats/summary.py stale --days 90
```

Dashboard columns: `name · verdict · sources · count · sessions · last used · days since · size (KB) · description`.

The `verdict` column auto-classifies each skill as **KEEP / REVIEW / DELETE** with a one-line reason:

| Verdict | Rule |
|---|---|
| `KEEP` | count > 0 and last used ≤ 90 days, **or** never used but installed ≤ 30 days (recent install, likely intentional) |
| `REVIEW` | never used but installed 31–60 days ago, or used long ago (≤ 180 days) |
| `DELETE` | never used and installed > 60 days, or last used > 180 days |

Use the filters at the top of the dashboard (only KEEP / REVIEW / DELETE, only zero-use, only stale) to drill into a decision view.

---

## skill-cleanup — usage

```bash
# Always dry-run by default
python3 ~/.claude/skills/skill-cleanup/archive.py

# Apply (move skills under ~/.claude/skills-archive/<source>/<name>/)
python3 ~/.claude/skills/skill-cleanup/archive.py --apply

# Wider criteria
python3 ~/.claude/skills/skill-cleanup/archive.py --verdict DELETE,REVIEW --days 90 --apply

# Restrict to one source
python3 ~/.claude/skills/skill-cleanup/archive.py --source claude-project --apply

# Reversible
python3 ~/.claude/skills/skill-cleanup/archive.py list-archive
python3 ~/.claude/skills/skill-cleanup/archive.py restore <skill-name>
```

### Archive guarantees

- **Never deletes**. Skills are moved under `~/.claude/skills-archive/<source>/<name>/` (or `~/.codex/skills-archive/...`).
- **Renames `SKILL.md` → `SKILL.md.archived`** inside the archived directory. Since the harness only auto-discovers `SKILL.md`, the agent loses access to that skill, but every file is recoverable.
- Writes `.archived-meta.json` next to each archived skill with the original path, archive date, and stats-report reason.
- Refuses to operate on `~/.claude/plugins/cache/...` — those are managed by the marketplace and would be re-installed.
- Restore is a single command; the dry-run mode lets you preview every move before committing.

---

## Cross-platform notes

- Tested on macOS. Pure stdlib Python 3.10+ — should work on Linux and Windows.
- Path-comparison logic normalizes backslashes to forward slashes, so Windows-style logged paths in transcripts match correctly.
- HTML is opened via `Path.as_uri()`, which produces a valid file URI on all platforms.

---

## License

MIT. See [LICENSE](./LICENSE).
