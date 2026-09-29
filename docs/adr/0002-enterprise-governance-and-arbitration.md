# RFC: Enterprise Governance, Pre-Flight Data Guardrails & 4-Act Decision Synthesis for Cezar

**Proposers:** Marcin Orocz Ecosystem Team  
**Target Project:** [open-mercato/cezar](https://github.com/open-mercato/cezar)  
**Status:** Proposal / Contribution RFC  
**Date:** 2026-09-29  

---

## 1. Executive Summary

Cezar is an outstanding agentic development orchestrator, offering seamless git-worktree isolation, queue management, and a clean web cockpit.

In our production multi-agent collaboration ecosystem, we identified **4 critical enterprise governance gaps** that arise when running autonomous coding agents at scale:

1. **Pre-flight Privacy & Secrets Guardrails:** Agents currently run with unrestricted access to repositories; sensitive keys, tokens, customer PII, and NDA-bound assets risk leaking to cloud LLM providers without a deterministic pre-flight check.
2. **Cognitive Overhead in Variant Review:** Cezar's *Variants (×2 / ×3)* feature is brilliant, but reviewing raw side-by-side git diffs places high cognitive load on product managers and decision-makers.
3. **Loss of Failure Memory (Single-Loop Learning):** When a run fails after bounded retries, the lessons and root causes are lost once the task is closed, causing future runs to repeat the same mistakes.
4. **Skill Drift & Regression Gate:** Editing skills lacks regression safeguards to ensure that prompt tuning doesn't break baseline accuracy.

This RFC proposes introducing modular enterprise extensions and workflow conventions to Cezar to address these gaps.

---

## 2. Proposed Contributions

### Component 1: Pre-Flight L0 Security & PII Sanitization Hook
* **Problem:** Developers running `claude` or `codex` on customer codebases may inadvertently pass private customer data or `.env` credentials to third-party model APIs.
* **Proposal:** A lightweight pre-flight hook (`cezar pre-task`) that runs deterministic regex and optional local SLM (e.g. Ollama/Qwen) sanitization before task dispatch.
* **Behavior:**
  - Blocks execution if unmasked API keys or private tokens are found.
  - Automatically redacts sensitive identifiers into neutral placeholders (`[REDACTED_CLIENT_ID]`).

### Component 2: 4-Act Decision Synthesis for Variants (Automated Arbitration)
* **Problem:** Comparing 2 or 3 large git diffs requires 15–30 minutes of developer time.
* **Proposal:** An automated *Arbitration Synthesizer* step in the workflow that generates an executive **4-Act Decision Card**:
  - **Act 1 (Scenario):** User story & affected component.
  - **Act 2 (Value Risk):** Technical debt vs implementation cost comparison.
  - **Act 3 (Comparative Matrix):** Variant A vs Variant B (trade-offs, breaking surface, dependencies).
  - **Act 4 (Actionable Recommendation):** 2-sentence recommendation with expected KPI impact.
* **Result:** Decision-makers can choose the winning variant in under 60 seconds.

### Component 3: Single-Loop Learning Journal (`learnings.jsonl` / `learnings.md`)
* **Problem:** Cezar has `todos.json` for forward-looking tasks, but lacks backward-looking error memory.
* **Proposal:** When a task exhausts its `onFail.max` retries or is rejected at the review gate, Cezar automatically appends an entry to `.ai/cezar/learnings.jsonl`:
  ```json
  {
    "timestamp": "2026-09-29T08:00:00Z",
    "task_id": "cez-a1b2c3d4",
    "root_cause": "Import cycle between module A and module B",
    "remedy": "Separate interface types into shared types/ package",
    "runner": "claude"
  }
  ```
  Future tasks running in that repository automatically receive recent relevant learnings in their system prompt context.

### Component 4: Eval-Driven Regression Gate (Golden Eval Benchmark)
* **Problem:** Tuning `.ai/skills/*.md` can improve one task while degrading performance across other workflows.
* **Proposal:** Support for `.ai/evals/golden_dataset.json` with a 0% regression threshold on critical test paths before skill changes are committed.

---

## 3. Included Ready-to-Use Artifacts

As part of this contribution package, we provide:
1. `skills/l0-preflight-guard.md`: Agent playbook for privacy audit & verification honesty.
2. `skills/decision-synthesizer.md`: Agent playbook for 4-Act Decision Synthesis.
3. `skills/single-loop-learning.md`: Playbook for logging structured post-mortems.
4. `workflows/variants-arbitration.yaml`: Workflow executing parallel implementation and arbitration synthesis.

---

## 4. Discussion & Feedback

We welcome feedback from the Open Mercato and Cezar maintainers. If aligned with the project roadmap, we are prepared to open modular Pull Requests implementing these features natively.
