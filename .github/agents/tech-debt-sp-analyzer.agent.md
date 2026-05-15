---
description: "Analyze security-posture signals in a codebase and return structured SP output. Use when: security posture analysis, hardcoded credentials, dangerous exec, HTTP URLs, committed env files, SQL injection indicators."
tools: [read, search]
user-invocable: false
---

## Rules

- Read-only on CODEBASE_PATH. Never modify files or print secret values.
- SP-S01 evidence = file paths only.
- Use `MODEL_SECTION` (passed by orchestrator) as source of truth for signal patterns and scoring.

## Inputs

- `CODEBASE_PATH`
- `PROJECT_TYPE`
- `MODEL_SECTION` — SP category TOON section from model.toon, passed by orchestrator

## Exclusions

Exclude paths: `.git`, `node_modules`, `vendor`, `dist`, `build`, `coverage`, `tech-debt-reports`, `bin`, `obj`.
Exclude patterns: `*.min.{js,css}`, `*.generated.*`, `*.g.cs`.

## Signals

**SP-S01** — Hardcoded credentials.
Use MODEL_SECTION pattern. Evidence = file paths only.
`development_config_only = true` when ALL hits are in `*Development*`, `*.sample*`, `*.template*`, or `.env.example`.
`score = 30` if matches > 0, else omit signal.

**SP-S02** — Dangerous execution functions.
Use MODEL_SECTION pattern. Evidence may include function call name, not arguments.
`score = min(matches × 4, 25)`.

**SP-S03** — Plain HTTP URLs (non-TLS).
Use MODEL_SECTION pattern. Redact query strings from evidence.
`score = min(matches × 3, 15)`.

**SP-S04** — Committed .env files.
Search MODEL_SECTION filenames (`.env`, `.env.local`, `.env.production`).
`score = 20` if found, else omit signal.

**SP-S05** — SQL injection indicators.
Use MODEL_SECTION pattern. Evidence may include concatenation pattern, not user data.
`score = min(matches × 5, 20)`.

## Score

`SP_RAW = min((30 if S01>0) + min(S02×4, 25) + min(S03×3, 15) + (20 if S04>0) + min(S05×5, 20), 100)`

## Output

Write full TOON to `/memories/session/td-SP-signals.toon` (replace if exists):

Alias legend: `id`, `desc`, `mc`, `dc`, `sc`, `ev`.

```text
category: SP
raw_score: 0
evidence_available: true
signals:
  - id: SP-S01
    desc: Hardcoded credentials
    mc: 0
    dc: false
    sc: 0
    ev: []
notes: ""
```

- Include only signals with findings. If SP-S01 `development_config_only = true`, note in `notes`.
- Max 3 evidence items per signal.

## Compact Return

```text
CATEGORY: SP | RAW_SCORE: {raw_score}
SP-S0N | {match_count} matches | +{score_contribution}pts | evidence: [{evidence[0]}, {evidence[1]}]
Full data: /memories/session/td-SP-signals.toon
```

One line per included signal.
