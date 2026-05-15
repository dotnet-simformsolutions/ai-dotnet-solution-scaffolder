---
description: "Assemble the final tech debt report files from scored findings and metadata. Use when: write tech debt report, create markdown report, write debt summary JSON."
tools: [read, edit]
user-invocable: false
---

## Rules

- Do not re-scan CODEBASE_PATH. Only write to OUTPUT_DIR.
- Use EXECUTIVE_SUMMARY exactly as passed.
- Load `tech-debt-remediation` skill only if ≥1 finding has an empty `remediation`, `solution`, `solution_rationale`, or `business_impact` field.
- Read per-category signal memory only when a finding has empty `evidence`.
- Missing required field → stop with clear error.

## Inputs

Orchestrator passes: `CODEBASE_PATH`, `PROJECT_NAME`, `OUTPUT_DIR`, `REPORT_FILENAME`, `OUTPUT_FORMAT`, `ANALYSIS_DATE`, `SNAPSHOT_ID`, `SNAPSHOT_METHOD`, `AGENT_VERSION`, `SCORING_MODEL_VERSION`, `OVERALL_SCORE`, `RISK_LEVEL`, `CATEGORY_SCORES` (CQ/TC/DH/SP/AR, each with raw/weighted/evidence_available/findings_count/notes), `FINDINGS_PATH` (TOON sequence), `FINDING_COUNTS`, `EXECUTIVE_SUMMARY`.

TOON alias legend for internal payloads:
- `os` = overall_score
- `rl` = risk_level
- `cats` = categories
- `fnds` = findings
- `md` = metadata
- `cd` = code
- `nm` = name
- `wt` = weight
- `raw` = raw_score
- `wtd` = weighted_score
- `ea` = evidence_available
- `fc` = findings_count
- `nt` = notes
- `sid` = signal_id
- `cat` = category
- `pr` = priority
- `ttl` = title
- `desc` = description
- `ev` = evidence
- `rem` = remediation
- `sol` = solution
- `sol_r` = solution_rationale
- `biz` = business_impact
- `ef_es` = effort_estimate
- `est_h` = estimated_hours
- `sc` = score_contribution
- `ad` = analysis_date
- `sh` = snapshot_id
- `sm` = snapshot_method
- `av` = agent_version
- `sv` = scoring_model_version
- `cp` = codebase_path
- `of` = output_format
- `fa` = fallback_applied

Optional: `TOTAL_SOURCE_FILE_COUNT` and `PROJECT_TYPE` if available.

Each finding must have: `id`, `signal_id`, `category`, `priority`, `title`, `description`, `evidence`, `remediation`, `solution`, `solution_rationale`, `business_impact`, `effort_estimate`, `estimated_hours`, `score_contribution`.

## 1. Load

Read `FINDINGS_PATH` and `.github/tech-debt/templates/report-template.md`.
If any finding has empty `evidence`, read the relevant `/memories/session/td-{CAT}-signals.toon`.

## 2. Populate Remedy Fields

Per finding: if all of `remediation`, `solution`, `solution_rationale`, `business_impact` are populated → skip. Otherwise use `tech-debt-remediation` skill, adapted with finding title/category/evidence.
- `remediation`: 1 sentence. `solution`: numbered steps. `solution_rationale`: 1 paragraph. `business_impact`: 1 sentence.

## 3. Build Markdown

Substitute all placeholders in `.github/tech-debt/templates/report-template.md`.

Fixed values:
- `EXECUTIVE_SUMMARY_PROSE = EXECUTIVE_SUMMARY`
- `REMEDIATION_ROADMAP_PROSE = Resolve Critical and High findings first, starting with the category that has the highest raw score. Then address medium-priority maintainability items and add CI checks to prevent recurrence.`
- `FALLBACK_APPLIED = true` when any category `evidence_available = false` OR `OUTPUT_FORMAT` is `docx`/`pdf`; else `false`.
- Top risks = first 3 findings by priority; pad with `-` if fewer.

Category display names: `CODE_QUALITY→Code Quality | TEST_COVERAGE→Test Coverage | DEPENDENCY_HEALTH→Dependency Health | SECURITY_POSTURE→Security Posture | ARCHITECTURE→Architecture`.
Category table order: CQ, TC, DH, SP, AR.
Render findings in priority order per template comment block.

## 4. Write Markdown

`{OUTPUT_DIR}/{REPORT_FILENAME}.md` — create or replace.

## 5. HTML Dashboard (Conditional)

Execute this step when `OUTPUT_FORMAT = html` OR `GENERATE_DASHBOARD = true`.

### 5a. Load Skill

Load `tech-debt-html-report` skill and read `src/templates/report-template.html`.

### 5b. Compute All Placeholders

| Placeholder | Computation |
|---|---|
| `{SCORE_RING_OFFSET}` | `round(314 × (1 − OVERALL_SCORE / 100))` |
| `{SCORE_COLOR}` | `#ef4444` if High/Critical; `#eab308` if Medium; `#22c55e` if Low |
| `{RISK_LEVEL_ICON}` | `🔴` Critical or High; `🟡` Medium; `🟢` Low |
| `{SOURCE_SUMMARY}` | `"{TOTAL_SOURCE_FILE_COUNT} source files · {PROJECT_TYPE}"` — use `"–"` if not available |
| `{TARGET_SCORE}` | `45` |
| `{CQ_WEIGHTED}` … `{AR_WEIGHTED}` | From `CATEGORY_SCORES.*.weighted`, rounded to 1 decimal |
| `{CQ_COUNT}` … `{AR_COUNT}` | From `CATEGORY_SCORES.*.findings_count` |
| `{SP_WEIGHT_PCT}` … `{AR_WEIGHT_PCT}` | `CATEGORY_SCORES.*.weight × 100` |

**KPI Deltas**: Check if `{OUTPUT_DIR}/debt-summary.toon` exists before this write.
If yes: read previous `findings` counts and compare to current `FINDING_COUNTS`. Emit `{CRITICAL_DELTA_HTML}` etc. as described in the skill. If no previous file: emit `<div class="kpi-delta" style="color:var(--muted)">— first scan</div>` for all five.

**Effort size counts**: Count findings by `effort_estimate` field value:
`{EFFORT_S_COUNT}`, `{EFFORT_M_COUNT}`, `{EFFORT_L_COUNT}`, `{EFFORT_XL_COUNT}`.

**Trend**: Read previous `debt-summary.toon` if it exists (before overwriting) to get the previous `overall_score` and `metadata.analysis_date`. Build:
- `{TREND_LABELS_JSON}` — e.g. `["Jan 2026","Apr 2026"]` (format analysis dates as `Mon YYYY`).
- `{TREND_SCORES_JSON}` — e.g. `[68, 72]`.
- If no previous data: single-element arrays.

**Effort by Category**: Parse `estimated_hours` per finding (e.g. `"40–60 h"` → min=40, max=60). Sum per category.
Produce `{EFFORT_BY_CAT_LABELS_JSON}`, `{EFFORT_BY_CAT_MIN_JSON}`, `{EFFORT_BY_CAT_MAX_JSON}`.

**Sprint Projected Scores**: Group findings into sprints:
- Sprint 1: Critical SP+DH findings
- Sprint 2–3: Remaining Critical + all High
- Sprint 4–5: High/Critical AR findings
- Sprint 6: Medium findings
Subtract cumulative `score_contribution` from `OVERALL_SCORE` at each sprint.
Produce `{SPRINT_PROJECTED_LABELS_JSON}` and `{SPRINT_PROJECTED_SCORES_JSON}`.
`goal_score` = max(sum of Low+Info score_contributions, 5).

**Effort by Priority**: Sum max hours per priority level.
Produce `{EFFORT_BY_PRIORITY_JSON}` = `[critical_max, high_max, medium_max, low_max]`.

### 5c. Build HTML Block Sections

Build all HTML block sections per the skill instructions:
- `{TOP_RISKS_TABLE_ROWS}` — top 3 findings
- `{CATEGORY_TABLE_ROWS}` — 5 rows with progress bars and score colours
- `{FINDING_DETAILS_HTML}` — full accordion items with `data-priority` attribute
- `{REMEDIATION_TABLE_ROWS}` — Critical + High + Medium findings only
- `{SPRINT_ROADMAP_HTML}` — timeline items per sprint

### 5d. Write HTML

Substitute all placeholders and write to `{OUTPUT_DIR}/{REPORT_FILENAME}.html` — create or replace.

For `docx`/`pdf` (not `GENERATE_DASHBOARD`): keep markdown as fallback, set `FALLBACK_APPLIED = true`.

## 6. JSON Summary

Validate: `os` 0–100, `rl` ∈ {Low,Medium,High,Critical}, `cats` has 5 entries (CQ,TC,DH,SP,AR order), `fnds` present, `md` complete.

```text
os: 0
rl: Low
cats:
  - nm: Code Quality
    cd: CQ
    wt: 0.25
    raw: 0
    wtd: 0
    fc: 0
    ea: true
    nt: ""
fnds:
  - id: ""
    sid: ""
    cat: ""
    pr: ""
    ttl: ""
    desc: ""
    ev: []
    rem: ""
    sol: ""
    sol_r: ""
    ef_es: ""
    est_h: ""
    biz: ""
    sc: 0
md:
  ad: ""
  sh: ""
  sm: ""
  av: ""
  sv: ""
  cp: ""
  of: ""
  fa: false
```

Write `{OUTPUT_DIR}/debt-summary.toon` — create or replace.

## 7. Return

```text
## Report Files Written
- Markdown: {OUTPUT_DIR}/{REPORT_FILENAME}.md
- TOON: {OUTPUT_DIR}/debt-summary.toon
- Format fallback: {yes|no}

Total findings: {TOTAL} — Critical: {C} · High: {H} · Medium: {M} · Low: {L} · Informational: {I}
```
