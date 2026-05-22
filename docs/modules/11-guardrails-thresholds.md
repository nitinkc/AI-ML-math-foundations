# Module 11: Guardrails and Thresholds

## What's the big idea?

Your model generates a probability: 0.72 for intent A.

Now what? 

**Guardrails** turn that number into an *action*. They're like traffic lights:

- 🟢 Green: AUTO (system handles it)
- 🟡 Yellow: REVIEW (human takes a look)
- 🔴 Red: ABSTAIN (don't touch it, escalate)

Without guardrails, teams either auto-route everything and fail spectacularly, or escalate everything and burn out analysts.

With guardrails, you get controlled automation with safe fallback.

## The math in plain terms

Define three confidence bands:

$$
\text{action}(p)=
\begin{cases}
\text{auto}, & p \ge \tau_{high} \\
\text{human-review}, & \tau_{low} < p < \tau_{high} \\
\text{abstain}, & p \le \tau_{low}
\end{cases}
$$

- $\tau_{high}$ is your high confidence boundary
- $\tau_{low}$ is your low confidence boundary
- Everything in between goes to a human

## Real-world scenario: Auto-close policy

You handle three ticket categories with different risk profiles:

**Low-risk (e.g., "how do I reset password?"):**
- Auto-close if confidence ≥ 0.90
- Human review if confidence 0.70–0.90
- Abstain if confidence < 0.70

**High-risk (e.g., "potential security breach"):**
- Auto-close if confidence ≥ 0.98 (super high bar)
- Human review if confidence 0.60–0.98 (lower threshold, more caution)
- Abstain if confidence < 0.60

Different thresholds, different risk levels. Makes sense.

## How to try this

Run `notebooks/math-foundations/11_guardrails_thresholds.ipynb` and:

- Simulate different threshold bands on real confidence scores
- Measure auto-rate, review-rate, abstain-rate
- Compare outcomes under strict vs relaxed policies

## The traps to avoid

❌ **The trap:** Tune only for "automation rate" (move more tickets automatically)  
✅ **The smart move:** Tune for *business impact*: quality, SLA, incidents, analyst load

## Checklist for you

- [ ] Are thresholds defined per risk class, not globally?
- [ ] Are fallback paths actually implemented and tested?
- [ ] Is abstention logged and monitored?
- [ ] Do threshold changes go through a review/approval process?

--8<-- "_abbreviations.md"
