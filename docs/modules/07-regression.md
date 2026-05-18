# Module 7: Regression for AI Operations

## Plain-English first

Classification answers "which bucket?"
Regression answers "what number?"

In IT operations, useful numeric targets include:

- expected resolution hours
- expected queue wait minutes
- expected escalation handling effort

## Minimal math second

Simple linear regression model:

$$
\hat{y} = \beta_0 + \beta_1 x
$$

- $x$: input feature (for example, token count or queue load)
- $\hat{y}$: predicted numeric outcome (for example, resolution time)

Decision this supports:

**How should we staff shifts today based on predicted workload duration?**

## IT scenario: Time-to-resolution estimate

Suppose higher token counts in tickets correlate with longer handling time.
A regression model can estimate likely resolution duration and trigger staffing adjustments.

Use this as a planning aid, not a hard rule.

## Notebook third

Run `notebooks/math-foundations/07_regression.ipynb` to:

- Fit a simple line from synthetic service-desk data
- Compute residuals and basic error metrics
- Interpret the slope for operational planning

## Pitfall and anti-pitfall

- Pitfall: treating correlation as guaranteed causation
- Anti-pitfall: use regression for forecasting, then validate with live outcomes

## Quick checklist

- Is your target numeric and operationally meaningful?
- Do you monitor prediction error after deployment?
- Are outliers reviewed separately from normal tickets?
- Is the model used for guidance, not blind automation?

--8<-- "_abbreviations.md"
