---
name: single-loop-learning
description: Appends verified failures and fixes to a bounded local learnings log to prevent repeat errors.
---

# Single-Loop Learning Memory

Whenever a step fails after exhausting retries (`onFail.max`), or when a human reviewer rejects a variant:

1. Identify the **Root Cause** (e.g. missing dependency, unconfigured env var, breaking API change).
2. Format a sanitized one-line JSON entry for `.ai/cezar/learnings.jsonl`:
```json
{"timestamp": "ISO-8601", "module": "path/or/feature", "trigger": "what caused the failure", "fix_rule": "rule to prevent recurrence"}
```
3. Ensure no unscrubbed credentials or PII exist in the entry before appending.
4. Keep file bounded to the 50 most recent records.
