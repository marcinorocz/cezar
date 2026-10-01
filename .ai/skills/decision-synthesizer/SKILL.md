---
name: decision-synthesizer
description: Synthesizes diffs and evidence from parallel variants into a structured 4-Act Decision Card with race-safe snapshotting and safety escalation.
---

# 4-Act Decision Synthesizer (Arbitration Engine)

Use this skill when comparing competing code variants or parallel worktrees to generate an executive decision card for tech leads in under 60 seconds.

## 1. Immutable Evidence Snapshot Pattern
To avoid race conditions when worktrees are deleted on Pick (`server.ts:5000`) or cleaned up by retention (`runs/retention.ts`):
- **Pre-Evaluation Snapshot:** Before reading live files, compile an immutable in-memory or JSON evidence object:
  - `diffStat`: Summary of files touched, insertions, deletions (`git diff --stat`).
  - `unifiedDiff`: Bounded diff excerpt (max 300 lines per file, ignoring lockfiles).
  - `verificationEvidence`: Test commands executed and exact exit codes (0/1) extracted from sibling transcripts.
  - `sha256`: Checksum of the snapshot payload and git tree hashes (`inputRevisionId`).
- Evaluate exclusively against this snapshot — never rely on worktree paths staying mounted during human decision-making.
- If a variant changes or is continued before synthesis finishes, the result is marked `stale` and discarded rather than overwriting.

## 2. Race Condition & Safety Escalation Rules
- **Early-Pick Safety:** If a human reviewer clicks **„Pick Winner”** while synthesis is running, the evaluator task immediately aborts via `AbortController`. Directory deletion on pick never blocks.
- **Safety Objection Escalation (No Majority Voting):**
  - If any agent or review step raises a **security, privacy (PII/GDPR), or data corruption objection** against a variant, **it CANNOT be outvoted** by other criteria.
  - The arbitration engine must immediately escalate the safety objection to the human lead in Act 2 and Act 4, requiring explicit human sign-off.

## 3. The 4-Act Output Structure
Generate both human-readable Markdown and structured JSON for UI matrix rendering:

### Act 1: Concrete Scenario
The exact problem, affected modules, and user story being resolved.

### Act 2: Value at Risk / Cost of Inaction
What architectural flaws, performance degradations, regressions, or security hazards risk emerging if the inferior approach is selected.

### Act 3: Trade-off Matrix (Structured Table)
| Criterion | Variant A | Variant B |
| :--- | :--- | :--- |
| **Cognitive Load & Complexity** | Minimal / High | Clean / Complex |
| **Test Coverage & Verification** | [EXECUTED: exit 0] | [REASONED only] |
| **Security & Safety Invariants** | Verified clean | Flagged risk (Escalate) |
| **Backward Compatibility & Breaking Risk** | None | Schema change |
| **Diff Blast Radius** | +42 / -12 lines | +310 / -45 lines |

### Act 4: Decisive Recommendation with Grounded Metrics
- **Winner:** Variant A or B (or Escalation if safety violation detected).
- **Decisive Argument:** Single most compelling factor.
- **Verification Metric:** Verified command from transcript proving correctness (e.g. `npm run test:unit passed in 4.2s`).
