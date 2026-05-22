# Module 8: Classification and Calibration

## The routing pipeline is working — but can you trust the numbers?

By now your Monday ticket has made it through the full pipeline:

```
top_prob = 0.576   margin = 0.190   H_norm = 0.74
Decision: clarification queue (didn't clear the 0.80 threshold)
```

The system behaved correctly. But your security lead asks a different question:

> *"When your model says 0.80 confidence on a security incident —
> how often is it actually right?"*

You don't know. And that's the problem this module solves.

**Classification** is the act of predicting a category with a confidence number.
**Calibration** is whether that confidence number means what it claims to mean.

A model that says "0.80 confidence" on 100 tickets should be right on roughly
80 of them. If it's only right on 55 — your threshold policy from Modules 2–4
is built on a lie. You're routing based on confidence scores that don't reflect
reality.

---

## What miscalibration looks like in practice

Your team pulls 300 historical tickets where the model returned confidence scores
between 0.75 and 0.85 for `security_incident`. You check how many were actually
security incidents after human review:

```
Model said: "~0.80 confident this is a security incident"
Actual result: 156 out of 300 were security incidents = 52% accuracy
```

The model's confidence said 80%. Reality was 52%.

That 28-point gap means your 0.80 threshold is not doing what you think it's doing.
Every time you auto-escalate a ticket at 0.80 security confidence, you're actually
working with a coin flip dressed up as a confident decision.

Now look at the `account_unlock` class over the same period:

```
Model said: "~0.80 confident this is account_unlock"
Actual result: 247 out of 300 were account_unlocks = 82% accuracy
```

The model is well-calibrated for account unlocks. It's badly miscalibrated for
security incidents. Same confidence number. Completely different meaning depending
on which class you're looking at.

**This is why you need class-specific calibration — not just one global check.**

---

## The math of the threshold decision

Every classification decision boils down to:

$$
\hat{y} =
\begin{cases}
\text{auto-route / escalate}, & p \ge \tau \\
\text{human review / hold}, & p < \tau
\end{cases}
$$

- $p$ — the model's predicted confidence for the top class
- $\tau$ (tau) — your decision threshold

The threshold $\tau$ is not a magic number. It's a business decision that encodes
your tolerance for two types of error:

**False positive** — you acted when you shouldn't have.
For `account_unlock`: you reset someone's password when they didn't need it. Minor inconvenience.

**False negative** — you didn't act when you should have.
For `security_incident`: you sent a potential breach to the clarification queue
instead of the security team. Active attacker stays on network.

The asymmetry is stark. For security incidents, a false negative is far more
dangerous than a false positive. So you set a *lower* threshold — flag more
aggressively, accept more false alarms — to catch more real incidents.

---

## Class-specific thresholds for the Monday ticket's three intents

| Class | Risk of false negative | Risk of false positive | Threshold $\tau$ |
|---|---|---|---|
| `account_unlock` | Low — slightly delayed reset | Low — unnecessary reset | 0.80 |
| `vpn_issue` | Medium — user stays offline | Low — extra network check | 0.72 |
| `security_incident` | **Critical** — attacker stays in | Medium — wasted senior engineer time | **0.60** |

Your Monday ticket had `security_incident` at only **0.039** — well below even
the aggressive 0.60 threshold. No escalation triggered. Correct call.

But now imagine a different ticket arrives with:

```
account_unlock:    0.45
vpn_issue:         0.21
security_incident: 0.34
```

Under a global 0.80 threshold: nothing auto-routes, everything goes to clarification.
Under class-specific thresholds: `security_incident` at 0.34 is below 0.60 —
still human review. But the security team gets flagged as a *watch* case, not ignored.

One system protects you. The other creates a blind spot at exactly the risk level
that matters most.

---

## The calibration check: building a reliability table

To know if your model is calibrated, bin your historical predictions by confidence
range and compare to actual accuracy:

| Confidence bucket | Tickets in bucket | Actually correct | Calibrated? |
|---|---|---|---|
| 0.90 – 1.00 | 412 | 89% correct | ✅ Well calibrated |
| 0.80 – 0.89 | 389 | 82% correct | ✅ Well calibrated |
| 0.70 – 0.79 | 310 | 71% correct | ✅ Well calibrated |
| 0.60 – 0.69 | 287 | **52% correct** | ❌ Miscalibrated |
| 0.50 – 0.59 | 203 | **61% correct** | ❌ Inverted — more accurate than stated |

Two problems visible here:

1. **The 0.60–0.69 bucket is badly overconfident.** The model says 65% but delivers 52%.
   Any threshold you set in this range is unreliable.

2. **The 0.50–0.59 bucket is underconfident.** The model says 55% but delivers 61%.
   Tickets being sent to human review here are actually better than the model admits.

Now break this down **by class**. The table above is aggregate — it hides the fact
that `account_unlock` may be perfectly calibrated while `security_incident` is the
source of all the miscalibration:

```
security_incident, confidence 0.60–0.69:  actual accuracy = 38%  ❌ Very overconfident
account_unlock,    confidence 0.60–0.69:  actual accuracy = 71%  ✅ Fine
vpn_issue,         confidence 0.60–0.69:  actual accuracy = 58%  ⚠️  Slightly off
```

Your threshold policy should be set against the *class-specific* calibration table,
not the aggregate one. The aggregate table was hiding a serious problem in the
highest-stakes class.

---

## Connecting to entropy: calibration and uncertainty agree

In Module 4, high entropy (H_norm = 0.74) told you the model was uncertain about
the Monday ticket. Calibration adds a second layer of evidence:

- Entropy measures uncertainty *within a single prediction* (how spread is this distribution?)
- Calibration measures reliability *across many predictions* (does 0.80 mean 80% correct historically?)

When both signals point the same direction, you can be more confident in your routing decision.
When they contradict — high confidence but poor historical calibration for that class —
entropy is the safer signal to trust in the moment.

For `security_incident` specifically: the model is miscalibrated in the 0.60–0.69 range
(actual accuracy 38%) AND tends to produce high-entropy outputs when security is involved.
Two independent reasons to route conservatively. Both were already firing on the Monday ticket.

---

## Recalibration: how often and with what data

Calibration drifts. The ticket language your users write in March is different from January.
A major patch rollout (like the one that triggered the Monday ticket) can shift the
distribution enough to break calibration built on pre-patch data.

**Recalibration schedule:**

```
Monthly:   rebuild the reliability table from the last 30 days of production tickets
Triggered: any time a major infrastructure change ships (patches, new software, org changes)
Triggered: any time residual drift is detected in the Module 7 regression monitoring
```

**What data to use:**

Only tickets where you have a verified ground truth — cases a human reviewed and
confirmed the correct intent. Auto-routed tickets can't be used for calibration
because you don't know if the routing was right (that's circular).

**Minimum sample size per class per confidence bucket:** 50 tickets.
Below that, your calibration table is noise. The `security_incident` class
will be the hardest to populate — it's the rarest class and the one you most
need accurate calibration for. Prioritise getting human-reviewed labels on
every security ticket that comes through.

---

## The updated routing policy

Bringing calibration into the policy from Module 4:

```python
# Per-class thresholds (set from calibration table, not guesswork)
thresholds = {
    "account_unlock":    0.80,
    "vpn_issue":         0.72,
    "security_incident": 0.60,
}

# Routing logic
top_class = argmax(probs)          # deterministic (Module 6)
p = probs[top_class]
tau = thresholds[top_class]

if p >= tau AND margin >= 0.30 AND H_norm <= 0.45:
    route_to_specialist(top_class)
elif top_class == "security_incident" AND p >= 0.34:
    flag_as_security_watch()       # lower watch threshold for highest-risk class
elif H_norm >= 0.80:
    escalate_immediately()
else:
    send_to_clarification_queue()
```

Applied to the Monday ticket:
```
top_class = account_unlock,  p = 0.576,  tau = 0.80
0.576 < 0.80 → does not clear threshold
security_incident p = 0.039 < 0.34 → no watch flag
H_norm = 0.74 < 0.80 → no immediate escalation
→ clarification queue  ✅ (same decision as before, now with class-aware logic)
```

The routing decision didn't change — but the reasoning is now calibration-aware.
If `security_incident` had come in at 0.38 instead of 0.039, the watch flag
would have fired even though the overall routing went to clarification.

---

## How to practice this

Open `notebooks/math-foundations/08_classification_calibration.ipynb` and work through:

1. Build the aggregate reliability table from provided historical ticket data —
   identify which confidence buckets are miscalibrated
2. Break the table down by class — find which class is driving the miscalibration
3. Set class-specific thresholds based on the calibration results and your risk tolerance
4. Simulate how the Monday ticket would route under the old global threshold vs
   the new class-specific policy — does the security watch flag fire on any variants?

---

## Checklist before moving on

- [ ] Have you built a reliability table broken down by class, not just in aggregate?
- [ ] Are your thresholds set from calibration data — not chosen arbitrarily?
- [ ] Is `security_incident` (or your highest-stakes class) getting enough human-reviewed labels to calibrate on?
- [ ] Do you recalibrate after major infrastructure changes, not just on a fixed calendar?
- [ ] When calibration and entropy disagree, do you have a rule for which to trust?

--8<-- "_abbreviations.md"