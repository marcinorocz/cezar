---
name: decision-synthesizer
description: Synthesizes diffs and evidence from parallel variants into a structured 4-Act Decision Card with atomic snapshotting, manifest verification, and safety veto escalation.
---

# 4-Act Decision Synthesizer (Arbitration Engine)

Use this skill when comparing competing code variants or parallel worktrees to generate an executive decision card for tech leads in under 60 seconds.

## 1. Immutable Evidence Snapshot Pattern
To eliminate TOCTOU (time-of-check to time-of-use) race conditions when worktrees are deleted on Pick (`server.ts:5000`) or cleaned up by retention (`runs/retention.ts`):

- **Atomic Write-Once Assembly:**
  1. Snapshot is prepared in a temporary directory (e.g. `.ai/cezar/tmp/<groupId>-<snapshotId>`) and finalized via a single atomic `fs.rename` (or equivalent POSIX atomic rename).
  2. Snapshot directory is read-only after publication.
- **Manifest as Single Source of Truth (`manifest.jsonl`):**
  - Contains one JSON line per file: `{"path": "...", "sha256": "...", "size": 1240, "mtime": 1711928000, "timestamp": "...", "runId": "..."}`.
  - Downstream evaluator tools and comparisons MUST read exclusively from this manifest and snapshot payload, NEVER from live worktrees.
- **Hard Abort on Drift (`EVIDENCE_DRIFT`):**
  - If any downstream read detects a hash mismatch against `manifest.jsonl`, the arbitration job immediately aborts with error code `EVIDENCE_DRIFT`. It never quietly proceeds with corrupted or altered state.
- **Filesystem Security Hygiene:**
  - **Symlink Policy:** Refuse to traverse symlinks pointing outside the workspace (or canonicalize and verify boundaries) to prevent path traversal attacks.
  - **Resource Bounds:** Hard bounds per variant: max 300 diff lines per file, max 10MB per snapshot, max 50 touched files.
  - **Retention & GC:** Evidence snapshots share the lifecycle of the parent `runId` / `groupId` and are cleaned up by `runs/retention.ts`.

## 2. Multi-Agent Decision Protocol & Objection Categorization

When synthesizing feedback or arbitrating across multiple agents/evaluators:

### A. Blocking Objections (Veto - No Majority Voting)
- **Categories:** Security vulnerabilities, credential/PII leaks, data corruption risks, broken safety invariants.
- **Escalation Rule:** If ANY agent or tool flags a blocking objection against a variant, **it CANNOT be overridden by majority vote**.
- **Action:** The synthesis card flags an immediate **SAFETY_ESCALATION** to the human tech lead in Act 2 and Act 4, blocking automated merge.

### B. Non-Blocking Objections (Majority / Trade-off)
- **Categories:** Code aesthetics, naming conventions, minor performance trade-offs, optional refactor suggestions.
- **Resolution:** Resolved via majority consensus or weighted trade-off analysis in the Act 3 matrix.

### C. Dissensus Transparency
- Never fabricate false consensus. If a significant divergence of technical approach exists, the decision card explicitly documents the dissenting opinion and trade-off in Act 3 and Act 4.

## 3. The 4-Act Output Structure

Generate both human-readable Markdown and structured JSON for UI cockpit rendering:

### Act 1: Concrete Scenario
The exact problem, affected modules, and user story being resolved.

### Act 2: Value at Risk / Cost of Inaction
Architectural flaws, performance regressions, or security hazards if the inferior approach or unsafe variant is selected.

### Act 3: Trade-off Matrix (Structured Table)
| Criterion | Variant A | Variant B |
| :--- | :--- | :--- |
| **Cognitive Load & Complexity** | Minimal / High | Clean / Complex |
| **Test Coverage & Verification** | [EXECUTED: exit 0] | [REASONED only] |
| **Security & Safety Invariants** | Verified clean | Flagged risk (`EVIDENCE_DRIFT` / Block) |
| **Backward Compatibility & Breaking Risk** | None | Schema change |
| **Diff Blast Radius** | +42 / -12 lines | +310 / -45 lines |

### Act 4: Decisive Recommendation with Grounded Metrics
- **Winner:** Winning Run ID (or **ESCALATE TO HUMAN** if blocking objection is active).
- **Decisive Factor:** Single most compelling architectural or empirical argument.
- **Verification Metric:** Verified command from transcript proving correctness (e.g. `npm run test:unit passed in 4.2s (exit 0)`).
- **Snapshot Ref:** `manifestSha256` proving immutable provenance.
