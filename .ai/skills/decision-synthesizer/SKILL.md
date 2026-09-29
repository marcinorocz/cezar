---
name: decision-synthesizer
description: Synthesizes diffs from parallel variants (variant_a vs variant_b) into an executive 4-Act Decision Card for reviewers.
---

# 4-Act Decision Synthesizer

When multiple variants are evaluated across sibling worktrees, compare the active worktree branches and synthesize an executive 4-Act Decision Card:

### Act 1: The Specific Scenario
Identify the concrete trigger, impacted modules, and problem statement addressed by the variants.

### Act 2: Value at Risk / Cost of Inaction
Highlight what breaks, degrades, or incurs unnecessary maintenance overhead if a suboptimal architecture is chosen.

### Act 3: Trade-off Matrix
Compare the variants across key dimensions:
| Dimension | Variant A | Variant B |
| :--- | :--- | :--- |
| **Cognitive Load & Readability** | Low / Medium / High | Low / Medium / High |
| **Test Coverage & Regression Risk** | ... | ... |
| **Architectural Extensibility** | ... | ... |
| **Performance Impact** | ... | ... |

### Act 4: Unambiguous Recommendation & Measurable Verification
- **Recommended Variant:** [Variant A or B]
- **Key Reason:** Decisive argument for why this variant wins.
- **Verification Metric:** Deterministic test or metric proving correctness.
