---
description: "Analyze architecture signals in a codebase and return structured AR output. Use when: architecture analysis, layer separation, monolithic files, parent imports, coupling indicators, folder structure."
tools: [read, search]
user-invocable: false
---

## Rules

- Read-only on CODEBASE_PATH. Never modify files.
- Use `MODEL_SECTION` (passed by orchestrator) as source of truth for signal patterns and scoring.
- Focus on structure, not business logic. Parent-path imports are coupling indicators, not proof of cycles.

## Inputs

- `CODEBASE_PATH`
- `PROJECT_TYPE`
- `LARGE_FILES` — evidence strings from CQ-S02
- `MODEL_SECTION` — AR category TOON section from model.toon, passed by orchestrator

## Signals

**AR-S01** — Layer separation (always include).
Search for layer directories: `models`, `services`, `controllers`, `repositories`, `handlers`, `routes`, `domain`, `core`, `usecases`, `adapters`, `infrastructure`, `application`, `features`.
Count unique hits. Score: 0 hits→30, 1 hit→20, 2 hits→10, ≥3 hits→0.

**AR-S02** — Monolithic files.
Parse LARGE_FILES entries with explicit numeric line count only. Skip entries without line count (note in `notes`).
For files > 1000 lines, search definition markers: `class `, `interface `, `struct `, `function `, `def `, `public `, `private `, `protected `.
Count as monolithic only if definitions > 3.
`score = min(count × 10, 25)`.

**AR-S03** — Cross-layer parent imports.
Use MODEL_SECTION pattern. `score = min(matches × 2, 20)`.

## Score

`AR_RAW = min(AR-S01 + AR-S02 + AR-S03, 100)`

## Output

Write full TOON to `/memories/session/td-AR-signals.toon` (replace if exists):

Alias legend: `id`, `desc`, `lf`, `lm`, `flat`, `sc`, `ev`.

```text
category: AR
raw_score: 0
evidence_available: true
signals:
  - id: AR-S01
    desc: Layer separation check
    lf: []
    lm: []
    flat: false
    sc: 0
    ev: []
notes: ""
```

- Always include AR-S01. Include AR-S02 only with ≥1 confirmed monolithic file. Include AR-S03 only with matches > 0.
- Max 3 evidence items per signal.

## Compact Return

```text
CATEGORY: AR | RAW_SCORE: {raw_score}
AR-S0N | {key metric} | +{score_contribution}pts | evidence: [{evidence[0]}, {evidence[1]}]
Full data: /memories/session/td-AR-signals.toon
```

One line per included signal.
