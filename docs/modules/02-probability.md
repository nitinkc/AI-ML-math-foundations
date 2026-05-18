# Module 2: Probability Refresher for AI

## Plain-English first

When a model says intent A has 0.72 probability, it does **not** mean intent A is guaranteed true.
It means the model currently prefers A over alternatives based on learned patterns.

In operations, probabilities are confidence signals used for routing decisions.

## Minimal math second

For intents $i = 1..n$, probabilities should satisfy:

$$
\sum_{i=1}^{n} P(i) = 1, \quad 0 \le P(i) \le 1
$$

Useful expected-value formula for workload planning:

$$
\mathbb{E}[X] = \sum_{i=1}^{n} x_i P(i)
$$

Where $x_i$ is a numeric impact (for example, average handle minutes for intent $i$).

Decision this helps:

**How much total handling time should we expect from today's ticket mix?**

## IT scenario: Intent routing

Suppose a ticket gets:

- `account_unlock`: 0.60
- `vpn_issue`: 0.25
- `security_incident`: 0.15

If your auto-route threshold is 0.80, this ticket should **not** be auto-routed.
It should go to a clarification step or a human queue.

## Notebook third

Run `notebooks/math-foundations/02_probability.ipynb` to:

- Normalize raw scores into probabilities
- Check probability sums and invalid cases
- Compute expected handling time from a probability mix

## Pitfall and anti-pitfall

- Pitfall: treating probabilities as truth labels
- Anti-pitfall: combine probabilities with policy thresholds and fallbacks

## Quick checklist

- Do your class probabilities always sum to 1?
- Is your threshold policy explicit and documented?
- Are ambiguous cases routed safely?
- Do you monitor confidence drift week to week?

--8<-- "_abbreviations.md"
