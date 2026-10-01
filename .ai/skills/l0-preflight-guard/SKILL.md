---
name: l0-preflight-guard
description: Pre-flight security & privacy gate ensuring secrets, PII, and unverified execution claims are safely handled.
---

# L0 Pre-Flight Security & Privacy Guard

Use this skill before dispatching prompts or committing changes to ensure total protection of sensitive assets, credential isolation, and verification honesty.

## 1. Secrets & Privacy Guard (Safe Inspection Pattern)
- **Zero Raw Secrets in Prompts & Commits:** Scan staged, unstaged, and candidate context files for known token shapes (`sk-*`, `ghp_*`, `AKIA*`, `AIza*`, `glpat-*`).
- **Non-Leaking Reporting:** When a secret pattern is detected, **never print the matched token value** into the conversation or prompt log! Only report the file location and pattern category:
  `[BLOCKED: AWS_KEY detected in config/settings.py:14]`
- **Synthetic Test Fixture Escape Hatch:** Legitimate test fixtures testing redaction or token parsers should be marked with a local ignore comment (e.g. `// cezar-allow-mock-token`) to prevent false-positive rejection without disabling system-wide scrapers.

## 2. Pre-Prompt Sanitization & Deterministic Truth Table
The guard is a pure function: `(text, policy, hatch?) -> Decision`. Fail-closed: any detector error results in immediate `block`.

| Credential Match | PII Match | Valid Escape Hatch | Gate Action | Result Type |
|:---:|:---:|:---:|:---:|:---|
| **No** | **No** | N/A | `pass` | Original prompt sent |
| **No** | **Yes** | No | `redact` | Replaced with stable placeholders (e.g. `[PII:EMAIL:1]`) |
| **No** | **Yes** | Yes (`allowPii` + reason) | `pass_logged` | Pass original prompt; log audit count without values |
| **Yes** | **No** | Any | `block` | Hard rejection (`reasonCode: 'credential'`) |
| **Yes** | **Yes** | Any | `block` | Hard rejection (hatch never applies to credentials) |
| **Detector Error** | Any | Any | `block` | Fail-closed (`reasonCode: 'detector_error'`) |

### Stable Placeholders & Git Identity Whitelist
- PII masking generates deterministic tokens (e.g. `[PII:EMAIL:1]`, `[PII:PHONE:1]`), preserving structure for coding agents without destroying semantic flow.
- Standard benign email metadata (`git commit --author`, `Co-Authored-By:`, `package.json`, `CODEOWNERS`) is whitelisted from PII blocking.

### Pure Function Interface (TypeScript)
```ts
export type PreflightDecision =
  | { action: 'pass' }
  | { action: 'redact'; text: string; findings: readonly Finding[] }
  | { action: 'pass_logged'; findings: readonly Finding[]; reason: string }
  | { action: 'block'; reasonCode: 'credential' | 'detector_error' };

export function preflight(text: string, policy: Policy, hatch?: Hatch): PreflightDecision {
  let f: Findings;
  try { f = detect(text, policy); } catch { return { action: 'block', reasonCode: 'detector_error' }; }
  if (f.credentials.length) return { action: 'block', reasonCode: 'credential' };
  if (!f.pii.length) return { action: 'pass' };
  if (hatchValid(hatch, policy)) return { action: 'pass_logged', findings: f.pii, reason: hatch!.reason };
  return { action: 'redact', text: redact(text, f.pii), findings: f.pii };
}
```

## 3. First-Turn Task Ingestion & Auto-Naming Boundary
- Protect both the main task prompt and auxiliary LLM calls (e.g. `auto-name.ts`):
  - Do not pass raw task orders with unmasked PII to auto-naming runners. Fall back to deterministic rule-based task naming if PII is detected.

## 4. Verification Honesty
- Clearly differentiate:
  - `[EXECUTED]`: Commands that genuinely ran on disk with verifiable exit code 0 and captured tool-call output in the transcript.
  - `[REASONED]`: Analytical deduction or code inspection. Never attest test execution without actual execution logs.
