---
name: decision-synthesizer
description: Synthesizes diffs and evidence from parallel variants into a structured 4-Act Decision Card with race-safe snapshotting.
---

# 4-Act Decision Synthesizer (Arbitration Engine)

Use this skill when comparing competing code variants or parallel worktrees to generate an executive decision card for tech leads in under 60 seconds.

## 1. Immutable Evidence Snapshot Pattern
To avoid race conditions when worktrees are deleted on Pick (`server.ts:5000`) or cleaned up by retention (`runs/retention.ts`):
- **Pre-Evaluation Snapshot:** Before reading live files, compile an immutable evidence object using `resolveTaskDiffBase`:
  - `diffStat`: Summary of files touched, insertions, deletions (`git diff --stat`).
  - `unifiedDiff`: Bounded diff excerpt (max 300 lines per file, ignoring lockfiles).
  - `verificationEvidence`: Test commands executed and exact exit codes (0/1) extracted from sibling transcripts.
  - `inputRevisionId`: Hash of the settled sibling heads.
- **Tool-Free Evaluator:** Evaluate exclusively against this snapshot payload in prompt context without mounting live worktrees.

## 2. Race Condition & Early-Pick Safety
- If a human reviewer clicks **„Pick Winner”** while synthesis is running:
  - The evaluator task immediately aborts via `AbortController`.
  - Discard partial output to avoid stale card flashes.
  - Sibling directory deletion on pick never blocks or fails due to an active evaluator.

## 3. Structured 4-Act Output
Generate both human-readable Markdown and structured JSON for UI matrix rendering:

### Act 1: Concrete Scenario
The exact problem, affected modules, and user story being resolved.

### Act 2: Value at Risk / Cost of Inaction
What architectural flaws, performance degradations, or regressions risk emerging if the inferior approach is selected.

### Act 3: Trade-off Matrix (Structured Table)
| Criterion | Variant A | Variant B |
| :--- | :--- | :--- |
| **Cognitive Load & Complexity** | Minimal / High | Clean / Complex |
| **Test Coverage & Verification** | [EXECUTED: exit 0] | [REASONED only] |
| **Backward Compatibility & Breaking Risk** | None | Schema change |
| **Diff Blast Radius** | +42 / -12 lines | +310 / -45 lines |

### Act 4: Decisive Recommendation with Grounded Metrics
- **Winner:** Variant A or B.
- **Decisive Argument:** Single most compelling factor.
- **Verification Metric:** Verified command from transcript proving correctness (e.g. `npm run test:unit passed in 4.2s`).
