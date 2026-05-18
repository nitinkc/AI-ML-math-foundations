# Module 6: Deterministic vs Stochastic Behavior

## Plain-English first

In some workflows, the same input must always produce the same output.
In other workflows, controlled variation is useful for brainstorming drafts.

Deterministic behavior improves auditability.
Stochastic behavior improves diversity.
You choose based on risk and business objective.

## Minimal math second

Deterministic next-choice policy:

$$
\hat{i} = \arg\max_i P(i)
$$

Stochastic policy samples from the full distribution $P(i)$.

Decision this supports:

**Should this step run in repeatable compliance mode, or exploratory drafting mode?**

## IT scenario: Routing vs drafting

- Ticket intent routing: use deterministic mode for consistent triage
- Draft response suggestions: allow stochastic mode for alternative phrasings

A common architecture is deterministic for control steps, stochastic for content-generation steps.

## Notebook third

Run `notebooks/math-foundations/06_determinism.ipynb` to:

- Compare argmax output with sampling output
- Show seeded vs unseeded sampling behavior
- Measure output diversity across repeated runs

## Pitfall and anti-pitfall

- Pitfall: enabling randomness in high-risk routing or policy enforcement
- Anti-pitfall: lock deterministic settings for control decisions and log configuration

## Quick checklist

- Is determinism required for this business function?
- Are random seeds and sampling settings captured in telemetry?
- Are creative steps isolated from compliance-critical steps?
- Can you reproduce a past output for audit?

--8<-- "_abbreviations.md"
