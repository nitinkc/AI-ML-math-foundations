# Module 5: Variance and Standard Deviation

## Plain-English first

Your team runs the same summary prompt across 30 tickets every day.
Average quality looks fine, but some days are very unstable.
That instability creates rework, escalation noise, and operator distrust.

The mean tells you center.
Variance and standard deviation tell you spread.

## Minimal math second

Let $x_1, x_2, ..., x_n$ be repeated outcomes (for example, quality scores).

$$
\mathrm{Var}(X) = \frac{1}{n}\sum_{i=1}^{n}(x_i - \mu)^2
$$

Where $\mu$ is the mean. Standard deviation is:

$$
\sigma = \sqrt{\mathrm{Var}(X)}
$$

Decision this supports:

**Is this prompt profile stable enough for production automation, or too noisy for safe rollout?**

## IT scenario: Run-to-run stability

- Profile A average quality: 4.2/5, low spread
- Profile B average quality: 4.2/5, high spread

If averages tie, choose lower spread for operational predictability.

## Notebook third

Run `notebooks/math-foundations/05_variance_stddev.ipynb` to:

- Compute mean, variance, and standard deviation from repeated runs
- Compare two prompt profiles with similar means but different spread
- Turn spread into a simple "go / monitor / hold" stability decision

## Pitfall and anti-pitfall

- Pitfall: reporting only average quality and hiding volatility
- Anti-pitfall: pair mean with standard deviation in every release review

## Quick checklist

- Do you measure variability across repeated runs?
- Do you compare prompt profiles on both mean and spread?
- Do unstable prompts require manual review before release?
- Do you track stability trend week over week?

--8<-- "_abbreviations.md"
