# Technical Debt Report: {PROJECT_NAME}

**Generated**: {ANALYSIS_DATE}  
**Overall Debt Score**: {OVERALL_SCORE}/100 — **{RISK_LEVEL} Risk**  
**Scoring Model**: v{SCORING_MODEL_VERSION} | **Agent**: v{AGENT_VERSION}  
**Snapshot**: `{SNAPSHOT_ID}` (via {SNAPSHOT_METHOD})

---

## Executive Summary

{EXECUTIVE_SUMMARY_PROSE}

### Top Risks

| # | Priority | Finding | Category | Effort |
|---|---|---|---|---|
| 1 | {P1_PRIORITY} | {P1_TITLE} | {P1_CATEGORY} | {P1_EFFORT} |
| 2 | {P2_PRIORITY} | {P2_TITLE} | {P2_CATEGORY} | {P2_EFFORT} |
| 3 | {P3_PRIORITY} | {P3_TITLE} | {P3_CATEGORY} | {P3_EFFORT} |

### Recommended Remediation Path

{REMEDIATION_ROADMAP_PROSE}

### Scoring Model — What These Numbers Mean

Debt scores range from **0 (no debt) to 100 (maximum debt)**. An overall score above 75 indicates
critical technical risk requiring immediate attention before major new feature work. Scores
between 51–75 represent significant but addressable debt that will increasingly slow delivery.
Scores of 25 or below indicate a well-maintained codebase.

Each category is weighted by its impact on delivery risk:

| Category | Weight |
|---|---|
| Code Quality | 25% |
| Dependency Health | 25% |
| Test Coverage | 20% |
| Security Posture | 20% |
| Architecture | 10% |

---

## Debt Score Breakdown

| Category | Weight | Raw Score | Weighted Score | Findings |
|---|---|---|---|---|
| Code Quality | 25% | {CQ_RAW}/100 | {CQ_WEIGHTED} | {CQ_COUNT} |
| Test Coverage | 20% | {TC_RAW}/100 | {TC_WEIGHTED} | {TC_COUNT} |
| Dependency Health | 25% | {DH_RAW}/100 | {DH_WEIGHTED} | {DH_COUNT} |
| Security Posture | 20% | {SP_RAW}/100 | {SP_WEIGHTED} | {SP_COUNT} |
| Architecture | 10% | {AR_RAW}/100 | {AR_WEIGHTED} | {AR_COUNT} |
| **Overall** | **100%** | — | **{OVERALL_SCORE}/100** | **{TOTAL_COUNT}** |

> Categories marked `[evidence unavailable]` used a conservative default score.
> Findings from those categories should be validated manually.

---

## Prioritized Finding List

> Ordered: Critical → High → Medium → Low

{FINDING_LIST}

<!--
Each finding uses this format:

### [{PRIORITY}] {ID}: {TITLE}

**Category**: {Category Name}  
**Effort**: {SIZE} ({time label}) — Estimated: {HOURS_MIN}–{HOURS_MAX} hours  
**Business Impact**: {business_impact}

**Evidence**: `{evidence_path}` [, `{evidence_path_2}`, ...]

**Description**: {description}

**Recommended Solution**:

1. {step_1}
2. {step_2}
3. {step_3}
...

**Why This Approach**: {solution_rationale}

---
-->

---

## Report Metadata

| Field | Value |
|---|---|
| Analysis Date | {ANALYSIS_DATE} |
| Codebase Path | `{CODEBASE_PATH}` |
| Snapshot ID | `{SNAPSHOT_ID}` |
| Snapshot Method | {SNAPSHOT_METHOD} |
| Agent Version | {AGENT_VERSION} |
| Scoring Model Version | {SCORING_MODEL_VERSION} |
| Output Format | {OUTPUT_FORMAT} |
| Fallback Applied | {FALLBACK_APPLIED} |

---

*This report is a draft deliverable. Review and validate all findings before sharing with the client.*
