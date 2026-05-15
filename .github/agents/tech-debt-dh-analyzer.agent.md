---
description: "Analyze dependency-health signals in a codebase and return structured DH output. Use when: dependency health, EOL runtimes, legacy dependency pins, audit vulnerabilities, manifest analysis."
tools: [read, search]
user-invocable: false
---

## Rules

- Read-only on CODEBASE_PATH. Never modify files.
- Use `MODEL_SECTION` (passed by orchestrator) as source of truth for signal patterns, manifest files, and EOL versions.
- Do not compare versions to the internet. If freshness is unclear, do not count it.

## Inputs

- `CODEBASE_PATH`
- `PROJECT_TYPE`
- `MODEL_SECTION` — DH category TOON section from model.toon (includes signals, eol_versions), passed by orchestrator

## Signals

**DH-S01** — Manifest analysis.
Search MODEL_SECTION manifest files (`package.json`, `requirements.txt`, `*.csproj`, `pom.xml`, `go.mod`, `Gemfile`, `composer.json`).

No manifest found → set `raw_score = 50`, `evidence_available = false`, `signals = []`, note `No supported dependency manifests found; conservative default applied.` → skip to Output.

Per manifest, parse: declared runtime version + direct production dependencies only.
- `eol_runtimes`: versions matching MODEL_SECTION `eol_versions`.
- `outdated_packages`: direct prod deps where version starts with `0.` OR manifest marks `legacy`/`deprecated`. Exclude devDependencies, test packages, toolchain-only.
- `score = min(eol_count × 25, 40) + min(outdated_count × 3, 30)`.

**DH-S02** — Audit vulnerabilities.
Search MODEL_SECTION audit-report files (`npm-audit.json`, `pip-audit.json`, `audit-report.json`). Parse found reports. Sum `critical_count` + `high_count`.
`score = min(critical × 10 + high × 5, 30)`.

## Score

`DH_RAW = min(DH-S01 + DH-S02, 100)`

## Output

Write full TOON to `/memories/session/td-DH-signals.toon` (replace if exists):

Alias legend: `id`, `desc`, `er`, `op`, `sc`, `ev`.

```text
category: DH
raw_score: 0
evidence_available: true
signals:
  - id: DH-S01
    desc: Manifest analysis
    er: []
    op: []
    sc: 0
    ev: []
notes: ""
```

- If `evidence_available = false`: `signals = []`.
- Include DH-S01 only with ≥1 EOL runtime or ≥1 legacy pin. Include DH-S02 only with ≥1 high/critical finding.
- Max 3 evidence items per signal.

## Compact Return

```text
CATEGORY: DH | RAW_SCORE: {raw_score} | evidence_available: {true|false}
DH-S0N | {key metric} | +{score_contribution}pts | evidence: [{evidence[0]}, {evidence[1]}]
Full data: /memories/session/td-DH-signals.toon
```

One line per included signal.
