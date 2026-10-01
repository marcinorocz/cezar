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
- **`reconcileGroupArbitration(groupId, trigger)`:** Called whenever any sibling transitions to a terminal state (`completed`, `failed`, `cancelled`) or during restart recovery.
- **Idempotency Guard & Deduplication Table:**
  Guarded by an idempotency key `group-arbitration:<groupId>:<revisionHash>`, where `revisionHash` is the canonical SHA-256 of sorted variant snapshot hashes. Tracked in `arbitration_runs(idempotency_key UNIQUE, status, result_card_id, created_at)` to eliminate duplicate paid evaluations across concurrent events, cancellations, or restart sweeps.
- **Viability Predicate:** The hook launches evaluation **only when all siblings in the `groupId` are terminal, AND at least 2 siblings have reached `completed` or `review`** (with valid code). If fewer than 2 viable candidates exist, arbitration transitions immediately to `skipped(insufficient_candidates)`.

### 2. Bounded Immutable Evidence Snapshot Pattern
Comparative arbitration evaluates extracted evidence rather than mounting live filesystem trees.
- **Snapshot-Before-Delete Invariant:**
  Evidence snapshots are captured via an idempotent function `snapshotStore.ensure(variantId)` as soon as each variant reaches a terminal state. As a defensive guard, both the human Pick handler and `runs/retention.ts` invoke `snapshotStore.ensure` prior to unlinking worktree directories. If a worktree is missing and no snapshot exists, arbitration safely skips with `skipped: evidence_unavailable`.
- **Atomic Assembly & Schema (`evidence/v1`):**
  Compiled into a staging buffer in `.tmp/` and finalized via an atomic `fs.rename` to `.ai/cezar/groups/<groupId>/evidence/<variantId>.json`:
  - `diffStat`: Files changed, insertions, deletions.
  - `unifiedDiff`: Bounded diff excerpt (max 500 lines, sorted by churn; omitted files tracked in `omittedFiles` with `truncated: true`).
  - `verification`: Exact test commands, `exitCode` (0/1), execution duration, and truncated output tail (max 80 lines / 8 KB).
  - `transcriptExcerpt`: Last 100 lines of agent log (tool call hashes, non-leaking).
  - `snapshotSha256`: Content-addressed SHA-256 digest (excluding timestamps).
- **Hard Drift Abort (`EVIDENCE_DRIFT`):**
  If any evaluator step detects a hash mismatch against the snapshot manifest, evaluation immediately aborts with `EVIDENCE_DRIFT` rather than operating on stale data.
- **L0 Pre-Flight Sanitization:**
  Snapshot text is scrubbed via L0 pre-flight guard prior to prompt dispatch. If raw credentials are found in the diff, arbitration is skipped (`skipped: credential_in_evidence`).

### 3. Race Resolution: Authoritative Human Pick vs AI Evaluator
Pick by the human reviewer is authoritative, non-blocking, and terminal. The AI judge is purely advisory.
- **Group State Machine:**
  ```
  GROUP_ACTIVE -> (all terminal) -> GROUP_PENDING_ARBITRATION
  GROUP_PENDING_ARBITRATION -> (judge finished) -> GROUP_ARBITRATED
  GROUP_PENDING_ARBITRATION -> (user Pick)      -> GROUP_PICKED   [judge aborted]
  GROUP_ARBITRATED          -> (user Pick)      -> GROUP_PICKED   [card archived as history]
  ```
- **Cooperative Abort + Compare-and-Swap (CAS):**
  When a user clicks Pick:
  1. An immediate `AbortController.abort('picked')` is sent to halt active model streaming.
  2. The group state transitions to `GROUP_PICKED` via an atomic CAS (`UPDATE ... WHERE state = 'PENDING_ARBITRATION'`).
  3. Worktree directories of losing variants are unlinked without waiting for the model.
  4. Any late-arriving evaluation result (e.g. inference completed in the race window) is rejected by CAS, marked `stale: true`, and archived in the audit log without overwriting the human decision.

### 4. Background Evaluator Resource & Safety Contract

| Parameter | Specification | Rationale |
|---|---|---|
| **Activation** | Strictly opt-in (workspace / `--arbitrate` flag) | Default off; zero surprise costs |
| **Runner** | Inherited from parent run (`runner: inherit`) | Avoids shadow infrastructure and extra daemon dependencies |
| **Model** | Lightweight Tier-2 reasoning class (`arbitration.model`, e.g. Haiku / Flash-mini) | Fast structural synthesis; requires structured output capability |
| **Timeout** | 45–60 seconds hard kill (`AbortSignal.timeout`) | Bounded latency; 95th percentile analysis completes under 25s |
| **Token Limits** | Bounded: Input ~12–15k tokens, Output max 1500–2000 tokens | Deterministic cost (~$0.008 per evaluation) |
| **Concurrency** | Low-priority slot / out-of-band semaphore (max 2 concurrent global) | Never starves interactive developer runs |
| **Tool Access** | `tool-free` (zero shell, filesystem, or network access) | Immune to prompt-injection tool exploits |
| **Deterministic Rule** | A variant with failing tests (`exitCode != 0`) cannot beat a green variant | Hard validation gate enforced in post-processing |
| **Fail-Safe Fallback** | Timeout / model error degrades to manual diff view | UI is never blocked; surfaces clear error status with retry option |

### 5. Multi-Agent Synthesis & Objection Governance
- **Blocking Objections (Safety Veto):** If an evaluator flags a security vulnerability, PII exposure, or data-loss invariant, **majority voting is forbidden**. The card flags a mandatory `SAFETY_ESCALATION` requiring explicit human override.
- **4-Act Structured Output Schema:**
  Persisted to disk as `.ai/cezar/groups/<groupId>/synthesis.json` and validated by Zod schema (`variantSynthesisSchema` in `packages/contract`).

### 6. Web Cockpit Integration & Observability
- `packages/web/src/routes/compare-variants.tsx` subscribes to `group:arbitration:completed`.
- Renders explicit lifecycle states: `pending`, `evaluating`, `completed`, and `failed/timed-out`.
- File polling is eliminated; event receipt invalidates cached query keys to fetch structured JSON.

## Alternatives Considered

1. **Comparing via `mtime` and live files:**
   - *Rejected:* File timestamps are altered unpredictably by git checkouts, branch switches, and local linters. Additionally, `server.ts:5000` deletes losing worktrees immediately on Pick, triggering fatal `ENOENT` crashes.
2. **File watching / `inotify`:**
   - *Rejected:* Not portable across Linux, macOS, and containerized Docker environments; introduces race windows between watch registration and write completion.
3. **Filesystem File Locks (`flock`):**
   - *Rejected:* Stale locks remain on disk during runner crashes or unexpected daemon restarts, permanently blocking cleanup. Atomic snapshotting with write-once directory rename avoids locks entirely.
4. **Separate "Shadow" Executor Pool:**
   - *Rejected:* Introducing an independent worker pool for arbitration complicates deployment and monitoring. Inheriting the parent runner with a low-priority queue slot is transparent and auditable.
5. **Blocking Pick until Evaluation Finishes:**
   - *Rejected:* Human decisions must never be held hostage by third-party LLM latency. Human pick is always authoritative and immediate.

## Consequences

- Tech leads receive a structured, 60-second comparison matrix across surviving variants.
- Worktrees can be safely unlinked or cleaned up without causing `ENOENT` or broken evaluator reads.
- Zero race conditions: early picks cleanly abort evaluation and discard or archive stale data.
- Full idempotency across server restarts, queue cancellations, and concurrent terminal transitions.
