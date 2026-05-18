# Module 12: Evaluation in Production

## Plain-English first

Offline metrics are necessary but not sufficient.
Production evaluation asks: is the system still delivering value safely over time?

For IT copilots, focus on measurable outcomes:

- resolution time
- escalation accuracy
- analyst override rate
- user satisfaction trend

## Minimal math second

Track weighted portfolio quality:

$$
KPI_{weighted}=\sum_{i=1}^{n} w_i m_i
$$

- $m_i$: normalized metric score (for example precision, SLA hit-rate)
- $w_i$: business weight for that metric

Decision this supports:

**Can we safely expand rollout, or should we pause and remediate based on KPI movement?**

## IT scenario: Phased rollout gate

At 10% traffic, KPIs are stable.
At 50% traffic, override rate and incident miss-rate worsen.

A weighted KPI gate can trigger hold-and-fix before full rollout.

## Notebook third

Run `notebooks/math-foundations/12_eval_in_production.ipynb` to:

- aggregate multiple metrics into a weighted KPI score
- compare baseline vs new model over weekly windows
- simulate go / hold / rollback gating rules

## Pitfall and anti-pitfall

- Pitfall: optimizing one metric while harming operations elsewhere
- Anti-pitfall: evaluate with a balanced KPI set tied to business outcomes

## Quick checklist

- Are rollout gates defined before release?
- Are KPIs monitored by segment and risk class?
- Is drift monitored with alert thresholds?
- Is rollback criteria explicit and tested?

## From math to decision

Evaluation metrics become governance controls only when they are tied to explicit rollout and rollback actions.

--8<-- "_abbreviations.md"
