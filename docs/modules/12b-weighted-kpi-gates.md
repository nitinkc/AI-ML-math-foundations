# Weighted KPI gates — rollout decisions explained

## The problem

By the time you're ready to roll out a new model version, you have many metrics. They don't all matter equally, and they don't all point in the same direction.

A **weighted KPI gate** is a structured decision framework that:
- Assigns a weight to each metric based on how much it matters
- Sets a target and a hard limit for each metric
- Combines them into a single go / hold / rollback signal

Without it, you're doing gut-feel rollouts — someone looks at overall accuracy, says "looks good", and ships. That's how a security incident FNR of 45% sneaks through behind a 91.4% headline number.

---

## Building the gate for the Monday ticket pipeline

| Metric | Why it matters | Weight |
|:---|:---|:---:|
| Security incident FNR | Missing a security incident is the highest-cost failure | 30% |
| Overall routing accuracy | General system health | 20% |
| Calibration ECE | Can you trust the probabilities? | 15% |
| Variance σ | Is the system stable run-to-run? | 15% |
| P95 latency | Is it fast enough for analysts? | 10% |
| Override rate (R2) | Is the model missing signals humans catch? | 10% |

Weights sum to 100%. Security incident FNR gets the highest weight because misrouting that class has consequences that dwarf all the others — as established in the stakes table from Module 00.

---

## The three-tier scoring system

Each metric is scored 0, 0.5, or 1:

```
Score 1.0  — metric is within target
Score 0.5  — metric is marginal (between target and hard limit)
Score 0.0  — metric has breached hard limit
```

For a candidate model update:

```
Metric               Weight   Target      Hard limit   Measured    Score
─────────────────────────────────────────────────────────────────────────
Security FNR          0.30    ≤ 0.10      ≤ 0.20       0.07        1.0
Routing accuracy      0.20    ≥ 92%       ≥ 88%        93.1%       1.0
Calibration ECE       0.15    ≤ 0.07      ≤ 0.12       0.09        0.5
Variance σ            0.15    ≤ 0.20      ≤ 0.40       0.15        1.0
P95 latency           0.10    ≤ 3.0s      ≤ 5.0s       2.4s        1.0
Override rate R2      0.10    ≤ 30/wk     ≤ 50/wk      28/wk       1.0
─────────────────────────────────────────────────────────────────────────
Weighted score:
  (0.30×1.0) + (0.20×1.0) + (0.15×0.5) + (0.15×1.0) + (0.10×1.0) + (0.10×1.0)
  = 0.30 + 0.20 + 0.075 + 0.15 + 0.10 + 0.10
  = 0.925
```

---

## The decision rules

```
Weighted score ≥ 0.90  AND  no metric at 0.0  →  GO
Weighted score 0.70–0.89  OR  one metric at 0.0  →  HOLD
Weighted score < 0.70  OR  security FNR at 0.0  →  ROLLBACK
```

The candidate above scores 0.925 with no zeros — **GO**.

### The hard veto

Security FNR breaching its hard limit triggers rollback *regardless of the overall score*. A model that is excellent in every other dimension but misses security incidents too often does not ship.

That asymmetry is the whole point of weighting. It encodes what you actually care about.

---

## A failing candidate — hold

A fine-tuned model that fixed MFA classification but destabilised variance:

```
Metric               Weight   Measured    Score
─────────────────────────────────────────────────
Security FNR          0.30    0.06        1.0
Routing accuracy      0.20    92.8%       1.0
Calibration ECE       0.15    0.11        0.5
Variance σ            0.15    0.38        0.5   ← marginal
P95 latency           0.10    4.2s        0.5   ← marginal
Override rate R2      0.10    41/wk       0.5   ← marginal
─────────────────────────────────────────────────
Weighted score: 0.75  →  HOLD
```

Score is 0.75, no hard limit breached. The model doesn't deploy. Investigate why fine-tuning increased variance and latency before proceeding.

---

## A failing candidate — rollback

A model that's been live for 3 weeks and drift has set in:

```
Metric               Weight   Measured    Score
─────────────────────────────────────────────────
Security FNR          0.30    0.23        0.0   ← hard limit breached
Routing accuracy      0.20    89.1%       0.5
Calibration ECE       0.15    0.14        0.0   ← hard limit breached
Variance σ            0.15    0.19        1.0
P95 latency           0.10    2.6s        1.0
Override rate R2      0.10    47/wk       0.5
─────────────────────────────────────────────────
Weighted score: 0.40
Security FNR = 0.0  →  HARD VETO  →  ROLLBACK
```

Immediately revert to the previous model version, open a drift investigation (Module 20), and do not redeploy until root cause is identified.

---

## Where the gate runs

```
New model candidate ready
    │
    ▼
Run on evaluation set
    │
    ▼
Calculate weighted KPI score
    │
    ├── Score ≥ 0.90, no zeros  →  Deploy
    ├── Score 0.70–0.89          →  Hold, investigate
    └── Score < 0.70 or veto     →  Do not deploy

Once live:
    │
    ▼
Continuous metric monitoring
    │
    ├── All metrics within targets   →  Continue
    ├── Marginal metrics trending    →  Alert, plan retraining
    └── Hard limit breached          →  Auto-rollback
```

---

## Connection to every previous module

The gate doesn't produce new information. It combines everything already being collected:

| Gate metric | Built in |
|:---|:---|
| Security FNR | Module 13 — Bias & Fairness |
| Routing accuracy | Module 12 — Evaluation in Production |
| Calibration ECE | Module 08 — Calibration |
| Variance σ | Module 05 — Variance & Std Dev |
| P95 latency | Module 19 — Cost & Latency |
| Override rate R2 | Module 16 — Human-in-the-loop |

When your security lead asks "why did you ship that model update on Tuesday?", you show them the gate scorecard. Every number has a source. Every decision has a trail.

---

## Checklist

- [ ] Are your gate weights documented and reviewed by stakeholders, not just engineers?
- [ ] Does security FNR carry a hard veto regardless of overall score?
- [ ] Is the gate running both at deployment time and continuously in production?
- [ ] Is an auto-rollback triggered when a hard limit is breached in production?
- [ ] Is every gate evaluation logged with the full metric table for audit purposes?
- [ ] Are hold decisions tracked — what was investigated, what was changed, when it was re-submitted?

--8<-- "_abbreviations.md"
