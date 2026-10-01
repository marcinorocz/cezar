---
name: l0-preflight-guard
description: Pre-flight security & privacy gate ensuring deterministic regex sanitization, strict rule precedence, external allowlists, and credential isolation.
---

# L0 Pre-Flight Security & Privacy Guard

Use this skill before dispatching prompts or committing changes to ensure total protection of sensitive assets, credential isolation, and verification honesty.

## 1. Core Architectural Invariant: 100% Deterministic Regex Gate (No LLM Classifier)
- **Zero LLM Delegation:** PII and credential sanitization must be 100% deterministic (code, AST, and regex rule-based). Classification must NEVER be delegated to a language model — LLMs are non-deterministic, untestable against truth tables, and cannot guarantee cryptographic safety boundaries.
- **Adversarial Normalization:** Before pattern matching, text undergoes Unicode NFKC normalization to collapse homoglyphs and unescape obvious URL/percent-encoded payloads.

## 2. Strict Rule Precedence Hierarchy
When evaluating text, rules execute in rigid hierarchical order. A higher-priority match immediately determines or short-circuits the action:

```
[CRED-01: Raw Credentials]  --> ALWAYS BLOCK (No escape hatch ever)
       │ (clean)
       ▼
[PII-01: High-Risk Identifiers (PESEL, SSN, Credit Card)] --> REDACT or BLOCK
       │ (clean or unblocked)
       ▼
[HOST-01: Benign Git / Author Whitelist] --> PASS (Preserve git metadata)
       │ (unmatched)
       ▼
[PII-02: Contextual PII (Email, Phone)] --> REDACT (or PASS_LOGGED if valid hatch)
       │ (clean)
       ▼
[ALLOW-01: External Allowlist & Synthetic Fixture Hatch] --> PASS_LOGGED (Audited)
```

## 3. Pre-Prompt Sanitization & Truth Table
The guard is a pure function: `(text, policy, hatch?) -> Decision`. Fail-closed: any detector error results in immediate `block`.

| Credential Match | Strict PII Match | Contextual PII Match | Valid External Hatch | Gate Action | Result Type |
|:---:|:---:|:---:|:---:|:---:|:---|
| **Yes** | Any | Any | Any | `block` | Hard rejection (`reasonCode: 'credential'`) |
| **No** | **Yes** | Any | No | `redact` / `block` | Replace with stable token or block |
| **No** | **No** | **Yes** | No | `redact` | Replaced with stable placeholders (e.g. `[PII:EMAIL:1]`) |
| **No** | **No** | **Yes** | Yes (`allowPii` + reason) | `pass_logged` | Original text passed; audit count logged without raw values |
| **No** | **No** | **No** | N/A | `pass` | Clean original text sent |
| **Detector Error** | Any | Any | Any | `block` | Fail-closed (`reasonCode: 'detector_error'`) |

### Stable Placeholders & Git Identity Whitelist
- **Predictable Tokens:** PII masking generates deterministic tokens (e.g. `[PII:EMAIL:1]`, `[PII:PHONE:1]`), preserving structural syntax for coding agents without corrupting semantics.
- **HOST-01 Benign Whitelist:** Standard benign email metadata (`git commit --author`, `Co-Authored-By:`, `package.json`, `CODEOWNERS`) is whitelisted from PII blocking.

### Secure Escape Hatch (External Auth Only)
- **Prompt Injection Defense:** Escape hatches can NEVER be triggered solely by inline prompt instructions (e.g. an agent writing "ignore PII filter"). They require external authorization (environment flag `CEZ_ALLOW_SYNTHETIC_FIXTURES=1`, signed run parameter, or explicit UI composer toggle) accompanied by an audit reason and destination allowlist.

### Pure Function Interface (TypeScript)
```ts
export type PreflightDecision =
  | { action: 'pass' }
  | { action: 'redact'; text: string; findings: readonly Finding[] }
  | { action: 'pass_logged'; findings: readonly Finding[]; reason: string }
  | { action: 'block'; reasonCode: 'credential' | 'detector_error' | 'strict_pii' };

export function preflight(text: string, policy: Policy, hatch?: ExternalHatch): PreflightDecision {
  let f: Findings;
  try {
    const normalized = normalizeAdversarial(text);
    f = detectDeterministic(normalized, policy);
  } catch {
    return { action: 'block', reasonCode: 'detector_error' };
  }
  // CRED-01: Absolute precedence
  if (f.credentials.length > 0) return { action: 'block', reasonCode: 'credential' };
  // PII-01 & PII-02
  if (f.pii.length === 0) return { action: 'pass' };
  if (hatch && isExternalHatchAuthorized(hatch, policy)) {
    return { action: 'pass_logged', findings: f.pii, reason: hatch.reason };
  }
  return { action: 'redact', text: redactDeterministic(text, f.pii), findings: f.pii };
}
```

## 4. First-Turn Task Ingestion & Auxiliary Leaks
- Protect both the main task prompt and auxiliary runner calls:
  - `workflows/run.ts` auto-naming and `runs/auto-name.ts`: Do not dispatch unredacted text to LLM-based title generators. Fall back to deterministic slug `task-<id8>` if sensitive content is present.

## 5. Verification Honesty
- Clearly differentiate:
  - `[EXECUTED]`: Commands that genuinely ran on disk with verifiable exit code 0 and captured output in the transcript.
  - `[REASONED]`: Analytical deduction or code inspection. Never claim test execution without verifiable execution logs.
