# Bounded Single-Loop Learning Log (learnings.jsonl)

## Context

When an agent fails a task after exhausting its bounded retries (`onFail.max: 2`), or when a run is rejected during human review, the failure context and root cause are lost upon task closure. As a result, subsequent runs touching the same modules often repeat the exact same errors (e.g. missing environment variables, deprecated import paths, unmocked network services).

However, introducing a new state file under `.ai/cezar/` is a high-blast-radius change governed by `SDLC.md` and `BACKWARD_COMPATIBILITY.md`:
1. **Repository Churn & Privacy:** An agent-written failure log must not cause uncommitted churn in the user's PR diffs or accidentally leak into public git trees.
2. **Credential Safety:** Error outputs and command failures frequently contain tokens, keys, and paths that must never land in state files unscrubbed (#427).
3. **Prompt Bloat & Stale Rules:** An unbounded log injected blindly into agent prompts risks cognitive degradation and hallucinated constraints when code has moved on.

## Decision

1. **State File Location & Gitignore:** Store failure logs in `.ai/cezar/learnings.jsonl`. Register `learnings.jsonl` and `learnings.jsonl.tmp` directly in `DATA_GITIGNORE_ENTRIES` in `data-gitignore.ts` so the file is ignored by default in the local repo and does not pollute `git status`.
2. **Mandatory Secret Redaction:** All entries appended to `learnings.jsonl` must be scrubbed through `redactSecrets()` prior to disk persistence, maintaining the `.ai/cezar/` "no secrets in state files" invariant.
3. **Bounded FIFO Ceiling:** Cap `learnings.jsonl` to a maximum of 50 recent entries. When the ceiling is reached, older entries are evicted automatically (FIFO).
4. **Scattered Module Matching & Decay:** When injecting past learnings into subsequent tasks:
   - Match only by directory/module prefix of the target files touched in the task.
   - Inject at most 3 most recent applicable rules into the agent prompt context.
   - Invalidate/prune rules when the associated unit test suite subsequently passes.

## Schema

Each line in `learnings.jsonl` is an independent JSON object:
```json
{
  "timestamp": "ISO-8601",
  "module": "path/to/component",
  "trigger": "Failing test or command (redacted)",
  "root_cause": "Brief architectural or syntax diagnosis",
  "fix_rule": "Actionable directive to avoid recurrence"
}
```

## Consequences

- Agents retain memory of recent failure modes, cutting down repeat retry cycles.
- Zero git diff pollution or accidental credential exposure.
- State file size and token consumption remain strictly bounded.
