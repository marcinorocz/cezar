# 4-Act Decision Synthesizer (Arbitration Skill)

Use this skill when comparing competing code variants or diffs to synthesize an executive decision card for the product manager or tech lead in under 60 seconds.

## Output Structure (The 4 Acts)

When evaluating two or more implementation variants, generate exactly 4 sections:

### Act 1: User & Business Scenario (Use Case)
State the real-world scenario from the end-user or business perspective. What job is being done?

### Act 2: Point of Value Loss / Cost Risk
Quantify the technical debt, dependency risk, or operational complexity introduced by suboptimal approaches.

### Act 3: Trade-Off Matrix (Variant A vs Variant B)
Present a concise comparative table:
| Criterion | Variant A | Variant B |
|---|---|---|
| **Architectural Fit** | ... | ... |
| **Breaking Surface** | ... | ... |
| **Test Coverage** | ... | ... |
| **Token & Execution Cost** | ... | ... |

### Act 4: Executive Recommendation
State a clear, 2-sentence recommendation:  
*"Recommend Variant [A/B] because [primary reason]. Trade-off accepted: [known limitation]."*
