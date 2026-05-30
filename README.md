# project-status

A Claude Code skill that renders a project's progress as an ASCII terminal
dashboard when you reopen it after a gap — title box, progress bars, a
session-handoff timeline (prev / wip / next), phase checklist, recent activity,
and warnings.

## What it does

When you run `/project-status [project-name]` (or ask "어디까지 했지?",
"show progress on X"), it reconstructs project state from three read-only sources
and prints a single-screen dashboard:

1. **Memory files** — `~/.claude/projects/<slug>/memory/MEMORY.md` index + matching
   `project_*.md`. The body is the primary signal; a stale index is flagged.
2. **Project-folder status files** — rating/output dirs, progress logs, `STATUS`/
   `CHANGELOG`, version manifests.
3. **Git history** — branch, last commits, uncommitted state (skipped if not a repo).

If no project is named, it lists candidates from the memory index and asks which.

## Example

```
╭─ dharness ─────────────────────────────── v0.9.0 ─╮
│ 회귀 평가 페이즈 — ko01~85 평가 중, plugin 본체 fix 완료 │
│                                                    │
│ 평가     ███████████░░░░░░░░░  ~59%  (50/85 ko)    │
│ 패치     ████████████████████  100%  (9/9 적용)    │
╰────────────────────────────────────────────────────╯

 ▸ HANDOFF
   prev   ✓ CLAUDE.md doctrine 개편 + 정합 sweep (82511fe, v0.9.0)
   wip    ◔ 미커밋 — batchN 후속·eval-checklist.md·dharness-rating/
   next   → ko51~60 evaluator dispatch

 ▸ PHASES
   ✓ ko01-50 평가   ◔ ko51-60   ○ ko61-85   ○ ko86-100 산출물
 ...
```

## Install

Copy the `project-status/` directory into `~/.claude/skills/`.

## Design notes

The gather step is deliberately budgeted ("a glance, not an audit"): one repo, one
summary per track, glob for chunk existence rather than reading every file. In
testing this cut token cost ~78% (301k → 67k) versus an unbounded walk, with no loss
of dashboard quality. See `evals/` and the workspace benchmark for the measurements.
