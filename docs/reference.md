# Notebook Code Explanations and Math Reference

## Lab 00: Warmup

### Basic Probability Calculation

```python
# Normalize raw scores by dividing each value by the total
probs = [v / sum(exp_shifted) for v in exp_shifted]
```

**Math:** normalization creates a valid distribution where probabilities sum to 1.
This makes outputs comparable across different runs and inputs.

$$
P(i) = \frac{v_i}{\sum_j v_j}
$$

### Softmax Formula

Also check Pre-Activation

$$
P(i) = \frac{e^{z_i}}{\sum_j e^{z_j}}
$$

Softmax converts raw scores (logits) into probabilities in `[0, 1]`.
It also magnifies relative score differences so the model can rank choices clearly.

---

## Lab 01: Tokenization

### Token Counting

```python
def pseudo_token_count(text):
    return len(text.split())
```

This notebook uses a simple approximation. Real models use specialized tokenizers.
Use this approximation to reason about budget pressure before adding production tokenizers.

### Budget Constraint Equation

$$
T_{total} = T_{system} + T_{history} + T_{user} + T_{output}
$$

Constraint:

$$
T_{total} \le T_{window}
$$

Operational decisions:

- Short tickets: keep full history.
- Long tickets: summarize or trim history.

The key idea is that token budget is a capacity constraint, similar to memory limits.

---

## Lab 02: Probability

### Normalization Step

```python
total = sum(raw_scores.values())
probs = {k: v / total for k, v in raw_scores.items()}
```

This turns arbitrary scores into valid confidence values for policy thresholds.

### Expected Value Formula

$$
\mathbb{E}[X] = \sum_i x_i P(i)
$$

Example workload estimate:

- `6 * 0.6 + 14 * 0.25 + 32 * 0.15 = 11.9` minutes per ticket.

Expected value helps estimate staffing load from predicted intent mix.

---

## Lab 03: Logits and Softmax

### Numerical Stability

```python
shift = max(scaled)
exps = [math.exp(x - shift) for x in scaled]
```

Equivalent identity:

$$
\frac{e^{x-c}}{\sum_j e^{y_j-c}} = \frac{e^x}{\sum_j e^{y_j}}
$$

Subtracting a constant avoids overflow but keeps the same probability ratios.

### Temperature Scaling

$$
P_T(i) = \frac{e^{z_i/T}}{\sum_j e^{z_j/T}}
$$

- `T < 1`: sharper, more deterministic.

- `T = 1`: baseline.

- `T > 1`: flatter, more stochastic.

Use lower temperature for routing and higher temperature for exploratory drafting.

---

## Lab 04: Entropy

### Shannon Entropy

$$
H(P) = -\sum_i P(i) \log_2 P(i)
$$

Higher entropy means more uncertainty spread across classes.

### Normalized Entropy

$$
H_{norm} = \frac{H(P)}{\log_2 n}
$$

Policy example:

- Auto-route only if top probability is high and entropy is low.

Combining both signals reduces overconfident automation mistakes.

---

## Lab 05: Variance and Standard Deviation

### Variance Formula

$$
\mathrm{Var}(X) = \frac{1}{n}\sum_i (x_i - \mu)^2
$$

$$
\sigma = \sqrt{\mathrm{Var}(X)}
$$

Mean shows center; variance and standard deviation show stability.

### Stability Policy

```python
def stability_decision(std, good=0.20, watch=0.35):
    if std <= good:
        return "go"
    if std <= watch:
        return "monitor"
    return "hold"
```

- Low spread: stable for production.

- High spread: investigate before rollout.

This keeps quality predictable across repeated runs.

---

## Lab 06: Determinism

### Argmax Operation

```python
idx = max(range(len(p)), key=lambda i: p[i])
choice = labels[idx]
```

Argmax is deterministic for fixed inputs.
It is preferred in high-control workflows where reproducibility matters.

### Sampling Operation

```python
def sample_choice(labels, p, seed=None):
    rng = random.Random(seed)
    ...
```

Sampling is stochastic; seeding enables reproducibility.
This is useful when you want diversity but still need auditability.

---

## Lab 07: Regression

### Linear Regression Model

$$
\hat{y} = b_0 + b_1 x
$$

- `b0`: intercept.

- `b1`: slope.

Interpret slope as "expected output change per one-unit input increase."

### Residual Analysis

$$
r_i = y_i - \hat{y}_i
$$

Persistent residual patterns indicate bias.
Residual shape often reveals missing features or non-linear behavior.

### Error Metrics

$$
\mathrm{MAE} = \frac{1}{n}\sum_i |r_i|
$$

$$
\mathrm{RMSE} = \sqrt{\frac{1}{n}\sum_i r_i^2}
$$

RMSE penalizes large misses more heavily than MAE.

---

## Lab 08: Classification and Calibration

### Precision and Recall

$$
\mathrm{Precision} = \frac{TP}{TP + FP}
$$

$$
\mathrm{Recall} = \frac{TP}{TP + FN}
$$

Threshold tuning is a business trade-off between false positives and false negatives.

### Calibration Check

Compare predicted probabilities vs observed frequencies per bin.

- Well-calibrated model: predicted and observed rates are close.

If they diverge, confidence thresholds should be recalibrated.

---

## Lab 09: Correlation vs Causation

### Correlation Coefficient

$$
r = \frac{\mathrm{cov}(x, y)}{\sigma_x\sigma_y}
$$

`r` measures association, not causality.
Treat correlation as an investigation signal, not a policy proof.

### Confounder Segmentation

- Segment by severity/queue/time before making policy conclusions.

- Use controlled experiments for causal claims.

Segmentation helps reveal confounders before rollout decisions.

---

## Lab 10: Sampling Controls

### Top-k Filtering

```python
pairs = sorted(zip(labels, probs), key=lambda x: x[1], reverse=True)[:k]
renormalize(pairs)
```

Top-k caps candidate count; top-p caps cumulative probability mass.

### Top-p Filtering

```python
keep = []
total = 0.0
for label, pr in sorted_pairs:
    keep.append((label, pr))
    total += pr
    if total >= p:
        break
```

### Renormalization

$$
P_{new}(i) = \frac{P_{old}(i)}{\sum_{j \in S} P_{old}(j)}
$$

Renormalization is required so filtered probabilities still sum to 1.

---

## Lab 11: Guardrails and Thresholds

### Three-Band Policy

```python
def action(p, low=0.55, high=0.80):
    if p >= high:
        return 'auto'
    if p <= low:
        return 'abstain'
    return 'human-review'
```

- Auto: `p >= high`.

- Review: `low < p < high`.

- Abstain: `p <= low`.

This three-band policy maps confidence directly to operational action.

### Threshold Tuning

- More auto: faster, riskier.

- More review: safer, costlier.

Tune by risk tier rather than using one global threshold.

---

## Lab 12: Evaluation in Production

### Weighted KPI

```python
KPI = 0.4 * precision + 0.4 * sla_hit_rate + 0.2 * inverse_override_rate
```

Weights should reflect business priorities and compliance risk.

### Rollout Gate

```python
delta = candidate_kpi - baseline_kpi
if delta >= 0.01:
    rollout
elif delta <= -0.01:
    rollback
else:
    hold
```

Decision bands:

- `delta >= 0.01`: expand rollout.
- `-0.01 < delta < 0.01`: hold and monitor.
- `delta <= -0.01`: rollback.

Predefined gates prevent subjective rollout decisions under pressure.

---

## General Tips for Code and Math

1. Validate assumptions (probabilities sum to 1, thresholds documented).
2. Write formulas first, then code.
3. Test edge cases.
4. Monitor operational metrics, not only prediction outputs.