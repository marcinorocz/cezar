# Single-Loop Learning Journal

Use this skill whenever a task encounters repeated test failures, exceeds budget, or is rejected during review, to record a permanent post-mortem entry in `.ai/cezar/learnings.jsonl` or `docs/learnings.md`.

## Logging Format

Append a single, structured entry:
```json
{
  "timestamp": "ISO-8601",
  "task_id": "cez-<id>",
  "root_cause": "Concise summary of failure root cause",
  "remedy": "Concrete rule or prompt adjustment preventing recurrence",
  "affected_files": ["src/..."]
}
```

## Retrieval
At the start of new tasks, check existing learnings in `.ai/cezar/learnings.jsonl` for relevant patterns affecting the target files before generating code.
