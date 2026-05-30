---
name: project-status
description: Show a project's progress as an ASCII terminal dashboard (title box + progress bars + phase checklist + session handoff timeline + recent activity + next actions) at the start of a session. Renders labels in English or Korean (--lang ko/en, else matches the prompt language). Gathers state from three sources — the user's memory files, the project folder's own status files, and git history. Use when the user runs /project-status, asks "where is project X", "show progress on X", "현황 보여줘", "어디까지 했지", "이어서 하자", or wants a one-glance status of ongoing work when picking a project back up.
---

# project-status

Render a one-glance progress dashboard for ONE project in the terminal. The point
is to rebuild context fast when reopening a project after a gap — so the timeline
of "what I was doing last session" matters as much as the raw progress numbers.

## 0. Resolve which project

- If the user passed a name as argument → use it.
- If not → read the memory index `~/.claude/projects/<slug>/memory/MEMORY.md`,
  list the projects it mentions (one line each), and ask which one. Stop until answered.

On Windows the memory dir is e.g.
`C:\Users\user01\.claude\projects\C--Users-user01-awesome-files\memory\`.

## 0.5 Resolve the output language

The dashboard's fixed labels (section headers, `prev/wip/next`, the src line) can
render in English or Korean. Pick the language like this, in priority order:

1. **Explicit flag** — `--lang ko` / `--lang en` (or `ko` / `en` / `한글` / `english`
   as a trailing word) in the user's command → obey it.
2. **Prompt language** — otherwise match the language the user asked in. A Korean
   request ("현황 보여줘", "어디까지 했지") → Korean labels. An English request →
   English labels.
3. **Default** — English.

Only the fixed chrome is translated. The *content* (commit subjects, memory notes,
phase names) stays in whatever language the source wrote it — don't translate the
data, just the labels. Use this mapping:

| element | English | 한글 |
|---|---|---|
| handoff header | `▸ HANDOFF` | `▸ 인계` |
| phases header | `▸ PHASES` | `▸ 단계` |
| results header | `▸ RESULTS` | `▸ 결과` |
| recent header | `▸ RECENT` | `▸ 최근` |
| prev / wip / next | `prev` / `wip` / `next` | `이전` / `진행` / `다음` |
| status line one-liner hint | `(where I left off)` | `(작업 재개 지점)` |
| provenance prefix | `src:` | `출처:` |
| not-a-repo marker | `git(n/a)` | `git(없음)` |
| no-memory marker | `user-memory(none)` | `메모리(없음)` |

Keep the glyphs (`✓ ◔ ○ → █ ░ ⚠ ▸`) the same in both languages.

## 1. Gather from THREE sources (read-only)

Collect what each source offers; skip silently any that don't apply.

1. **Memory files** — `MEMORY.md` (index) + every `project_*.md` whose subject matches.
   Read frontmatter `description` and the body. These hold phase status, version,
   commits, and "미수행/미결" (pending) notes. **The body is primary**: when the index
   line and the body disagree (e.g. version), trust the body and flag the index as stale.
2. **Project-folder status files** — inside the project dir, look for the
   things the memory points at: rating/output dirs (e.g. `dharness-rating/`),
   progress logs, `STATUS`/`PROGRESS`/`TODO` files, `CHANGELOG`, version in
   `plugin.json`/`package.json`. Read only files the memory or obvious names point to.
3. **Git history** — only if the project dir is a git repo. Get current branch,
   last ~5 commits (`git log --oneline -5`), and uncommitted/untracked state
   (`git status --short`). If not a repo, skip — say "git: n/a".

**Budget the gather — this is a glance, not an audit.** Reading everything is the
main failure mode here: a status view that takes 80 tool calls and reads every
result file defeats its own purpose. Hold yourself to roughly:
- Memory: the index + matching `project_*.md` only.
- Status files: one summary per track (e.g. each chunk's `summary.md`, not every
  per-item file inside it), plus the version manifest. Glob to confirm a chunk is
  *done* (files exist) without reading each file's contents.
- Git: the ONE repo the memory points at. Don't go discovering and walking sibling
  repos unless the memory says the project spans several. `git log --oneline -5` +
  `git status --short`, nothing deeper.

If the memory body already states the tallies/version/next-step, trust it and use
the files only to spot-check and to find what the memory doesn't have (recent
commits, uncommitted state). Do NOT enumerate the whole tree.

## 2. Derive the numbers and the timeline

- **Overall %** = estimate from the phase checklist (done / total). Label it `~`
  (estimate). If memory states an explicit completion, use that instead. Multiple
  independent tracks → one bar each (e.g. one for eval progress, one for patches).
- **Phase marks**: `✓` done · `◔` in progress · `○` not started.
- **Session handoff (CHANGELOG)** — the most important reconstruction signal.
  From the memory body + git, infer three things:
  - `PREV` — what the *last* session finished (the most recent committed/done work).
  - `WIP` — what was in progress / left mid-flight when the last session stopped.
    Look for "중단", "후속 박제", "미커밋", "다음 =", dangling tasks. If nothing was
    mid-flight, say `WIP — (없음, 깔끔히 종료)`.
  - `NEXT` — the next intended action.
  This PREV → WIP → NEXT chain is what lets the user resume without re-reading everything.
- **Warnings**: uncommitted work, stale baseline/index, blocked items ("미커밋", "보류", "미결").

## 3. Render the dashboard

Fixed outer box width = 54 chars (the `╭…╮` / `╰…╯` borders). Keep every `│ … │`
row padded to that width so the right border lines up — misaligned boxes read as
broken. Progress bar = 20 cells, `█` filled / `░` empty. Round to nearest cell.

Use this shape (fill real data; sections in fixed order top→bottom). Labels shown
in English here — swap to the §0.5 한글 column when the resolved language is Korean:

```
╭─ <name> ──────────────────────────── <version> ─╮
│ <one-line status from memory>                    │
│                                                  │
│ <track1>  ███████████░░░░░░░░░  ~60%  (50/85)    │
│ <track2>  ████████████████████  100%  (9/9)      │
╰──────────────────────────────────────────────────╯

 ▸ HANDOFF                          (where I left off)
   prev   ✓ <last finished work + commit>
   wip    ◔ <left mid-flight, or "없음">
   next   → <next intended action>

 ▸ PHASES
   ✓ <done>   ✓ <done>   ◔ <in progress>   ○ <pending>

 ▸ RESULTS                          (omit if project has no metrics)
   <key tallies on one or two lines>

 ▸ RECENT
   • <commit hash + subject, or memory note>
   • <…>

 ⚠ <warning line>                   (one per warning; omit if none)

 src: memory(<file>) + <status dir> + git(<state>)
```

Korean rendering of the same shape (labels translated, data left as-is):

```
╭─ <name> ──────────────────────────── <version> ─╮
│ <메모리 한 줄 상태>                              │
│                                                  │
│ <트랙1>   ███████████░░░░░░░░░  ~60%  (50/85)    │
│ <트랙2>   ████████████████████  100%  (9/9)      │
╰──────────────────────────────────────────────────╯

 ▸ 인계                             (작업 재개 지점)
   이전   ✓ <마지막 완료 작업 + 커밋>
   진행   ◔ <중단된 작업, 없으면 "없음">
   다음   → <다음 작업>

 ▸ 단계
   ✓ <완료>   ✓ <완료>   ◔ <진행중>   ○ <대기>

 ▸ 결과
   <핵심 집계 1~2줄>

 ▸ 최근
   • <커밋 해시 + 제목, 또는 메모리 노트>

 ⚠ <경고 줄>

 출처: memory(<파일>) + <상태 디렉토리> + git(<상태>)
```

Note the label widths differ between languages (`prev`=4 vs `이전`=2 chars wide on
screen). Re-pad the `이전/진행/다음` column so the `✓/◔/→` glyphs still align.

Formatting rules that keep it scannable:
- Section headers use `▸ NAME` so they stand out from the indented body rows.
- The parenthetical hints (`where I left off`, etc.) are optional — drop them once
  the shape is familiar, or keep for a first-time reader. Use judgment.
- `HANDOFF` is the headline section — always render it, right after the box.
- Align the `prev/wip/next` labels and the `✓/◔/→` glyphs in columns.
- One project per dashboard. No multi-project compare.
- If a section has no data, drop the whole section (no empty headers). HANDOFF and
  the `src:` line are the only always-present parts.
- No prose paragraphs around the box — the dashboard IS the answer.
- ANSI color is fine if the terminal supports it (green ✓ / yellow ◔ / dim ○),
  but never at the cost of alignment, and the plain glyphs must carry the meaning
  on their own in case color is stripped.

## 4. Offer next step

End with a single suggested action drawn from `next` —
e.g. "이어서 ko51~60 평가 dispatch 할까?". One line. Don't start work unprompted.
