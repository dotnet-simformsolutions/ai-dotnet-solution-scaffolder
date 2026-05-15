---
description: "Analyze test-coverage signals in a codebase and return structured TC output. Use when: test coverage analysis, missing tests, coverage report parsing, low test ratio, no test directories."
tools: [read, search]
user-invocable: false
---

## Rules

- Read-only on CODEBASE_PATH. Never modify files.
- Use `MODEL_SECTION` (passed by orchestrator) as source of truth for signal patterns and scoring.
- Use exactly one scoring path: `no_tests`, `coverage_report`, or `ratio_fallback`.

## Inputs

- `CODEBASE_PATH`
- `PROJECT_TYPE`
- `TOTAL_SOURCE_FILE_COUNT`
- `MODEL_SECTION` — TC category TOON section from model.toon, passed by orchestrator

## Signals

**TC-S01** — Test directory presence.
Search: `**/tests/**`, `**/__tests__/**`, `**/test/**`, `**/spec/**`.
`found = true` if any match.

**TC-S02** — Coverage report.
Search in precedence: `coverage-summary.json` → `coverage-report.json` → `coverage.xml` → `lcov.info`. Parse first match only. `coverage_percentage` must be 0–100.

**TC-S03** — Test-to-source ratio (fallback).
Only when no coverage report found. Search test files using MODEL_SECTION patterns.
`ratio = test_file_count / TOTAL_SOURCE_FILE_COUNT` (0 if denominator is 0).

## Scoring Path

| Path | Condition | Formula | Signals |
|---|---|---|---|
| `no_tests` | No test dirs AND test_file_count = 0 | `TC_RAW = 80` | TC-S01 (score=80) |
| `coverage_report` | Coverage report exists | `TC_RAW = min(max(0, 100 - coverage_pct), 100)` | TC-S01 (score=0), TC-S02 (score=TC_RAW) |
| `ratio_fallback` | Test dirs exist, no coverage report | `TC_RAW = min(max(0, 60 - ratio×120), 100)` | TC-S01 (score=0), TC-S03 (score=TC_RAW) |

## Output

Write full TOON to `/memories/session/td-TC-signals.toon` (replace if exists):

Alias legend: `id`, `desc`, `fd`, `sc`, `ev`.

```text
category: TC
raw_score: 0
evidence_available: true
scoring_path: no_tests
signals:
  - id: TC-S01
    desc: Test directory presence
    fd: false
    sc: 80
    ev: []
notes: ""
```

- Always include TC-S01. Include TC-S02 only in `coverage_report` path. Include TC-S03 only in `ratio_fallback` path.
- Max 3 evidence items per signal.

## Compact Return

```text
CATEGORY: TC | RAW_SCORE: {raw_score} | scoring_path: {path}
TC-S0N | {key metric} | +{score_contribution}pts | evidence: [{evidence[0]}]
Full data: /memories/session/td-TC-signals.toon
```

One line per included signal.
