---
name: l0-preflight-guard
description: Pre-flight security & privacy gate ensuring secrets, PII, and unverified execution claims are safely handled.
---

# L0 Pre-Flight Security & Privacy Guard

Use this skill before dispatching prompts or committing changes to ensure total protection of sensitive assets, credential isolation, and verification honesty.

## 1. Secrets & Privacy Guard (Safe Local Inspection)
- **Local Content Scan:** Do not rely solely on `git status` (which returns only filenames and status). Perform a content scan of candidate files (staged, unstaged, untracked).
- **Non-Leaking Reporting:** When a secret or PII pattern is detected, **never print the matched token value** into the conversation or prompt log! Only report the file location and pattern category:
  `[BLOCKED: AWS_KEY detected in config/settings.py:14]`
- **Pattern Alignment:** Uses the canonical credential patterns (`sk-*`, `ghp_*`, `github_pat_`, `AKIA*`, `AIza*`, `glpat-*`).
- **Escape Hatch:** Synthetic test fixtures testing redaction should be annotated with `// cezar-allow-mock-token` or run under `CEZ_ALLOW_SYNTHETIC_FIXTURES=1`.

## 2. Pre-Prompt Sanitization & Truth Table
To guarantee deterministic security without silent, destructive masking, the guard enforces the following truth table:

| `CEZ_REDACT_SECRETS` | `CEZ_REDACT_PII` | PII Match | Secret Match | Gate Action |
|:---:|:---:|:---:|:---:|:---|
| **1** | **0** | No | Yes | **REJECT** (Secret in prompt) |
| **1** | **1** | Yes | No | **REJECT** (PII in prompt) |
| **1** | **1** | Yes | Yes | **REJECT** (Multiple violations) |
| **0** | **0** | - | - | **ACCEPT** (No pre-prompt scan) |
| **1** | **0** | - | No | **ACCEPT** (Secrets scrubbed) |

**Auxiliary Coverage:** Applies equally to main task dispatch and auxiliary calls (`workflows/run.ts` auto-naming). A rejected task never leaks to external APIs for title generation.

## 3. Scope Bounding
Verify that `git diff --stat` does not introduce unintended modifications to unrelated root configuration files or dependencies.
