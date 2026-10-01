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
To eliminate TOCTOU (time-of-check to time-of-use) race conditions when a user clicks **"Pick Winner"** while evaluation is running:
- **Atomic Write-Once Compilation:**
  - Before dispatching the evaluator, the engine compiles the evidence tree inside a temporary directory (`.ai/cezar/tmp/<groupId>-<snapshotId>`) and finalizes it via a single atomic `fs.rename` to `.ai/cezar/groups/<groupId>/evidence/`.
- **Manifest as Single Source of Truth (`manifest.jsonl`):**
  - Alongside bounded diffs, the snapshot includes `manifest.jsonl` recording each file entry:
    `{"path": string, "sha256": string, "size": number, "mtime": number, "timestamp": string, "runId": string}`.
  - All comparative and verification evaluation reads execute strictly against this immutable manifest and snapshot payload, NEVER against live worktree paths.
- **Hard Drift Abort (`EVIDENCE_DRIFT`):**
  - If any evaluator step encounters a hash mismatch or missing file against `manifest.jsonl`, arbitration immediately halts with exit code `EVIDENCE_DRIFT` rather than silently continuing with corrupted state.
- **Snapshot Security & Resource Hygiene:**
  - **Symlink Containment:** Symlink traversal outside the target repository worktree is strictly rejected to prevent path-traversal vulnerabilities.
  - **Resource Bounds:** Hard bounds enforced: max 300 diff lines per file, max 10MB per snapshot, and max 50 touched files.
  - **Retention Sync:** Evidence snapshots share the exact lifecycle of the parent `groupId` runs and are reaped automatically by `runs/retention.ts`.
- **Tool-Free Evaluator:** The evaluator agent runs in **tool-free, read-only mode**, receiving the snapshot payload directly in its prompt rather than mounting or inspecting live worktrees.
- **Early-Pick / Stale Result Disposal:** If a user clicks Pick before evaluation completes, the evaluator is cancelled via `AbortController`, and any late-arriving synthesis is discarded if its `input_revision_id` does not match the active state.

### 3. Multi-Agent Synthesis & Objection Governance
- **Blocking Objections (Safety Veto):** If an evaluator or tool raises a security, privacy, or data-integrity defect, it **cannot be outvoted** by trade-off metrics. The card surfaces an immediate `SAFETY_ESCALATION` requiring explicit lead override.
- **Non-Blocking Trade-offs:** Stylistic or optional architecture divergences are represented in the Act 3 trade-off matrix without blocking winner selection.
- **Traceable Attribution:** Every finding links to its specific transcript step and `manifestSha256`.

### 4. State & Artifact Storage (Structured Data)
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
    winningRunId: z.string().nullable(),
    isEscalated: z.boolean().default(false),
    rationale: z.string(),
    verificationSummary: z.string(),
  }),
  inputRevisionId: z.string(),
  manifestSha256: z.string(),
  createdAt: z.string(),
});
```

### 5. Background Evaluator Resource & Safety Contract
- **Strictly Opt-In:** Gated by an explicit `--arbitrate` flag or composer checkbox; default **off**.
- **Execution Budget:** Uses a fast, cost-effective model (inheriting runner profile or defaulting to a lightweight evaluator).
- **Execution Limits:** Hard timeout of 60 seconds. Does not consume a standard interactive concurrency slot.
- **Fail-Safe Pick:** If evaluation fails or times out, the manual **Pick button remains 100% active and unblocked**.

### 6. Web Cockpit Integration & Observability
- The web cockpit in `packages/web/src/routes/compare-variants.tsx` subscribes to the live event stream (`group:arbitration:completed`).
- The comparison header renders explicit lifecycle states: `pending`, `evaluating`, `completed`, and `failed/timed-out`.
- File polling is eliminated; the arrival of the event triggers a targeted query invalidation to fetch the structured JSON matrix.

## Alternatives Considered

1. **Comparing via `mtime` and live files:**
   - *Rejected:* File modification timestamps are altered unpredictably by git checkouts, branch switches, and local linters. Additionally, `server.ts:5000` deletes losing worktrees immediately on Pick, triggering fatal `ENOENT` crashes in downstream evaluators.
2. **File watching / `inotify`:**
   - *Rejected:* Not portable across Linux, macOS, and containerized Docker environments; introduces race windows between watch registration and write completion.
3. **Filesystem File Locks (`flock`):**
   - *Rejected:* Fragile during runner crashes or unexpected daemon restarts; stale locks leave worktree cleanup permanently blocked. Atomic snapshotting with write-once directory rename avoids locks entirely.

## Consequences

- Tech leads gain a structured, 60-second comparison matrix across surviving variants.
- Worktrees can be safely cleaned up or deleted without causing `ENOENT` or broken evaluator reads.
- Zero race conditions: early picks cleanly abort evaluation and discard stale data.
- Full idempotency across server restarts and queued cancellations.
