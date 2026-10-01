---
name: l0-preflight-guard
description: Pre-flight security & privacy gate with strict token placeholder redaction, deterministic truth table, and fail-closed credential blocking.
---

# L0 Pre-Flight Security & Privacy Guard

## 1. Problem Statement
When dispatching prompts to LLM vendor runners or committing changes, tools and developers risk leaking sensitive secrets, credentials, API keys, or customer Personal Identifiable Information (PII). Relying on model "good behavior" or fuzzy prompts to keep data confidential is unsafe. Sanitization must be an uncompromising, deterministic pre-flight filter operating prior to prompt departure.

---

## 2. Hard Security Policy & Invariants
- **Deterministic Code-Level Gate:** Sanitization is executed by deterministic parsers and regex patterns before network dispatch. Never delegate PII/secret classification to an LLM.
- **Fail-Closed on Credentials:** Any matched credential, session token, private key, or API key causes immediate prompt abortion (`block`). Credentials can NEVER be whitelisted or bypassed via an escape hatch.
- **Predictable Placeholder Masking:** Detected PII items are substituted with sequential tokens:
  - Emails: `EMAIL_001`, `EMAIL_002`, ...
  - Phone numbers: `PHONE_001`, ...
  - Names / SSN / National IDs: `ID_001`, ...
  - Generic Secrets / Tokens: `TOKEN_001`, `SECRET_001`, ...
- **Local Secret Mapping:** The mapping table (`EMAIL_001 -> real@corp.com`) is retained strictly in volatile local memory or ephemeral local storage. It is NEVER transmitted upstream, written to shared transcripts, or included in outgoing payloads.
- **Safe Escape Hatch:** Only explicit, user-configured allowlists with documented reasons and audit trails may exempt known non-secret fixtures. There is NO default "looks safe" bypass.

---

## 3. Decision Truth Table (When to Block / When to Allow)

| Payload Content Type | Example Pattern | Gate Action | Prompt State | Logging / Escalation |
|---|---|:---:|:---:|---|
| **Public / Safe Value** | Generic code, standard docs, open repo text | **ALLOW** | Original unchanged | No alert |
| **Benign Git / Dev Identity** | `git log --author`, `package.json` author, `CODEOWNERS` | **ALLOW** | Original unchanged | Whitelisted developer context |
| **Contextual PII** | Email addresses, phone numbers | **REDACT** | Replaced with `EMAIL_001`, `PHONE_001` | Map stored locally only |
| **High-Risk PII / National ID** | PESEL, SSN, Credit card numbers | **REDACT + WARN** | Replaced with `ID_001` | Log finding count in audit |
| **Credentials & Auth Tokens** | AWS Key (`AKIA*`), GitHub Token (`ghp_*`), OpenAI (`sk-*`), JWT, SSH private keys | **BLOCK** | **Aborted** (Prompt not sent) | Immediate rejection error: `reasonCode: 'credential'` |
| **Unknown Secret-Like String** | High-entropy hex/base64 blobs in sensitive fields | **BLOCK & ESCALATE** | **Aborted** (Prompt not sent) | Escalated to developer for explicit allowlisting |
| **Explicitly Whitelisted Fixture** | Test token in synthetic fixture with valid config flag | **ALLOW_AUDITED** | Original passed | Recorded in local audit log with reason tag |

---

## 4. Operational Checklist & Pseudocode

### Pre-Flight Checklist
- [ ] 1. Text normalized using Unicode NFKC (homoglyph attack prevention).
- [ ] 2. High-priority scan for raw credentials (`CRED-01`). If found -> **HALT IMMEDIATELY**.
- [ ] 3. Check for high-entropy secret-like strings. If unwhitelisted -> **HALT AND ESCALATE**.
- [ ] 4. Whitelist check for repository metadata (`git commit --author`, etc.).
- [ ] 5. Mask contextual PII into sequential tokens (`EMAIL_001`, `PHONE_001`).
- [ ] 6. Ensure token map is stored strictly in ephemeral local memory.
- [ ] 7. Dispatch sanitized prompt payload.

### Pure Function Implementation Pattern
```ts
export interface SanitizationResult {
  action: 'ALLOW' | 'REDACT' | 'BLOCK' | 'BLOCK_AND_ESCALATE';
  sanitizedPrompt?: string;
  localTokenMap?: Record<string, string>;
  reason?: string;
}

export function sanitizePreFlightPrompt(
  rawPrompt: string,
  allowlist: Set<string> = new Set()
): SanitizationResult {
  const normalized = rawPrompt.normalize('NFKC');

  // Step 1: Detect hard credentials (Zero-tolerance)
  const credentialHits = detectCredentials(normalized);
  if (credentialHits.length > 0) {
    return {
      action: 'BLOCK',
      reason: `Found raw credentials: ${credentialHits.map(h => h.type).join(', ')}. Dispatch aborted.`
    };
  }

  // Step 2: Detect unknown high-entropy secret-like strings
  const suspiciousBlobs = detectHighEntropyBlobs(normalized);
  const unwhitelistedBlobs = suspiciousBlobs.filter(b => !allowlist.has(b.hash));
  if (unwhitelistedBlobs.length > 0) {
    return {
      action: 'BLOCK_AND_ESCALATE',
      reason: 'Unknown secret-like tokens detected. Requires explicit allowlisting with justification.'
    };
  }

  // Step 3: Redact contextual PII into stable placeholders
  const tokenMap: Record<string, string> = {};
  let counter = 1;
  const sanitized = normalized.replace(PII_EMAIL_REGEX, (match) => {
    if (isGitIdentityContext(match, normalized)) return match;
    const placeholder = `EMAIL_${String(counter++).padStart(3, '0')}`;
    tokenMap[placeholder] = match;
    return placeholder;
  });

  return {
    action: counter > 1 ? 'REDACT' : 'ALLOW',
    sanitizedPrompt: sanitized,
    localTokenMap: tokenMap
  };
}
```
