# Module 5: Variance and Standard Deviation

## The problem with averages

Your team has been running the Monday ticket through two different prompt configurations
for the past 30 days. Both are being evaluated on response quality (1–5 scale) and
routing accuracy (did it send the ticket to the right queue?).

The weekly report lands on your desk:

```
Prompt Config A:  avg quality = 4.2 / 5   avg routing accuracy = 87%
Prompt Config B:  avg quality = 4.2 / 5   avg routing accuracy = 87%
```

Identical. Your manager asks: "Which one do we deploy?"

You can't answer that question from averages alone. Here's what the averages are hiding:

```
Config A daily quality scores (30 days):
4.1, 4.3, 4.2, 4.0, 4.4, 4.2, 4.1, 4.3, 4.2, 4.0,
4.3, 4.2, 4.4, 4.1, 4.2, 4.3, 4.0, 4.2, 4.3, 4.1,
4.2, 4.4, 4.1, 4.3, 4.2, 4.0, 4.3, 4.2, 4.1, 4.4

Config B daily quality scores (30 days):
4.8, 2.9, 4.7, 3.1, 4.8, 2.8, 4.9, 3.0, 4.7, 2.9,
4.8, 3.2, 4.6, 2.7, 4.9, 3.0, 4.8, 2.8, 4.7, 3.1,
4.8, 2.9, 4.7, 3.0, 4.9, 2.8, 4.8, 3.1, 4.7, 2.9
```

Config B is oscillating wildly — great one day, terrible the next. Config A barely
moves. The averages are the same. The systems are completely different.

**Mean tells you where the centre is. Variance tells you how far things wander from it.**

---

## The math, built up from scratch

Start with the mean — the centre of your distribution:

$$
\mu = \frac{1}{n}\sum_{i=1}^{n} x_i
$$

For both configs: $\mu = 4.2$

Now measure how far each day's score wanders from that centre.
You can't just subtract and average — the positive and negative gaps cancel out.
So you **square** each gap first, then average:

$$
\mathrm{Var}(X) = \frac{1}{n}\sum_{i=1}^{n}(x_i - \mu)^2
$$

Squaring does two things: it eliminates negatives, and it penalises large swings
more than small ones (a gap of 2 contributes 4× more than a gap of 1).

Finally, take the square root to bring the units back to the original scale
(quality points, not quality points squared):

$$
\sigma = \sqrt{\mathrm{Var}(X)}
$$

This is your **standard deviation** — the typical distance any single day's score
sits from the average.

---

## Calculated on our two configs

**Config A** (scores hovering between 4.0 and 4.4):

$$
\mathrm{Var}(A) = \frac{1}{30}\sum(x_i - 4.2)^2 \approx 0.017
$$
$$
\sigma_A = \sqrt{0.017} \approx 0.13
$$

On any given day, Config A's quality score is typically within **±0.13** of 4.2.
The tightest band imaginable for a live system.

**Config B** (scores bouncing between 2.7 and 4.9):

$$
\mathrm{Var}(B) = \frac{1}{30}\sum(x_i - 4.2)^2 \approx 0.81
$$
$$
\sigma_B = \sqrt{0.81} \approx 0.90
$$

On any given day, Config B's quality score is typically within **±0.90** of 4.2.
That means a "normal" day could land anywhere from 3.3 to 5.0.

| Config | Mean | Std Dev | Worst realistic day | Deploy? |
|---|---|---|---|---|
| A | 4.2 | 0.13 | ~3.94 | ✅ Stable |
| B | 4.2 | 0.90 | ~3.30 | ❌ Too volatile |

Same average. Config A wins — not because it's better on average,
but because you know what you're getting.

---

## Now connect this to the Monday ticket

Remember: our Monday ticket has `security_incident` as a possible class.
A security incident misrouted to a tier-1 agent isn't just a quality dip —
it's a potential breach.

Now think about what Config B's volatility means in *routing accuracy* terms.
If routing accuracy bounces between 72% and 96% day-to-day (same average: 87%),
on a bad day with 500 tickets:

```
500 tickets × (1 - 0.72) = 140 misrouted tickets
5% are security incidents = 7 active threats sent to wrong queue
```

On a good day, that number drops to near zero. You don't know which kind of day
you're having until it's over.

Config A's routing accuracy, by contrast, stays between 85% and 89%.
Worst day: `500 × (1 - 0.85) = 75 misrouted`, security incidents ≈ 3–4.
Not zero — but predictable enough to staff for.

**Stability isn't just a quality metric. It's a risk management metric.**

---

## What drives variance in prompt outputs?

Before you can reduce variance, you need to know where it comes from.
For the Monday ticket and tickets like it, the usual culprits are:

**1. Ambiguous ticket language**
Tickets that mention multiple symptoms (account lock + VPN + patch notice)
give the model more room to interpret differently on each run.
High-entropy tickets (from Module 4) tend to produce high-variance outputs.

**2. Temperature settings**
Recall from Module 3: higher temperature flattens the probability distribution
and increases randomness. Temperature directly controls output variance.
A T=1.5 setting on a production classifier is asking for Config B behaviour.

**3. Context window changes**
If conversation history grows and gets truncated differently across runs
(Module 1), the model sees a slightly different input each time.
Different input → different output → variance that looks random but isn't.

**The operational rule:** When you see high variance, diagnose *which* of these
is driving it before you change anything. Lowering temperature when the real
problem is truncated context just masks the symptom.

---

## The 68-95-99.7 rule: your intuition calibrator

Once you have a standard deviation, this rule tells you what to expect:

- **68%** of days will fall within **±1σ** of the mean
- **95%** of days will fall within **±2σ** of the mean
- **99.7%** of days will fall within **±3σ** of the mean

For Config A (σ = 0.13):
- 95% of days: quality between **3.94 and 4.46** — tight, trustworthy
- Extreme outlier (3σ): quality could drop to **3.81** — still acceptable

For Config B (σ = 0.90):
- 95% of days: quality between **2.40 and 6.00** — floored at 2.4, theoretical ceiling above 5.0
- Extreme outlier: quality could theoretically drop to **1.50** — unusable

The 3σ floor is your **worst-case planning number.** Staff for it.
If Config B's 3σ floor lands below your SLA minimum quality threshold,
it should never reach production regardless of its average.

---

## The go/no-go decision framework

Before deploying any new prompt configuration, require it to pass both gates:

```
Gate 1 — Mean:       μ ≥ your quality floor (e.g. 4.0 / 5)
Gate 2 — Stability:  σ ≤ your variance ceiling (e.g. 0.25)
```

If it passes Gate 1 but fails Gate 2: it's too unpredictable. Keep tuning.
If it fails Gate 1: it's not good enough on average. Fundamental rework needed.
Only pass both: deploy to production.

Config A: μ = 4.2 ✅, σ = 0.13 ✅ → **Deploy**
Config B: μ = 4.2 ✅, σ = 0.90 ❌ → **Do not deploy**

---

## How to practice this

Open `notebooks/math-foundations/05_variance_stddev.ipynb` and work through:

1. Calculate mean, variance, and standard deviation for Config A and B by hand, then verify with code
2. Apply the 68-95-99.7 rule to find the realistic worst-case quality floor for each config
3. Build the two-gate go/no-go decision and run it on three new prompt configs (provided in the notebook)
4. Investigate: does Config B's variance correlate with ticket entropy? (Hint: it should)

---

## Checklist before moving on

- [ ] Can you explain why two configs with the same average can be completely different in practice?
- [ ] Do your prompt evaluation reports always show standard deviation alongside mean?
- [ ] Have you set an explicit σ ceiling for your go/no-go deployment gate?
- [ ] Do you investigate the *source* of variance (temperature, truncation, ambiguity) before patching it?
- [ ] Are you tracking stability week-to-week, not just at deployment time?

--8<-- "_abbreviations.md"