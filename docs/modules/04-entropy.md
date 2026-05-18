# Module 4: Entropy and Uncertainty

## Plain-English first

Two predictions can share the same top class but differ in uncertainty.

- Distribution A: `[0.92, 0.05, 0.03]` is confident.
- Distribution B: `[0.45, 0.33, 0.22]` is uncertain.

[Entropy] quantifies this spread.
Higher entropy means the model is less sure and human review may be safer.

## Minimal math second

For a discrete distribution $P(i)$ over $n$ classes:

$$
H(P) = -\sum_{i=1}^{n} P(i) \log_2 P(i)
$$

- Minimum entropy: 0 (all probability on one class)
- Maximum entropy: $\log_2 n$ (uniform distribution)

Optional normalized score:

$$
H_{norm} = \frac{H(P)}{\log_2 n}
$$

Decision this supports:

**Should we auto-route this ticket, ask a clarifying question, or escalate to human triage?**

## IT scenario: Escalation safety

For security-sensitive queues, define a policy such as:

- Auto-route only if top probability >= 0.80 **and** normalized entropy <= 0.45
- Otherwise send to human triage

This reduces overconfident automation on ambiguous tickets.

## Notebook third

Run `notebooks/math-foundations/04_entropy.ipynb` to:

- Compute entropy for peaked and flat distributions
- Normalize entropy for different class counts
- Simulate a simple entropy-aware routing policy

## Pitfall and anti-pitfall

- Pitfall: using only top-1 probability and ignoring spread across alternatives
- Anti-pitfall: combine top-1 probability with entropy threshold in one policy

## Quick checklist

- Do you compute entropy on every classification output?
- Is your entropy threshold tuned with historical outcomes?
- Do high-entropy cases trigger safer handling paths?
- Are policy decisions auditable and explainable?

--8<-- "_abbreviations.md"
