# Module 3: Logits and Softmax

## Plain-English first

Models often produce [Logits] first.
Think of logits as unscaled preference scores, not probabilities.

Higher logit means "more preferred," but only relative to other options.
To get probabilities, apply [Softmax].

## Minimal math second

For logits $z_i$ over classes $i$:

$$
P(i) = \frac{e^{z_i}}{\sum_{j=1}^{n} e^{z_j}}
$$

Temperature-adjusted softmax:

$$
P_T(i) = \frac{e^{z_i/T}}{\sum_{j=1}^{n} e^{z_j/T}}, \quad T > 0
$$

- Lower $T$: sharper distribution (more deterministic)
- Higher $T$: flatter distribution (more exploratory)

Decision this supports:

**Do we prioritize stable routing (low temperature) or broader exploration (high temperature)?**

## IT scenario: Competing ticket intents

If logits are `[3.2, 2.8, 0.5]`, top two intents are close.
After softmax, the gap may still be too small for confident automation.

If logits are `[5.0, 1.2, -0.4]`, softmax is sharply peaked and auto-route is usually safer.

## Notebook third

Run `notebooks/math-foundations/03_logits_softmax.ipynb` to:

- Convert logits into probabilities step by step
- Compare low vs high temperature effects
- Compute top-1 confidence gaps for routing policy

## Pitfall and anti-pitfall

- Pitfall: comparing logits across unrelated requests as absolute confidence
- Anti-pitfall: compare relative probabilities and margin inside the same request

## Quick checklist

- Are you converting logits with numerically stable softmax?
- Do you test behavior under multiple temperature settings?
- Do you use confidence margin, not just top class label?
- Is your routing policy tied to measured outcomes?

--8<-- "_abbreviations.md"
