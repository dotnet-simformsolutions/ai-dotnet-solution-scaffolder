---
name: tech-debt-clarification
description: 'Guide dynamic score clarification for tech debt findings. Identifies ambiguous signals, asks targeted user questions, and applies score adjustments. Load in Step 4f when any sub-agent has 1+ signal matches. Triggers: clarification, ambiguous findings, score adjustment, ask questions, user context.'
user-invocable: false
---

# Tech Debt Clarification

For each signal with 1+ matches, decide whether user context could materially change the score. Generate questions based on actual findings — not a pre-written script.

## Step 1 — Identify Ambiguous Findings

Per signal with 1+ matches, ask: "Could this be explained by a legitimate decision, reducing score by ≥3 points?"
- **Yes** → ambiguous → formulate a question.
- **No** → score directly.

Almost always **unambiguous** (skip unless unusual context): SQL injection patterns, dangerous exec with user input, `.env` files with real secrets.

Almost always **ambiguous** (always reason about):
- Credentials in config → could be dev-only stubs (Critical→Low)
- Large files → could be generated/migrated/framework code
- Magic numbers → could be HTTP codes, domain constants, documented rules
- Deep nesting → could be ORM/framework patterns
- No coverage report → may be enforced externally
- Outdated packages → may be intentionally pinned

## Step 2 — Reason Before Asking

Per ambiguous finding:
1. Examine specific files, counts, or versions that triggered it.
2. Identify what only the user can confirm.
3. Only ask if score delta ≥ 3 points.
4. Reference actual file names/counts/versions — not boilerplate.

## Step 3 — Ask Questions

Use `vscode_askQuestions`. Group related signals. Each question must:
- Open with factual summary (count, paths, patterns).
- Explain why ambiguous (1 sentence).
- Present 2–4 options — at least one "legitimate" and one "genuine debt".

## Step 4 — Apply Score Adjustments

| User confirms | Adjustment |
|---|---|
| Fully explained by legitimate decision | Reduce contribution 50–80% |
| Partially explained | Remove explained proportion |
| Confirmed genuine debt | Keep full contribution |
| Unsure / cannot confirm | Keep full + note "Conservative score applied" |

Record user's answer in the finding's `description`. Return control to orchestrator for Step 5.
