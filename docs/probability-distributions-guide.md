# Probability Distributions and Stochastic Processes

This guide demystifies common distribution terms in AI/ML and explains what they mean for IT operations.

## Quick orientation: what this page covers

- How to think about **stochastic vs deterministic** behavior in AI systems
- The difference between **discrete** and **continuous** distributions
- The core distributions used in practice (with formulas, IT examples, and plots)
- How these map to common AI operations decisions

Further reading: https://www.linkedin.com/pulse/8-probability-distributions-every-risk-analyst-should-david-vose/

---

## Foundation: discrete vs continuous distributions

A probability distribution describes how likely each value of a random variable is.

### Discrete distributions

Use these when outcomes are countable (0, 1, 2, ...).

- Uses a **PMF** (Probability Mass Function)
- Gives exact probability for each outcome
- Examples: coin toss, die roll, incidents per hour
- In this guide: Bernoulli, Binomial, Poisson

### Continuous distributions

Use these when outcomes can take any value in a range.

- Uses a **PDF** (Probability Density Function)
- Probability at a single exact point is not used directly; probability is interpreted over an interval
- Examples: latency, wait time, temperature
- In this guide: Normal, Exponential

### One more useful term

- **CDF** (Cumulative Distribution Function): probability that a variable is less than or equal to a value, `P(X <= x)`

---

## What is a stochastic process?

A stochastic process is a sequence of events where each outcome is governed by probability(of the previous one), not a fixed rule.

In IT copilots, stochastic behavior appears often:

- Each generated token is sampled from a probability distribution.
- Running the same prompt twice at temperature > 0 can produce different outputs.
- A classifier can assign different likely intents from the same input.

A memorable analogy: in a wedding sequence, each next event depends on previous social context (a pattern exists), 
but the exact reaction still varies (random outcome). That is structured randomness.
 - [Shaadi and foofa ka rona - each event is determined with previous one](https://www.youtube.com/watch?v=JXvMgM5NfXQ)

**Deterministic** means fixed and repeatable.

**Stochastic** means random but patterned by a probability distribution.

---

## Why distributions matter

A distribution maps possible outcomes to probabilities.

Understanding the active distribution helps you reason about:

- likely values (mean, mode)
- spread (variance)
- event behavior (rare, common, bursty)

When you tune temperature or top-p, you are reshaping the underlying distribution.

---

## Main distributions (with IT context)

### Uniform distribution (discrete example)

Every outcome has the same probability.

$$
P(x) = \frac{1}{n} \quad \text{for } n \text{ equally likely outcomes}
$$

**AI relevance:** very high temperature can flatten output toward uniform, usually too random for IT workflows.

**IT example:** if a classifier is almost equally split across labels like "account unlock," "vpn issue," and "security incident," uncertainty is high (entropy near maximum), so escalation to a human may be safer.

**Visualize it:**
```python
import matplotlib.pyplot as plt
import numpy as np

outcomes = 10
probs = np.ones(outcomes) / outcomes
plt.bar(range(outcomes), probs, color='steelblue', alpha=0.7, edgecolor='black')
plt.xlabel('Outcome'); plt.ylabel('Probability')
plt.title('Uniform Distribution (10 equally likely outcomes)')
plt.ylim(0, 0.2); plt.grid(axis='y', alpha=0.3)
plt.show()
```

*Each outcome has equal probability 0.1 -- perfectly flat.*

---

### Bernoulli distribution (discrete)

A single yes/no trial with probability of success `p`.

$$
P(X = 1) = p \qquad P(X = 0) = 1 - p
$$

**AI relevance:** one classification decision (correct or incorrect) is Bernoulli.

**IT example:** if a classifier routes a security incident correctly with probability `p = 0.88`, each single routing event is Bernoulli.

**Visualize it:**
```python
import matplotlib.pyplot as plt

# p = 0.7 (70% success rate)
outcomes = ['Failure', 'Success']
probs = [0.3, 0.7]
plt.bar(outcomes, probs, color=['#ff6b6b', '#51cf66'], alpha=0.7, edgecolor='black')
plt.ylabel('Probability')
plt.title('Bernoulli Distribution (p=0.7)')
plt.ylim(0, 1); plt.grid(axis='y', alpha=0.3)
plt.show()
```

*Two outcomes: 30% fail, 70% succeed.*

---

### Binomial distribution (discrete)

Counts successes in `n` independent Bernoulli trials.

$$
P(X = k) = \binom{n}{k} p^k (1-p)^{n-k}
$$

- `n`: number of trials
- `k`: number of successes
- `p`: success probability per trial

**AI relevance:** correct routes in a batch follow a binomial pattern.

**IT example:** in 200 daily tickets with route accuracy `p = 0.88`, expected correctly routed tickets are `n * p = 176`, and daily variation around 176 follows Binomial behavior.

**Visualize it:**
```python
import matplotlib.pyplot as plt
from scipy.stats import binom

n, p = 20, 0.7  # 20 trials, 70% success rate
x = range(0, n+1)
probs = [binom.pmf(k, n, p) for k in x]
plt.bar(x, probs, color='teal', alpha=0.7, edgecolor='black')
plt.xlabel('Number of Successes'); plt.ylabel('Probability')
plt.title(f'Binomial Distribution (n=20, p=0.7)')
plt.grid(axis='y', alpha=0.3)
plt.show()
```

*Most likely outcome: ~14 successes out of 20 (peaks around `n * p = 14`).*

---

### Normal (Gaussian) distribution (continuous)

Bell-curve behavior: values cluster around a mean.

$$
P(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{(x-\mu)^2}{2\sigma^2}}
$$

- `mu`: mean
- `sigma`: standard deviation

**AI relevance:** many aggregated metrics are approximately normal (often explained by the Central Limit Theorem).

**IT example:** if average resolution time is 11.9 minutes with standard deviation 2.1, roughly 68% of tickets fall within `11.9 +/- 2.1` and ~95% within `11.9 +/- 4.2`.

**Visualize it:**
```python
import matplotlib.pyplot as plt
import numpy as np
from scipy.stats import norm

mu, sigma = 0, 1  # mean=0, std=1 (standard normal)
x = np.linspace(-4, 4, 100)
y = norm.pdf(x, mu, sigma)
plt.plot(x, y, linewidth=2.5, color='darkviolet')
plt.fill_between(x, y, alpha=0.3, color='darkviolet')
plt.xlabel('Value'); plt.ylabel('Probability Density')
plt.title(f'Normal Distribution (mu=0, sigma=1)')
plt.grid(alpha=0.3)
plt.show()
```

*Classic bell curve: 68% of values within +/-1 sigma, 95% within +/-2 sigma.*

---

### Exponential distribution (continuous)

Models waiting time between random events at a constant average rate.

$$
P(t) = \lambda e^{-\lambda t} \quad t \ge 0
$$

- `lambda`: event rate
- mean waiting time: `1 / lambda`

**AI relevance:** time between arrivals or escalations is often modeled this way.

**IT example:** if tickets arrive at 12/hour, expected gap is `1/12` hour (~5 minutes). Useful for queue and staffing design.

**Visualize it:**
```python
import matplotlib.pyplot as plt
import numpy as np
from scipy.stats import expon

lam = 2  # lambda = 2 (2 events per unit time on average)
t = np.linspace(0, 3, 100)
y = expon.pdf(t, scale=1/lam)  # scale = 1/lambda
plt.plot(t, y, linewidth=2.5, color='coral')
plt.fill_between(t, y, alpha=0.3, color='coral')
plt.xlabel('Time'); plt.ylabel('Probability Density')
plt.title(f'Exponential Distribution (lambda=2)')
plt.grid(alpha=0.3)
plt.show()
```

*Decays rapidly: most events happen soon, long waits are rare.*

---

### Poisson distribution (discrete)

Counts events in a fixed window when arrivals are independent with constant average rate.

$$
P(X = k) = \frac{\lambda^k e^{-\lambda}}{k!}
$$

- `lambda`: expected count in the window
- `k`: observed count

**AI relevance:** incidents per hour, errors per 1,000 requests, arrivals per interval.

**IT example:** if average routing errors are 3 per 1,000 tickets, probability of zero errors in a 1,000-ticket batch is `P(X=0)=e^{-3}~0.05`.

**Visualize it:**
```python
import matplotlib.pyplot as plt
from scipy.stats import poisson

lam = 5  # lambda = 5 (expect 5 events per window)
k = range(0, 15)
probs = [poisson.pmf(x, lam) for x in k]
plt.bar(k, probs, color='darkgreen', alpha=0.7, edgecolor='black')
plt.xlabel('Count'); plt.ylabel('Probability')
plt.title(f'Poisson Distribution (lambda=5)')
plt.grid(axis='y', alpha=0.3)
plt.show()
```

*Peak at `k=5` (expected count), with a right tail for rarer high counts.*

---

## How these map to AI operations

| What you observe | Distribution in play |
|---|---|
| Single token selection | Categorical (generalized Bernoulli) |
| Correct routes per batch | Binomial |
| Average score across runs | Approx. Normal |
| Time between arrivals | Exponential |
| Incidents per hour | Poisson |
| Very high-temperature output | Approaches Uniform |

---

## Stochastic vs deterministic in LLM settings

| Setting | Behavior | Distribution shape |
|---|---|---|
| temperature = 0 (argmax) | Deterministic: always top token | Degenerate spike |
| temperature = 0.7 | Mildly stochastic | Peaked categorical |
| temperature = 1.0 | Baseline stochastic | Native model distribution |
| temperature = 2.0 | Highly stochastic | Flatter, toward uniform |

Lower temperature concentrates probability mass around top choices.
Higher temperature spreads probability mass across more choices.

**Compare temperature effects:**
```python
import matplotlib.pyplot as plt
import numpy as np

# Logits: [5, 3, 1, 0.5, 0.1] (top choice strongly favored)
logits = np.array([5, 3, 1, 0.5, 0.1])

fig, axes = plt.subplots(1, 4, figsize=(14, 3))
temperatures = [0.5, 1.0, 1.5, 2.0]

for ax, T in zip(axes, temperatures):
    probs = np.exp(logits / T) / np.sum(np.exp(logits / T))
    ax.bar(range(5), probs, color='steelblue', alpha=0.7, edgecolor='black')
    ax.set_title(f'Temperature = {T}')
    ax.set_ylim(0, 1)
    ax.set_ylabel('Probability' if T == 0.5 else '')
    ax.grid(axis='y', alpha=0.3)

plt.tight_layout()
plt.show()
```

*At T=0.5: concentrated on choice 0. At T=2.0: spread across all choices.*

---

## Decision quick-reference

- Planning staffing: Poisson for arrival counts, Exponential for wait times.
- Routing accuracy in batches: Binomial.
- Quality at scale and SLA banding: Normal for averaged metrics.
- Confidence spread: Categorical/Uniform depending on decoding settings.
- Uncertainty flagging: entropy over the categorical distribution.

---

## Common traps

- Treating all data as normal (rare events often fit Poisson better).
- Looking at mean without variance.
- Treating stochastic behavior as unstructured noise.
- Using probabilities that do not sum to 1.

---

## Key takeaway

You do not need to derive distributions from scratch. You need to identify which pattern fits your operational data and apply the right tool:

- Event counts -> Binomial or Poisson
- Waiting times -> Exponential
- Averages across many trials -> Normal
- Single token choice -> Categorical


Once you match the right distribution to your situation, the math becomes a practical tool
rather than abstract notation.

--8<-- "_abbreviations.md"