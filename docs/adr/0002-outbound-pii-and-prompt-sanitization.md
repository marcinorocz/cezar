# First-Turn Task Ingestion Sanitization & Outbound PII Guard

- **Status:** Proposed
- **Relates to:** Issue #1156

## Context

Cezar implements inbound secret redaction (#427) via `packages/cezar/src/core/secret-redaction.ts` and `packages/cezar/src/runs/stream-redaction.ts`. This protects the web cockpit and persisted transcripts from storing credentials dumped by agent tool execution (`printenv`, etc.).

However, when composing tasks or attaching context briefs, users and integrations risk injecting raw credentials or customer PII into the initial prompt handed to vendor runners (`claude`, `codex`, `cursor`, `opencode`).

### Explicit Seam & Scope Boundaries
To remain architecturally honest:
- **In-Scope (First-Turn Ingestion Seam):** Sanitization covers text composed by Cezar and handed to a runner on turn 1: task descriptions, continuation context, attached prompt briefs, and system-prompt augmentations (`--append-system-prompt` or prepended prompt blocks).
- **Auxiliary Call Coverage:** Crucially covers auxiliary LLM calls: `workflows/run.ts:1301` auto-naming and `runs/auto-name.ts:94–100,163–165`. A task rejected at the main dispatch gate must not have already leaked PII or credentials via the naming prompt. If unredacted sensitive tokens are detected, auto-naming falls back to a deterministic slug (`task-<id8>`) without calling external APIs.
- **Out-of-Scope (Sub-process Tool Calls):** This hook does **not** act as an egress proxy or inspect subsequent vendor CLI tool executions (e.g. if an agent runs `cat .env` on turn 2+ inside the spawned vendor process). Those require vendor-specific hooks or an external egress proxy.

## Decision

### 1. Rejection Gate over Silent Destructive Redaction
Blindly replacing email addresses or identifiers with `[REDACTED]` in task prompts causes severe semantic degradation (e.g. `git log --author=alice@corp.com` becomes `git log --author=[REDACTED]`).
Therefore, the first-turn outbound guard functions as a **Validation / Rejection Gate**:
- When an actionable violation is detected, task dispatch fails closed with a clear error:
  `"Task contains raw credentials or PII; please remove before dispatch."`
- In the cockpit web composer, this surfaces as an inline validation error preventing queueing.

### 2. Guard Semantics & Flag Truth Table
The guard strictly adheres to the following truth table:

| `CEZ_REDACT_SECRETS` | `CEZ_REDACT_PII` | PII Match | Secret Match | Gate Action |
|:---:|:---:|:---:|:---:|:---|
| **1 (Default)** | **0 (Default)** | Ignored | Yes | **REJECT** (Secret in prompt) |
| **1** | **0** | Ignored | No | **ACCEPT** |
| **1** | **1** | Yes | No | **REJECT** (PII in prompt) |
| **1** | **1** | No | Yes | **REJECT** (Secret in prompt) |
| **1** | **1** | Yes | Yes | **REJECT** (Multiple violations) |
| **0** | **0** | Ignored | Ignored | **ACCEPT** (No pre-prompt scan) |
| **0** | **1** | Yes | Ignored | **REJECT** (PII scan active, secrets bypassed) |

- **One Token Pattern List:** The dispatch gate reuses the exact `TOKEN_PATTERNS` from `packages/cezar/src/core/secret-redaction.ts` so inbound scrubbing and outbound rejection cannot drift apart.
- **Git Identity Whitelist:** Standard benign email occurrences (`git commit --author`, `Co-Authored-By:`, `package.json` author, and `CODEOWNERS`) are whitelisted from triggering PII rejection.

### 3. Escape Hatch for Test Fixtures (Documented Limitation)
To avoid false-positive claims: synthetic token strings present in test fixtures (such as `secret-redaction.test.ts`) would naturally trigger rejection on tasks aimed at modifying them.
- Rather than coupling to `CEZ_REDACT_SECRETS=0` (which dangerously disables transcript scrubbing), a dedicated flag `CEZ_ALLOW_SYNTHETIC_FIXTURES=1` bypasses outbound dispatch rejection while keeping inbound transcript redaction fully intact.
- Prompts can also include local inline ignore pragmas (e.g. `// cezar-allow-mock-token`).

### 4. Attestation Separation
Report honesty and `[EXECUTED]` vs `[REASONED]` verification are orthogonal to credential sanitization and are tracked in proposal #1155.

## Consequences

- Task context cannot leak credentials or PII on turn 1 to cloud model APIs.
- Auxiliary naming calls are quarantined, eliminating pre-dispatch side-channel leaks.
- Prompt intent is preserved: task orders are not silently corrupted into unactionable text.
- Backwards compatibility: tasks containing legitimate fixture tokens have an explicit, documented escape hatch without weakening on-disk transcript safety.
