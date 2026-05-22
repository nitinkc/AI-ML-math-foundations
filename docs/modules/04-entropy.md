# Module 4: Entropy and Uncertainty

## What's the big idea?

Imagine two probability predictions:

- **Distribution A:** `[0.92, 0.05, 0.03]` — super confident. Clear winner.
- **Distribution B:** `[0.45, 0.33, 0.22]` — all over the place. No clear winner.

They might both have the same "top prediction," but A is *way* more certain than B.

**[Entropy]** is a number that measures this uncertainty. High entropy = confused prediction. Low entropy = confident prediction.

In operations, high entropy usually means "ask a human" instead of auto-routing.

## The math in plain terms

Entropy formula:

$$
H(P) = -\sum_{i=1}^{n} P(i) \log_2 P(i)
$$

Here's what this tells you:

- **Entropy = 0:** All probability is on one class. Maximum confidence. (That first distribution A.)
- **Entropy = $\log_2 n$:** Probability is split evenly. Maximum confusion. (Imagine 50-50 split.)

To compare across different class counts, normalize it:

$$
H_{norm} = \frac{H(P)}{\log_2 n}
$$

Now your entropy score is always between 0 (certain) and 1 (completely lost).

## Real-world scenario: Escalation safety

Let's say you handle three ticket queues: low-risk, medium-risk, and high-risk (like security).

Your policy:
- **Auto-route** only if top probability ≥ 0.80 AND normalized entropy ≤ 0.45
- **Otherwise** send to human triage

This combo catches both of these bad cases:
1. High probability but high entropy (conflicting signals)
2. Moderate probability with high entropy (genuine confusion)

Fewer mistakes, fewer escalations. Safer production.

## How to try this

Run `notebooks/math-foundations/04_entropy.ipynb` and:

- Calculate entropy for different probability distributions
- See how entropy changes with different class counts
- Simulate a routing policy that uses entropy + confidence

## The traps to avoid

❌ **The trap:** Using only the top-1 probability for routing. (Misses the spread confusion.)  
✅ **The smart move:** Combine top probability (confidence) + entropy (clarity) in one rule

## Checklist for you

- [ ] Do you calculate entropy on every single classification output?
- [ ] Has someone tuned your entropy threshold using historical ticket outcomes?
- [ ] Do high-entropy tickets go to safer handling paths?
- [ ] Can you audit and explain your routing decisions?

--8<-- "_abbreviations.md"
