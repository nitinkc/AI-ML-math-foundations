# Module 11: Guardrails and Thresholds

## Plain-English first

Guardrails convert model confidence into actions.
Without thresholds, teams either over-automate risky cases or over-escalate simple ones.

The goal is not perfect prediction.
The goal is controlled automation with safe fallback behavior.

## Minimal math second

Simple three-band policy over confidence $p$:

$$
	ext{action}(p)=
\begin{cases}
	ext{auto}, & p \ge \tau_{high} \\
	ext{human-review}, & \tau_{low} < p < \tau_{high} \\
	ext{abstain}, & p \le \tau_{low}
\end{cases}
$$

Decision this supports:

**When should the system auto-act, ask for review, or abstain entirely?**

## IT scenario: Auto-close policy

For low-risk tickets, high-confidence outputs may auto-close.
For security-sensitive queues, route middle-confidence cases to analysts and abstain on low-confidence cases.

Use separate thresholds by risk class, not one global value.

## Notebook third

Run `notebooks/math-foundations/11_guardrails_thresholds.ipynb` to:

- simulate threshold bands over ticket confidence scores
- measure auto-rate, review-rate, and abstain-rate
- compare strict vs relaxed guardrail policies

## Pitfall and anti-pitfall

- Pitfall: tuning only for automation rate and ignoring failure cost
- Anti-pitfall: tune thresholds against business impact and incident risk

## Quick checklist

- Are thresholds defined per risk class?
- Are fallback paths implemented and tested?
- Is abstain behavior logged and monitored?
- Are threshold changes approved through governance review?

## From math to decision

Thresholds operationalize risk tolerance; they should be reviewed like access controls or alert severity policies.

--8<-- "_abbreviations.md"
