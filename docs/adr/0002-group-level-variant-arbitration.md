# Group-Level Variant Arbitration & 4-Act Decision Synthesis

## Context

Cezar provides parallel task execution via `variants: 1..3` on `POST /runs` (`.ai/specs/010-parallel-variants.md`). This spawns N independent sibling tasks in isolated worktrees grouped by `groupId`, which are compared side-by-side in `packages/web/src/routes/compare-variants.tsx`.

Line 45 of `.ai/specs/010-parallel-variants.md` explicitly reserved this capability for v2:
> *"Auto-ocena wariantów przez AI-sędziego (kusi, ale to v2 — najpierw człowiek)"*

Now that human review of variants is battle-tested, tech leads face high cognitive load comparing raw multi-file diffs. Furthermore, `packages/cezar/src/server/server.ts:5000` synchronously deletes losing worktrees and branches upon picking a winner. Therefore, any comparative arbitration must occur while the group worktrees remain intact.

## Decision

### 1. Trigger Predicate & Terminal Statuses
The hook hooks into `RunManager.dropActive()` (`packages/cezar/src/workflows/run.ts:1772`) where runs settle.
- **Terminal Set:** Matches `dispatch/engine.ts`: `TERMINAL_STATUSES = ['done', 'review', 'failed', 'cancelled']`.
- **Viability Guard:** Synthesis triggers **only when all siblings in the `groupId` are in `TERMINAL_STATUSES`, AND at least 2 siblings have reached `done` or `review`** (meaning at least 2 variants survived with live worktrees).
- **Mixed Groups:** If only one sibling survives (e.g. A `done`, B `failed`, C `cancelled`), arbitration is **skipped**: no redundant evaluation runs when there is nothing to compare.

### 2. State & Artifact Storage (Structured Data)
To avoid schema migrations on `runs.json` or polluting individual sibling records:
- The card is persisted as `.ai/cezar/groups/<groupId>/synthesis.json` (registered in `DATA_GITIGNORE_ENTRIES`).
- Served via `GET /api/v1/groups/:groupId/synthesis`.
- Stored as **structured data in `packages/contract`** rather than a raw Markdown string:
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
  createdAt: z.string(),
});
```

### 3. Evaluator Lifecycle, Budget & Safety
- **Strictly Opt-In:** Config-gated and composer-selectable (`--arbitrate` flag; default **off**) to avoid unexpected charges.
- **Out-of-Band Execution:** Executes as a background helper task that does **not** appear in the task table as an extra sibling, avoiding corruption of `groupRuns`.
- **Worktree:** Reads sibling worktrees (`.ai/cezar/worktrees/<runId>`) in read-only mode from the repository root.
- **Fail-Safe Pick:** If the evaluator fails, errors, or times out, the **Pick button remains 100% active and unblocked**. The UI displays a subtle note (`"Synthesis unavailable — pick manually"`).

### 4. Grounding of Act 4 (Diffs + Transcripts, No Re-Execution)
- Grounded in **actual diffs (`git diff --stat`) and sibling execution transcripts** (evaluating exit codes and test outcomes already produced during the runs).
- Does **not** re-run test suites inside the evaluator to prevent double execution overhead and timeouts.

### 5. Web Cockpit Integration & UI Wakeup
- Rendered as an interactive collapsible card inside `packages/web/src/routes/compare-variants.tsx` above the diff comparison columns.
- UI invalidates its group query upon member terminal transition; if synthesis is pending, it polls/listens for the `synthesis.json` arrival before rendering the structured matrix.

## Consequences

- Tech leads gain a structured, 60-second comparison matrix across surviving variants.
- Worktrees and branches are analyzed safely before synchronous deletion on pick.
- Zero unexpected billing: evaluation is opt-in and fail-safe.
