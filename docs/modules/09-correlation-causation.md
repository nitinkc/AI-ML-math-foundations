# Module 9: Correlation vs Causation in AI Ops

## Plain-English first

Two metrics moving together does not prove one causes the other.
In IT copilots, this is a common trap during incident review.

Example: "Long prompts caused SLA breaches."
Maybe true, maybe not.
A hidden factor like outage complexity could increase both prompt length and resolution time.

## Minimal math second

Use correlation as a signal, not a conclusion:

$$
r = \frac{\sum_i (x_i-\bar{x})(y_i-\bar{y})}{\sqrt{\sum_i (x_i-\bar{x})^2}\sqrt{\sum_i (y_i-\bar{y})^2}}
$$

- $r$ near 1: strong positive association
- $r$ near -1: strong negative association
- $r$ near 0: weak linear association

Decision this supports:

**Do we investigate a relationship further, or avoid making policy changes from correlation alone?**

## IT scenario: Ticket length and escalations

You observe correlation between token count and escalation rate.
Before changing routing policy:

- segment by ticket type
- check confounders (severity, team load, incident class)
- run controlled pilot before enforcing a new rule

## Notebook third

Run `notebooks/math-foundations/09_correlation_causation.ipynb` to:

- compute correlation on synthetic ticket data
- see how a confounder can create a misleading relationship
- compare naive and segmented interpretations

## Pitfall and anti-pitfall

- Pitfall: converting correlation into policy without controlled evidence
- Anti-pitfall: treat correlation as a triage signal and validate with experiments

## Quick checklist

- Did you identify potential confounders?
- Did you compare segmented views (by queue or severity)?
- Did you avoid causal language in reports unless tested?
- Is your policy change backed by pilot outcomes?

## From math to decision

Correlation is useful for prioritizing investigation, not assigning blame or enforcing production controls by itself.

--8<-- "_abbreviations.md"
