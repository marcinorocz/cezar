# Group-Level Variant Arbitration & 4-Act Decision Synthesis

- **Status:** Proposed
- **Relates to:** Issue #1157

## Context

Cezar provides parallel task execution via `variants: 1..3` on `POST /runs` (`.ai/specs/010-parallel-variants.md`). This spawns N independent sibling tasks in isolated worktrees grouped by `groupId`, which are compared side-by-side in `packages/web/src/routes/compare-variants.tsx`.

Line 45 of `.ai/specs/010-parallel-variants.md` explicitly reserved this capability for v2:
> *"Auto-ocena wariantów przez AI-sędziego (kusi, ale to v2 — najpierw człowiek)"*

Now that human review of variants is battle-tested, tech leads face high cognitive load comparing raw multi-file diffs. Furthermore, `packages/cezar/src/server/server.ts:5000` synchronously deletes losing worktrees and branches upon picking a winner, while `runs/retention.ts` reclaims finished worktree allocations. Therefore, comparative arbitration must operate independently of the live, fragile filesystem.

## Decision

### 1. Unified Reconciliation Hook & Terminal Transitions
Rather than relying exclusively on `RunManager.dropActive()` (which is bypassed when `cancelOne()` cancels queued tasks in `workflows/run.ts:2810–2823` or during restart recovery in `:1580–1582, :1689–1693`), arbitration settlement funnels through a single shared entry point:
- **`reconcileGroupArbitration(groupId, idempotencyKey)`:** Called whenever any sibling transitions to a terminal state (`done`, `review`, `failed`, `cancelled`).
- **Idempotency Guard:** Guarded by an idempotency key `group-arbitration:<groupId>:<revisionHash>`, ensuring that concurrent settlements or restart sweeps never launch duplicate paid evaluations.
- **Viability Predicate:** The hook launches evaluation **only when all siblings in the `groupId` are terminal, AND at least 2 siblings have reached `done` or `review`** (with valid code). If only 1 sibling survives, arbitration is skipped.

### 2. Immutable Evidence Snapshot Pattern (Race-Free Evaluation)
To prevent stale-result races when a user clicks **"Pick Winner"** while evaluation is running:
- **Pre-Evaluation Snapshot:** Before dispatching the evaluator, the engine compiles an immutable evidence snapshot using `resolveTaskDiffBase`:
  - `diffStat`: Files touched, additions, deletions.
  - `unifiedDiff`: Bounded diff excerpt (max 300 lines per file, ignoring lockfiles).
  - `verificationEvidence`: Exact test commands and exit codes (0/1) extracted from the sibling transcripts.
  - `input_revision_id`: Hash of the settled sibling heads.
- **Tool-Free Evaluator:** The evaluator agent runs in **tool-free, read-only mode**, receiving the snapshot payload directly in its prompt rather than mounting or inspecting live worktrees.
- **Early-Pick / Stale Result Disposal:** If a user clicks Pick before evaluation completes, the evaluator is cancelled via `AbortController`, and any late-arriving synthesis is discarded if its `input_revision_id` does not match the active state.

### 3. State & Artifact Storage (Structured Data)
- The synthesis card is persisted to disk as `.ai/cezar/groups/<groupId>/synthesis.json` (registered in `DATA_GITIGNORE_ENTRIES`).
- Served via `GET /api/v1/groups/:groupId/synthesis`.
- Contract schema in `packages/contract`:
```ts
export const variantSynthesisSchema = z.object({
  groupId: z.string(),
  scenario: z.string(),
  valueAtRisk: z.string(),
  tradeoffs: z.array(z.object({
    dimension: z.string(),
    variants: z.record(z.string(), z.string()), // runId -> summary
  })),
  recommendation: z.object({
    winningRunId: z.string(),
    rationale: z.string(),
    verificationSummary: z.string(),
  }),
  inputRevisionId: z.string(),
  createdAt: z.string(),
});
```

### 4. Background Evaluator Resource & Safety Contract
- **Strictly Opt-In:** Gated by an explicit `--arbitrate` flag or composer checkbox; default **off**.
- **Execution Budget:** Uses a fast, cost-effective model (inheriting runner profile or defaulting to a lightweight evaluator).
- **Execution Limits:** Hard timeout of 60 seconds. Does not consume a standard interactive concurrency slot.
- **Fail-Safe Pick:** If evaluation fails or times out, the manual **Pick button remains 100% active and unblocked**.

### 5. Web Cockpit Integration & Observability
- The web cockpit in `packages/web/src/routes/compare-variants.tsx` subscribes to the live event stream (`group:arbitration:completed`).
- The comparison header renders explicit lifecycle states: `pending`, `evaluating`, `completed`, and `failed/timed-out`.
- File polling is eliminated; the arrival of the event triggers a targeted query invalidation to fetch the structured JSON matrix.

## Consequences

- Tech leads gain a structured, 60-second comparison matrix across surviving variants.
- Worktrees can be safely cleaned up or deleted without causing `ENOENT` or broken evaluator reads.
- Zero race conditions: early picks cleanly abort evaluation and discard stale data.
- Full idempotency across server restarts and queued cancellations.
