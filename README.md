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

Run against `stockdb` (a KR/US OHLCV + macro time-series warehouse on Neon
Postgres). It has no user-memory file, so state is reconstructed from the project
folder's changelog/telemetry alone — note the `git(n/a)` / `user-memory(none)` in
the `src:` line:

```
╭─ stockdb ───────────── time-series DW · Neon PG ─╮
│ KR/US OHLCV + macro warehouse on Neon Postgres   │
│ read-side: ext viz app + BI (backtest/ML def)    │
│                                                  │
│ pipeline   ████████████████████  ~95%  live      │
│ asset-cov  ███████████████░░░░░  ~75%  KR full   │
╰──────────────────────────────────────────────────╯

 ▸ HANDOFF
   prev   ✓ retention 3yr→1yr pivot (Neon 512MB hit) → promote
            668,743 rows, DB 113MB/512MB
   wip    ◔ clean close, 3 deferred items carried
   next   → calendar_gap re-run on 2770 KR eq / 1yr window
            + fix 2 test_pykrx defects

 ▸ PHASES
   ✓ schema   ✓ ingestion   ✓ quality   ✓ serving   ✓ daily-auto

 ▸ RESULTS
   curated 668,743 rows · pykrx 2770 KR sym · KRX 123 idx live
   DB 113MB/512MB (22%) · pytest 5P+1S · 16 migrations · 11 adapters

 ▸ RECENT (telemetry + changelog)
   • 05-28  retention pivot + promote 658,925 + VACUUM→113MB
   • 05-28  KRX OpenAPI key issued → 123 idx live (29,417 rows)
   • 05-27  KR full universe 2770 sym, 3yr backfill (42m)

 ⚠ deferred: calendar_gap re-run · 2 test_pykrx defects · FRED/ECOS keys
 ⚠ no git repo — all work untracked despite Trunk-based rule

 src: changelog.md + _workspace(telemetry ×4d) + CLAUDE.md
      + git(n/a) + user-memory(none for stockdb)
```

## Language

Labels render in English or Korean. The skill picks the language from, in order:
`--lang ko` / `--lang en` flag → the language of your request → English default.
So `/project-status stockdb --lang ko` or simply asking in Korean
("stockdb 현황 보여줘") yields Korean chrome. Only the fixed labels are translated;
the data (commit subjects, notes) stays in its source language.

Same stockdb state, Korean labels:

```
╭─ stockdb ───────────── time-series DW · Neon PG ─╮
│ KR/US OHLCV + 매크로 웨어하우스 (Neon Postgres)  │
│ 읽기측: 외부 시각화 앱 + BI (백테스트/ML 예정)   │
│                                                  │
│ 파이프라인 ████████████████████  ~95%  운영중    │
│ 자산범위   ███████████████░░░░░  ~75%  KR 전종목  │
╰──────────────────────────────────────────────────╯

 ▸ 인계                             (작업 재개 지점)
   이전   ✓ retention 3yr→1yr 전환 (Neon 512MB 도달) →
            promote 668,743행, DB 113MB/512MB
   진행   ◔ 정상 종료, 보류 3건 이월
   다음   → calendar_gap 재실행 (2770 KR eq / 1년 윈도우)
            + test_pykrx 결함 2건 수정

 ▸ 단계
   ✓ 스키마   ✓ 적재   ✓ 품질   ✓ 서빙   ✓ 일일자동화

 ▸ 결과
   curated 668,743행 · pykrx 2770 KR 종목 · KRX 123 지수 운영
   DB 113MB/512MB (22%) · pytest 5P+1S · 마이그레이션 16 · 어댑터 11

 ▸ 최근 (telemetry + changelog)
   • 05-28  retention 전환 + promote 658,925 + VACUUM→113MB
   • 05-28  KRX OpenAPI 키 발급 → 123 지수 운영 (29,417행)
   • 05-27  KR 전체 유니버스 2770 종목, 3년 백필 (42분)

 ⚠ 보류: calendar_gap 재실행 · test_pykrx 결함 2건 · FRED/ECOS 키
 ⚠ git repo 없음 — Trunk-based 규칙에도 전 작업 미추적

 출처: changelog.md + _workspace(telemetry ×4일) + CLAUDE.md
      + git(없음) + 메모리(stockdb 없음)
```

## Install

This repo is a Claude Code plugin marketplace. In Claude Code:

```
/plugin marketplace add gimangdoo/project-progress
/plugin install project-status@gimangdoo
```

Then run `/project-status` (or ask "어디까지 했지?", "show progress on X").

Manual alternative: copy `skills/project-status/` into `~/.claude/skills/`.

## Design notes

The gather step is deliberately budgeted ("a glance, not an audit"): one repo, one
summary per track, glob for chunk existence rather than reading every file. In
testing this cut token cost ~60% (301k → 120k) versus an unbounded walk, with no
loss of dashboard quality.
