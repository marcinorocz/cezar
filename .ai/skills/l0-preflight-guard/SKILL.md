---
name: l0-preflight-guard
description: Pre-flight check ensuring no secrets leak, PII is sanitized, and git diff scope remains bounded before task completion.
---

# L0 Pre-flight Guard

Execute this verification step before committing changes or completing any task in Cezar:

## 1. Secrets & PII Check
Scan modified files (`git diff --staged` or `git status`) for:
- API Keys (`AIza[0-9A-Za-z_-]{35}`, `sk-[a-zA-Z0-9]{20,}`, `ghp_[a-zA-Z0-9]{36}`, `AKIA[0-9A-Z]{16}`)
- Hardcoded tokens in configuration or test fixtures
- Customer PII (unmasked emails, phone numbers, real personal records)

## 2. Scope Bounding
Verify that `git diff --stat` does not introduce unintended modifications to unrelated root configuration files or dependencies.
