# Module 13 — Bias & Fairness: are certain users or issues being misclassified?

Your classifier is live. Accuracy looks fine in aggregate. Then someone on the operations team asks a question nobody thought to ask before:

> "Does the model classify tickets differently depending on *who sent them* — department, role, writing style, shift?"

You pull 30 days of resolved tickets. The aggregate numbers are reassuring:

```
Overall routing accuracy:   91.4%
```

But then you segment.

---

## The disaggregated view

| Segment                    | Tickets | Correct route | Accuracy |
|:---------------------------|:-------:|:-------------:|:--------:|
| HQ employees               |   620   |      581      |  93.7%   |
| Remote workers             |   410   |      364      |  88.8%   |
| Non-native English writers |   195   |      158      |  81.0%   |
| Night-shift submissions    |   140   |      109      |  77.9%   |

Aggregate accuracy: **91.4%** — fine. But night-shift tickets are being correctly routed only **77.9%** of the time. That's a 15-point gap.

**This is the central lesson of Module 13: aggregate metrics hide disparity. Always disaggregate.**

---

## Why does the gap exist? Three candidates

Before you act, you do what Module 8 (calibration) taught you — you don't trust a pattern until you've interrogated it. And what Module 9 (correlation vs causation) reinforced — correlation isn't causation.

### Candidate 1 — Training data imbalance

Your 30-day training corpus had:

```
HQ tickets:            620  (44%)
Remote tickets:        410  (29%)
Non-native English:    195  (14%)
Night-shift:           140  (10%)
Other:                  35   (3%)
```

Night-shift tickets were 10% of training data. The model saw fewer examples of how those tickets are written — shorter, sometimes more terse, written under fatigue — and learned less robust signal for that segment.

### Candidate 2 — Vocabulary shift

Tokenization (Module 1) is the silent culprit here. Night-shift tickets often contain abbreviated, clipped language:

```
Day-shift:    "I've been locked out of my account since this morning and
               my VPN connection keeps dropping every few minutes."

Night-shift:  "locked out again + vpn down. same as last week"
```

The second ticket produces far fewer tokens. Phrases like "I've been" and "since this morning" carry classification signal the model learned to rely on. Strip them out and confidence drops. Entropy rises. Misrouting follows.

### Candidate 3 — Proxy discrimination

This is the most dangerous candidate. "Night-shift" may be correlated with other attributes — job function, contract type, location — that the model has indirectly learned to use. The model was never told about shift patterns. But if shift correlates with department, and department correlates with ticket vocabulary, the model may have effectively learned a proxy for something it was never meant to consider.

You can't rule this out from accuracy numbers alone. You need to look at the logits (Module 3) and the interpretability layer (Module 14).

---

## Measuring the gap formally

For each segment $g$, define the **false-negative rate for security incidents** — the rate at which a real security incident gets misrouted to a lower-priority queue:

$$
\text{FNR}_g = \frac{\text{security incidents misrouted in segment } g}{\text{total security incidents in segment } g}
$$

| Segment            | Security incidents  | Misrouted  |    FNR    |
|:-------------------|:-------------------:|:----------:|:---------:|
| HQ                 |         38          |     2      |   0.053   |
| Remote             |         27          |     3      |   0.111   |
| Non-native English |         14          |     4      |   0.286   |
| Night-shift        |         11          |     5      | **0.455** |

Nearly **half of security incidents from night-shift writers are being silently misrouted to tier-1 agents.** Remember the stakes table from Module 00: misrouting a security incident leaves a potential attacker inside the network while a tier-1 agent sends a password reset link. A 45.5% miss rate on that category is not a metrics problem — it's a security posture problem.

---

## The fairness criteria in plain terms

There are several ways to define "fair." They conflict. Knowing which one you're using is not optional.

**Demographic parity** — each segment should receive the same *escalation rate*:

$$
P(\hat{y} = \text{security\_incident} \mid g = \text{HQ}) \approx P(\hat{y} = \text{security\_incident} \mid g = \text{night-shift})
$$

**Equalised odds** — each segment should have the same *true positive rate* and *false positive rate* for each class.

**Calibration fairness** — when the model says 0.80 for security incident, it should be right 80% of the time, regardless of which segment the ticket came from.

For a service desk copilot, **calibration fairness and equalised FNR** are the right criteria. You don't want equal escalation rates if some segments genuinely submit more security incidents. You want the model to be equally *reliable* across segments — not equally aggressive.

---

## What to do about it

### Short-term: segment-specific thresholds

You already have a routing threshold from Module 2: auto-route if `top_prob ≥ 0.80`. You can lower the escalation threshold for high-FNR segments:

```
Default escalation threshold:       top_prob ≥ 0.80
Night-shift override:               top_prob ≥ 0.65
Non-native English override:        top_prob ≥ 0.70
```

This reduces missed security incidents at the cost of more human reviews from those segments. That is the correct trade-off.

### Medium-term: rebalance training data

Night-shift tickets were 10% of training data. Oversample them, or collect more labelled examples specifically from that segment, before the next model update.

### Longer-term: audit the proxy risk

Before assuming the fix is purely about data volume, run the interpretability analysis (Module 14). If the words driving `account_unlock` classification in night-shift tickets are structurally different from those driving it in HQ tickets, you have a vocabulary problem. If they're similar words but lower weights, you have a data volume problem. The interventions are different.

---

## Plugging back into the Monday ticket

The Monday ticket was submitted at 9:47 AM by an HQ employee. It fell into the high-FNR segment for a different reason — content ambiguity (Module 4), not segment membership. But the pipeline now runs a segment check alongside the entropy check:

```
[M13] Segment check
    → submitter: HQ, standard hours
    → FNR for this segment: 0.053  (below 0.15 alert threshold)
    → no threshold override triggered
    → proceed with default thresholds
```

If the same ticket had arrived at 2 AM from a night-shift worker:

```
[M13] Segment check
    → submitter: night-shift
    → FNR for this segment: 0.455  (above 0.15 alert threshold)
    → escalation threshold lowered: 0.65
    → re-evaluate: top_prob = 0.576 still below 0.65
    → decision unchanged: clarification queue
    → BUT: flag for analyst that this is a high-FNR segment
```

The decision stays the same in this case — the ticket is ambiguous enough to need human review either way. But the analyst now sees a flag: *this submitter profile has a high miss rate for security incidents. Give the patch notice mention extra weight.*

That flag is Module 13 working correctly.

---

## The updated routing policy

From Modules 2–4, the routing rules were:

```
if top_prob ≥ 0.80 AND margin ≥ 0.60 AND H_norm ≤ 0.45 → auto-route
elif H_norm ≥ 0.80                                       → escalate immediately
else                                                     → clarification queue
```

Module 13 adds a layer before that logic runs:

```
[M13 pre-check]
if segment_FNR(submitter) ≥ 0.15:
    lower escalation threshold by 0.15
    flag ticket: "high-FNR segment — analyst: apply extra scrutiny"

then → apply standard routing rules with adjusted thresholds
```

---

## The full pipeline, updated

```
Ticket arrives
    │
    ▼
[M01] Tokenize
    │
    ▼
[M03] Logits → Softmax
    │
    ▼
[M13] Segment check  ← NEW
    → identify submitter segment
    → look up historical FNR for that segment
    → adjust thresholds if FNR ≥ 0.15
    → attach segment flag to ticket record
    │
    ▼
[M02] Probability check
    │
    ▼
[M03] Margin check
    │
    ▼
[M04] Entropy check → routing decision
    │
    ▼
[M05] Stability gate (deploy-time)
    │
    ▼
[M06] Deterministic routing logged
    │
    ▼
[M06] Stochastic draft generation
    │
    ▼
[M07] Resolution time estimate
    │
    ▼
Analyst reviews — with segment flag visible if applicable
Outcome logged → feeds residual tracking AND segment FNR update
```

---

## Checklist before shipping a model update

- [ ] Have you disaggregated accuracy by every identifiable segment (department, shift, language, role)?
- [ ] Have you computed FNR for security incidents — not just overall accuracy — per segment?
- [ ] Have you identified which fairness criterion applies (calibration, equalised odds, parity) and documented why?
- [ ] Have you checked whether any segment gap might be a proxy for an attribute you didn't intend to use?
- [ ] Have you set segment-specific thresholds where FNR exceeds your tolerance?
- [ ] Is the segment flag visible to analysts at review time?
- [ ] Are outcomes feeding back into per-segment FNR tracking so you catch drift?

---

> One number — 91.4% — made the system look fine. Fourteen numbers told a different story.
>
> Module 14 will go one level deeper: not just *which segments* are being misclassified, but *which words and phrases* are driving those decisions. That's where you find out whether the night-shift gap is a data problem, a vocabulary problem, or something harder to fix.

--8<-- "_abbreviations.md"
