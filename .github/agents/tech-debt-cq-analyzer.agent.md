---
description: "Analyze code-quality signals in a codebase and return structured CQ output. Use when: code quality analysis, TODOs, large files, magic numbers, large functions, deep nesting."
tools: [read, search]
user-invocable: false
---

## Rules

- Read-only on CODEBASE_PATH. Never modify files or surface secrets.
- Use `MODEL_SECTION` (passed by orchestrator) as source of truth for signal patterns and scoring.
- When a heuristic is unclear, do not count it.

## Inputs

- `CODEBASE_PATH`
- `PROJECT_TYPE`
- `SOURCE_GLOB` — file glob for this project type (passed by orchestrator)
- `TOTAL_SOURCE_FILE_COUNT`
- `MODEL_SECTION` — CQ category TOON section from model.toon, passed by orchestrator

## Exclusions

Exclude paths: `.git`, `node_modules`, `vendor`, `dist`, `build`, `coverage`, `tech-debt-reports`, `bin`, `obj`.
Exclude patterns: `**/migrations/**`, `**/Migrations/**`, `*DbContextModelSnapshot*`, `*.g.cs`, `*.generated.*`, `*.designer.*`, `*.Designer.*`, `*.min.{js,css}`, `*.resx`, `*.xlf`.

## Signals

**CQ-S01** — Deferred-work comments.
Search MODEL_SECTION pattern `TODO|FIXME|HACK|XXX` in SOURCE_GLOB files.
`score = min(matches × 0.5, 20)`.

**CQ-S02** — Files > 500 lines.
Build candidates from SOURCE_GLOB after exclusions.
- ≤ 200 source files: inspect all.
- \> 200: inspect only files under `src/`, `app/`, `lib/`, `services/`, `controllers/`, `handlers/`, `domain/`, `core/`, `features/`.
Read lines 1–650 per file. Line 501 exists → large. Evidence: `path (~N lines)` or `path (>650 lines)`.
`score = min(count × 3, 15)`.

**CQ-S03** — Magic numbers in business logic.
Use MODEL_SECTION pattern and `exclude_file_patterns`. Also exclude test files: `*.test.*`, `*.spec.*`, `*_test.*`, `*Test.*`, `*Tests.*`.
`score = min(matches × 0.3, 10)`.

**CQ-S04** — Large functions (>50 lines).
Only inspect CQ-S02 flagged files. Search markers: `def `, `function `, `func `, `public `, `private `, `protected `. Count only functions whose visible body clearly exceeds 50 lines. Skip unclear spans.
`score = min(count × 2, 15)`.

**CQ-S05** — Deep nesting (4+ levels).
Use MODEL_SECTION pattern. `score = min(matches × 1, 10)`.

## Score

`CQ_RAW = min(S01 + S02 + S03 + S04 + S05, 100)`

## Output

Write full TOON to `/memories/session/td-CQ-signals.toon` (replace if exists):

Alias legend: `id`, `desc`, `mc`, `sc`, `ev`.

```text
category: CQ
raw_score: 0
evidence_available: true
signals:
  - id: CQ-S01
    desc: "..."
    mc: 0
    sc: 0
    ev: []
notes: ""
```

- Signal order: S01→S05. Include only signals with `match_count > 0`. Max 3 evidence items per signal.

## Compact Return

```text
CATEGORY: CQ | RAW_SCORE: {raw_score}
CQ-S0N | {match_count} matches | +{score_contribution}pts | evidence: [{evidence[0]}, {evidence[1]}]
Full data: /memories/session/td-CQ-signals.toon
```

One line per included signal. Do not return full JSON in chat.
