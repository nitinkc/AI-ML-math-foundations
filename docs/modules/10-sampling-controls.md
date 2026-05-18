# Module 10: Sampling Controls

## Plain-English first

Sampling controls decide how conservative or creative model output is.

- Lower [temperature] usually increases reliability and repeatability.
- Higher [temperature] can increase variety but also risk.
- [top_p] and [top-k] narrow the candidate token set before selection.

These controls are operations knobs, not just prompt tweaks.

## Minimal math second

After filtering candidates, probabilities are renormalized:

$$
P'(i)=\frac{P(i)}{\sum_{j \in S} P(j)} \quad \text{for } i \in S
$$

Where $S$ is the kept token set after top-p/top-k filtering.

Decision this supports:

**Which decoding profile should be default for production-safe responses vs exploratory drafting?**

## IT scenario: Summary profile presets

For service desk summaries you might define:

- Safe profile: low temperature, tighter top-p
- Creative profile: moderate temperature, broader top-p

Route by use case: compliance notes use safe profile; optional draft replies may use creative profile.

## Notebook third

Run `notebooks/math-foundations/10_sampling_controls.ipynb` to:

- apply top-k and top-p filtering to toy distributions
- compare renormalized distributions under different settings
- inspect diversity vs concentration metrics

## Pitfall and anti-pitfall

- Pitfall: one global sampling profile for all tasks and risk levels
- Anti-pitfall: maintain task-specific decoding profiles with change control

## Quick checklist

- Are decoding presets documented by business risk tier?
- Are temperature and top-p changes tracked in release notes?
- Is compliance-sensitive output using conservative settings?
- Do you evaluate output drift after decoding changes?

## From math to decision

Sampling controls should be governed like any production configuration: versioned, tested, and tied to measurable outcomes.

--8<-- "_abbreviations.md"
