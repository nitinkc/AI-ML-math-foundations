# Module 9: Correlation vs Causation in AI Ops

## What's the big idea?

Here's the mistake people make constantly: **Two metrics go up together, so one must cause the other.**

Classic example: "Long prompts caused SLA breaches."

Maybe. Or maybe *complex incidents* cause both longer prompts AND longer resolution time. The prompt length is innocent.

This trap ruins production decisions. You change something based on correlation, but nothing actually improves. Worse, it might get worse.

## The math in plain terms

Correlation measures how two things move together:

$$
r = \frac{\sum_i (x_i-\bar{x})(y_i-\bar{y})}{\sqrt{\sum_i (x_i-\bar{x})^2}\sqrt{\sum_i (y_i-\bar{y})^2}}
$$

- $r = 1$: Perfect positive correlation (both increase together)
- $r = -1$: Perfect negative correlation (one increases, other decreases)
- $r = 0$: No linear relationship

**But here's the key:** Correlation doesn't say *why*. It just says there's a pattern.

## Real-world scenario: Ticket length and escalations

You notice: tickets with higher token count get escalated more often.

Correlation looks strong! So you think: *"Let's limit prompt length!"*

But hold on. What if:
- Complex tickets naturally have longer descriptions AND longer resolution times?
- You limit prompt length, and now analysts *miss context*, causing even more escalations?

Before changing your policy, you need to:
1. **Segment by ticket type** — Does correlation hold within each category?
2. **Look for confounders** — Could something else explain both variables?
3. **Run a controlled pilot** — Test your hypothesis with one queue, measure outcomes

## How to try this

Run `notebooks/math-foundations/09_correlation_causation.ipynb` and:

- Calculate correlation on real ticket data
- See how a hidden variable (confounder) creates misleading correlation
- Compare naive vs careful interpretations

## The traps to avoid

❌ **The trap:** Turn correlation into policy without testing it  
✅ **The smart move:** Treat correlation as a "worth investigating" signal, then validate with experiments

## Checklist for you

- [ ] Did you brainstorm potential confounders?
- [ ] Did you break down the data by segment (queue, type, severity)?
- [ ] Did you write your reports carefully (no causal language unless proven)?
- [ ] Is your policy change backed by a pilot, not just correlation?

--8<-- "_abbreviations.md"
