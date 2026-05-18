# Module 8: Classification and Calibration

## Plain-English first

Classification predicts a category with a probability.
Calibration checks whether that probability matches reality over time.

If a model says "0.80 confidence" for many tickets,
about 80% of those should actually be correct.
If not, your thresholds may create risk.

## Minimal math second

Threshold decision rule:

$$
\hat{y} =
\begin{cases}
1, & p \ge \tau \\
0, & p < \tau
\end{cases}
$$

- $p$: predicted class probability
- $\tau$: decision threshold

Decision this supports:

**What confidence level should trigger auto-action vs human review?**

## IT scenario: Incident auto-escalation

For potential security tickets, you may set a lower threshold to reduce missed incidents.
For low-risk categories, you may set a higher threshold to avoid false alarms.

Calibration ensures those thresholds remain trustworthy after model drift.

## Notebook third

Run `notebooks/math-foundations/08_classification_calibration.ipynb` to:

- Evaluate threshold effects on precision and recall
- Build a simple reliability table (predicted vs observed rate)
- Compare policy outcomes under stricter and looser thresholds

## Pitfall and anti-pitfall

- Pitfall: using one global threshold for all risk classes
- Anti-pitfall: set class-specific thresholds and recalibrate periodically

## Quick checklist

- Is each threshold tied to business risk tolerance?
- Do you monitor precision/recall by class?
- Do calibration checks run on fresh production-like data?
- Are low-confidence outputs routed to safe fallback paths?

--8<-- "_abbreviations.md"
