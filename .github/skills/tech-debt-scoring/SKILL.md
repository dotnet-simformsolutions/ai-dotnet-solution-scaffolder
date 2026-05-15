---
name: tech-debt-scoring
description: 'Priority classification, risk level thresholds, effort sizing, and conservative defaults for tech debt findings. Load before classifying findings (Step 6) or sizing remediation effort. Triggers: risk level, priority classification, effort size, T-shirt size, score contribution, conservative default score.'
user-invocable: false
---

# Tech Debt Scoring

## Priority Classification

Assign priority based on `score_contribution` for the finding:

| score_contribution          | Priority      |
|-----------------------------|---------------|
| ≥ 15                        | Critical      |
| 8–14                        | High          |
| 3–7                         | Medium        |
| 1–2                         | Low           |
| 0 (but ≥ 1 match; cap hit)  | Informational |

**Rule**: Every signal with 1+ matches MUST produce a finding. Never suppress a match silently.
Zero-match signals produce no finding and are noted in the category `notes` field only.
Sort all findings: Critical → High → Medium → Low → Informational.

## Effort Sizing

Map `score_contribution` to T-shirt size. Always include both the size label and a concrete hour range.

| score_contribution | Size | Label        | Estimated Hours |
|--------------------|------|--------------|-----------------|
| ≤ 2                | XS   | ≤ 2 hours    | 1–2 hrs         |
| 3–6                | S    | ≤ 1 day      | 3–8 hrs         |
| 7–14               | M    | ≤ 1 week     | 9–40 hrs        |
| 15–24              | L    | ≤ 1 month    | 41–160 hrs      |
| ≥ 25               | XL   | > 1 month    | 160+ hrs        |

Format: `M (≤ 1 week) — Estimated: 16–24 hours`

**Adjust hours UP when:**
- Finding spans multiple services or projects
- Codebase has low test coverage (higher validation effort)
- Fix requires data migration or credential rotation

**Adjust hours DOWN when:**
- Tooling already exists but is not wired (e.g., `coverlet` in csproj but no CI step)
- Fix is a single config change or `dotnet add package` command

## Conservative Defaults

When evidence is unavailable (manifest missing, access error, binary files only), apply these
conservative raw scores and note `[evidence unavailable]` in the report:

| Category         | Default Score |
|------------------|---------------|
| CQ               | 40            |
| TC               | 60            |
| DH               | 50            |
| SP               | 55            |
| AR               | 40            |
