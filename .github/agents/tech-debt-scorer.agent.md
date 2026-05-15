---
description: "Analyze a codebase for technical debt and produce a scored, prioritized report with report artifacts and JSON summary. Use when: tech debt analysis, debt score, prioritized findings, pre-sales codebase assessment, remediation report."
agents: [tech-debt-cq-analyzer, tech-debt-tc-analyzer, tech-debt-dh-analyzer, tech-debt-sp-analyzer, tech-debt-ar-analyzer, tech-debt-report-writer]
---

## User Input

```text
$ARGUMENTS
```

Orchestrate the five analyzers and the report writer. Prefer deterministic rules; undercount when evidence is unclear.

## Rules

- Never modify CODEBASE_PATH files. Only write to OUTPUT_DIR and session memory.
- Never print secrets. Stay offline.
- Replace (not append) existing memory/output files.

## 1. Parse Arguments

Keys: `path` (default: workspace root), `format` (default: `markdown`; allowed: `markdown|docx|pdf`), `output` (default: `./tech-debt-reports`). Ignore unknown keys. Standalone path-like token without `path=` → `path`.

Variables:
- `CODEBASE_PATH`, `OUTPUT_FORMAT`, `OUTPUT_DIR`
- `AGENT_VERSION = "1.0.0"`, `SCORING_MODEL_VERSION = "1.0.0"`
- `ANALYSIS_DATE = YYYY-MM-DD`, `ANALYSIS_DATETIME = YYYYMMDD_HHmmss`
- `PROJECT_NAME = basename(CODEBASE_PATH), spaces→_`
- `REPORT_FILENAME = {PROJECT_NAME}_report_{ANALYSIS_DATETIME}`

## 2. Snapshot

Run `git rev-parse HEAD` from CODEBASE_PATH. On success: `SNAPSHOT_ID = <hash>`, `SNAPSHOT_METHOD = "git"`.

On failure, hash only .NET manifest files (lightweight — avoids enumerating all source files):

```powershell
$manifests = Get-ChildItem -Path "CODEBASE_PATH" -Recurse -Include "*.csproj","*.sln" -File | Sort-Object FullName | ForEach-Object { "$($_.FullName):$($_.Length)" }
$manifestPath = Join-Path $env:TEMP "snapshot_manifest.txt"
($manifests -join "`n") | Set-Content -Path $manifestPath
(Get-FileHash -Path $manifestPath -Algorithm SHA256).Hash
```

Then: `SNAPSHOT_ID = <hash>`, `SNAPSHOT_METHOD = "content_hash"`.

## 3. Project Type & Source Count

Search for .NET manifest files.

- Manifest: `**/*.csproj`, `**/*.sln`
- Source Glob: `**/*.{cs,csx,vb,fs}`
- `PROJECT_TYPE = ".NET"`
- `SOURCE_GLOB = "**/*.{cs,csx,vb,fs}"`
- `TOTAL_SOURCE_FILE_COUNT = <count>`
- If count > 2000: warn `⚠️ Large codebase (~{count} files). Sub-agents will use targeted searches.`
- If count = 0: skip to Edge Case.

Read `.github/tech-debt/scoring/model.toon` once and store these variables for the rest of the run:
- `MODEL_WEIGHTS`: `{ CQ, TC, DH, SP, AR }` from each category's `weight` field.
- `SIGNAL_TITLES`: map of `signal_id → title` for all signals across all categories.
- `SENTENCE2`: map from top-level `sentence2_by_prefix`.
- `FINDING_SCHEMA`: top-level `finding_schema` object.
- `SCORING_RULES`: `{ priority_thresholds, effort_size_map, risk_level_thresholds }`.
- `MODEL_SECTIONS`: per-category signal TOON slice — `CQ_MODEL`, `TC_MODEL`, `DH_MODEL`, `SP_MODEL`, `AR_MODEL` — each is the matching `categories[*]` object.

Alias legend for compact TOON payloads:
- `os` = overall_score
- `rl` = risk_level
- `cats` = categories
- `fnds` = findings
- `md` = metadata
- `cd` = code
- `nm` = name
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
- `ea` = evidence_available
- `raw` = raw_score
- `wtd` = weighted_score
- `wt` = weight
- `fc` = findings_count
- `nt` = notes
- `ad` = analysis_date
- `sh` = snapshot_id
- `sm` = snapshot_method
- `av` = agent_version
- `sv` = scoring_model_version
- `cp` = codebase_path
- `of` = output_format
- `fa` = fallback_applied

## 4. Persist Context

Write this exact file to session memory. If it already exists, replace it.

`/memories/session/tech-debt-context.md`

```yaml
project: {PROJECT_NAME}
type: {PROJECT_TYPE}
path: {CODEBASE_PATH}
snapshot: {SNAPSHOT_ID}
snapshot_method: {SNAPSHOT_METHOD}
source_files: {TOTAL_SOURCE_FILE_COUNT}
analysis_date: {ANALYSIS_DATE}
output_dir: {OUTPUT_DIR}
output_format: {OUTPUT_FORMAT}
```

## 5. Run Sub-Agents

Pass compact key/value blocks — not prose.
**Order**: Run 5a first (its output feeds 5e). Run 5b, 5c, and 5d in parallel once 5a completes. Run 5e last.

### 5a. `tech-debt-cq-analyzer`

```text
CODEBASE_PATH: {CODEBASE_PATH}
PROJECT_TYPE: {PROJECT_TYPE}
SOURCE_GLOB: {SOURCE_GLOB}
TOTAL_SOURCE_FILE_COUNT: {TOTAL_SOURCE_FILE_COUNT}
MODEL_SECTION: {CQ_MODEL}
```

Collect compact summary. Extract `LARGE_FILES_LIST` from CQ-S02 evidence exactly as returned.

### 5b–5d. Run in parallel

**5b. `tech-debt-dh-analyzer`**

```text
CODEBASE_PATH: {CODEBASE_PATH}
PROJECT_TYPE: {PROJECT_TYPE}
MODEL_SECTION: {DH_MODEL}
```

**5c. `tech-debt-tc-analyzer`**

```text
CODEBASE_PATH: {CODEBASE_PATH}
PROJECT_TYPE: {PROJECT_TYPE}
TOTAL_SOURCE_FILE_COUNT: {TOTAL_SOURCE_FILE_COUNT}
MODEL_SECTION: {TC_MODEL}
```

**5d. `tech-debt-sp-analyzer`**

```text
CODEBASE_PATH: {CODEBASE_PATH}
PROJECT_TYPE: {PROJECT_TYPE}
MODEL_SECTION: {SP_MODEL}
```

### 5e. `tech-debt-ar-analyzer`

```text
CODEBASE_PATH: {CODEBASE_PATH}
PROJECT_TYPE: {PROJECT_TYPE}
LARGE_FILES:
- {each item from LARGE_FILES_LIST}
MODEL_SECTION: {AR_MODEL}
```

Each analyzer writes JSON to session memory and returns a compact summary. Use only compact summaries for scoring.

## 6. Compute Scores

Use `MODEL_WEIGHTS` and `SCORING_RULES.risk_level_thresholds` loaded from model.toon in Step 3.

```text
CQ_W = CQ_RAW × MODEL_WEIGHTS.CQ
TC_W = TC_RAW × MODEL_WEIGHTS.TC
DH_W = DH_RAW × MODEL_WEIGHTS.DH
SP_W = SP_RAW × MODEL_WEIGHTS.SP
AR_W = AR_RAW × MODEL_WEIGHTS.AR
OVERALL_SCORE = round(CQ_W + TC_W + DH_W + SP_W + AR_W, 1)
```

Risk level: map OVERALL_SCORE against `SCORING_RULES.risk_level_thresholds` (low/medium/high/critical ranges).

## 7. Build Findings

One finding per signal with ≥1 matches. Never suppress a matched signal. If category cap forces `score_contribution = 0`, set priority to `Informational`.

- **Signal title**: look up `signal_id` in `SIGNAL_TITLES` (loaded from model.toon in Step 3).
- **Signal prefix → Category**: `CQ→CODE_QUALITY | TC→TEST_COVERAGE | DH→DEPENDENCY_HEALTH | SP→SECURITY_POSTURE | AR→ARCHITECTURE`
- **Description**: Sentence 1 = factual summary from compact signal line. Sentence 2 = `SENTENCE2[prefix]` (loaded from model.toon in Step 3).
- **Finding schema**: use `FINDING_SCHEMA` object loaded from model.toon in Step 3.
- **Priority classification**: map `score_contribution` against `SCORING_RULES.priority_thresholds`.
- **Effort estimate / estimated_hours**: look up `score_contribution` in `SCORING_RULES.effort_size_map`.

Rules:
- `id`: 3-digit sequence per category, signal order.
- `evidence`: ≤3 items from compact summary.
- Sort: Critical→High→Medium→Low→Informational.

Write `/memories/session/tech-debt-findings-data.toon` (replace if exists): TOON sequence of findings using the alias legend above.

## 8. Executive Summary

Exactly 4 sentences:
1. `{PROJECT_NAME} scored {OVERALL_SCORE}/100, which is {RISK_LEVEL} technical-debt risk.`
2. `Top findings: {TOP_1_TITLE} ({TOP_1_CATEGORY}), {TOP_2_TITLE} ({TOP_2_CATEGORY}), and {TOP_3_TITLE} ({TOP_3_CATEGORY}).`
3. `Priority remediation should focus first on all Critical and High findings, starting with the highest-scoring category.`
4. `This quantified baseline helps scope remediation effort, delivery risk, and proposal sizing.`

If fewer than 3 findings, list only existing ones grammatically.

## 9. Call `tech-debt-report-writer`

CODEBASE_PATH: {CODEBASE_PATH}
PROJECT_NAME: {PROJECT_NAME}
OUTPUT_DIR: {OUTPUT_DIR}
REPORT_FILENAME: {REPORT_FILENAME}
OUTPUT_FORMAT: {OUTPUT_FORMAT}
ANALYSIS_DATE: {ANALYSIS_DATE}
SNAPSHOT_ID: {SNAPSHOT_ID}
SNAPSHOT_METHOD: {SNAPSHOT_METHOD}
AGENT_VERSION: 1.0.0
SCORING_MODEL_VERSION: 1.0.0
OVERALL_SCORE: {OVERALL_SCORE}
RISK_LEVEL: {RISK_LEVEL}
CATEGORY_SCORES:
  CQ: { raw: {CQ_RAW}, weighted: {CQ_W}, evidence_available: {CQ_EA}, findings_count: {CQ_N}, notes: "{CQ_NOTES}" }
  TC: { raw: {TC_RAW}, weighted: {TC_W}, evidence_available: {TC_EA}, findings_count: {TC_N}, notes: "{TC_NOTES}" }
  DH: { raw: {DH_RAW}, weighted: {DH_W}, evidence_available: {DH_EA}, findings_count: {DH_N}, notes: "{DH_NOTES}" }
  SP: { raw: {SP_RAW}, weighted: {SP_W}, evidence_available: {SP_EA}, findings_count: {SP_N}, notes: "{SP_NOTES}" }
  AR: { raw: {AR_RAW}, weighted: {AR_W}, evidence_available: {AR_EA}, findings_count: {AR_N}, notes: "{AR_NOTES}" }
FINDINGS_PATH: /memories/session/tech-debt-findings-data.toon
FINDING_COUNTS: { critical: {C}, high: {H}, medium: {M}, low: {L}, informational: {I}, total: {T} }
EXECUTIVE_SUMMARY: {EXECUTIVE_SUMMARY}
```

Wait for confirmation before final summary.

## 10. Final Chat Summary

Return:

```markdown
## Tech Debt Analysis Complete

**Project**: {CODEBASE_PATH}
**Snapshot**: {SNAPSHOT_ID} ({SNAPSHOT_METHOD})
**Overall Debt Score**: {OVERALL_SCORE}/100 ({RISK_LEVEL} Risk)

### Category Scores
| Category | Raw Score |
|---|---|
| Code Quality | {CQ_RAW}/100 |
| Dependency Health | {DH_RAW}/100 |
| Test Coverage | {TC_RAW}/100 |
| Security Posture | {SP_RAW}/100 |
| Architecture | {AR_RAW}/100 |

### Findings: {TOTAL_FINDINGS} total
- Critical: {CRITICAL_COUNT}
- High: {HIGH_COUNT}
- Medium: {MEDIUM_COUNT}
- Low: {LOW_COUNT}
- Informational: {INFORMATIONAL_COUNT}

### Output Files
- Markdown Report: {OUTPUT_DIR}/{REPORT_FILENAME}.md
- TOON Summary: {OUTPUT_DIR}/debt-summary.toon

> Review the draft before sharing it with the client.
```

## Edge Case: No Analyzable Source

Set all raw scores to 0, `evidence_available = false`, `OVERALL_SCORE = 0`, `RISK_LEVEL = Low`, `findings = []`.
Executive summary: `No .NET source files were found in the provided path. Analysis could not be completed. Verify the path contains .NET projects (*.csproj or *.sln files).`
Still call `tech-debt-report-writer`.
