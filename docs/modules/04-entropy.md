# Module 4: Entropy and Uncertainty

[https://medium.com/@quantclubiitkgp/entropy-and-its-application-in-stock-market-accd880ef241](https://medium.com/@quantclubiitkgp/entropy-and-its-application-in-stock-market-accd880ef241)

## The problem with only looking at the top score

In Module 3, we left our Monday ticket here:

```
account_unlock:    0.576
vpn_issue:         0.386
security_incident: 0.039
```

Margin = 0.19 — too close to auto-route. We sent it to escalation.

Now consider a different ticket that just came in — an outage report:

```
account_unlock:    0.58
vpn_issue:         0.21
security_incident: 0.21
```

The top score is almost identical (0.58 vs 0.576). The margin is similar (0.37).
By the routing rules from Module 3, both tickets look roughly the same.

But look more carefully. In the second ticket, **two classes are tied at 0.21.**
The model isn't just uncertain between first and second — it's equally lost between
second and third. The uncertainty is *spreading*, not concentrating.

Your margin metric can't see this. You need a new tool: **entropy**.

---

## What entropy measures

Entropy is a single number that captures how *spread out* a probability distribution is.

$$
H(P) = -\sum_{i=1}^{n} P(i) \log_2 P(i)
$$

Two anchor points to memorise:

**Entropy = 0** → all probability sits on one class. The model is certain.
```
[1.00, 0.00, 0.00]   →   H = 0
```

**Entropy = log₂(n)** → probability is split perfectly evenly. The model is maximally lost.
```
[0.33, 0.33, 0.33]   →   H = log₂(3) = 1.585 bits
```

Everything else falls between those two poles.

---

## Let's calculate it on the Monday ticket

**Ticket 1 — Monday's ambiguous ticket:**
```
P = [0.576, 0.386, 0.039]

H = -(0.576 × log₂0.576) + (0.386 × log₂0.386) + (0.039 × log₂0.039)
  = -(0.576 × -0.796)  + (0.386 × -1.374)  + (0.039 × -4.678)
  = 0.458 + 0.530 + 0.182
  = 1.17 bits
```

**Ticket 2 — The outage report:**
```
P = [0.58, 0.21, 0.21]

H = -(0.58 × log₂0.58) + (0.21 × log₂0.21) + (0.21 × log₂0.21)
  = -(0.58 × -0.786)  + (0.21 × -2.252)  + (0.21 × -2.252)
  = 0.456 + 0.473 + 0.473
  = 1.40 bits
```

Maximum possible entropy for 3 classes = log₂(3) = **1.585 bits**.

| Ticket | Top score | Margin | Entropy | Assessment |
|---|---|---|---|---|
| Monday ticket | 0.576 | 0.19 | 1.17 bits | Uncertain — escalate |
| Outage report | 0.580 | 0.37 | 1.40 bits | More uncertain — escalate faster |

The top scores are nearly identical. The margins are different but both low.
But entropy reveals that the outage report is *closer to maximum confusion*
than the Monday ticket — something the top score alone completely hides.

---

## Normalised entropy: comparing across different class counts

Raw entropy values only make sense within the same number of classes. A 3-class problem
maxes at 1.585 bits; a 10-class problem maxes at 3.32 bits. You can't compare them directly.

Normalise to a 0–1 scale:

$$
H_{norm} = \frac{H(P)}{\log_2 n}
$$

Now 0 always means "fully certain" and 1 always means "fully lost" — regardless of
how many classes your classifier uses.

| Ticket | Raw entropy | Normalised entropy |
|---|---|---|
| Monday ticket | 1.17 / 1.585 | **0.74** |
| Outage report | 1.40 / 1.585 | **0.88** |
| "Reset my password" (Module 3) | ~0.15 / 1.585 | **~0.09** |

The "Reset my password" ticket from Module 3 scores 0.09 — nearly certain.
Our Monday ticket scores 0.74 — well into uncertain territory.
The outage report scores 0.88 — close to maximum confusion. Flag it immediately.

---

## The combined routing policy

Here's why entropy needs to work *alongside* the confidence threshold from Module 2
and the margin metric from Module 3 — not instead of them.

Each metric catches a different failure mode:

| Failure mode | What it looks like | Which metric catches it |
|---|---|---|
| Model is unsure between top two | Top score 0.60, second 0.55 | **Margin** (Module 3) |
| Model spreads uncertainty widely | Top 0.58, others all similar | **Entropy** (this module) |
| Model looks confident but is often wrong | Top score 0.80, but historically 40% wrong | **Calibration** (future module) |

A robust routing policy uses all three:

```
if top_prob >= 0.80
   AND margin >= 0.60
   AND H_norm <= 0.45:
       → auto-route to specialist
elif H_norm >= 0.80:
       → escalate immediately (too confused to guess safely)
else:
       → clarification queue or human review
```

Applied to our running tickets:

| Ticket | Top prob | Margin | H_norm | Decision |
|---|---|---|---|---|
| "Reset my password" | 0.974 | 0.952 | 0.09 | ✅ Auto-route |
| Monday ticket | 0.576 | 0.190 | 0.74 | 🔁 Clarification queue |
| Outage report | 0.580 | 0.370 | 0.88 | 🚨 Escalate immediately |

Three different outcomes from three superficially similar top scores.
This is the difference between a routing system that looks right and one that *is* right.

---

## Why this matters for security incidents specifically

Return to the risk framing from the connecting scenario: misrouting a security incident
to a tier-1 agent doesn't just waste time — it leaves an active threat unaddressed.

Entropy is your last line of defence before auto-routing on ambiguous tickets.
A ticket with H_norm = 0.88 should **never** be auto-routed, regardless of what the
top class says. The model is telling you it genuinely doesn't know. Listen to it.

The instinct to override high entropy with a low temperature setting (from Module 3)
is exactly the wrong move here. Sharpening a confused distribution doesn't resolve
the confusion — it just makes it invisible until something breaks in production.

---

## How to practice this

Open `notebooks/math-foundations/04_entropy.ipynb` and work through:

1. Calculate raw and normalised entropy for all three running-scenario tickets by hand, then verify with code
2. Build the combined routing policy (top prob + margin + entropy) and test it on a sample queue
3. Find the H_norm threshold where your false-route rate drops to an acceptable level
4. Plot entropy distributions across a week of historical tickets — look for days where average entropy spiked

---

## Checklist before moving on

- [ ] Can you explain why two tickets with the same top score can have very different entropy?
- [ ] Are you calculating *normalised* entropy so it's comparable across different classifier sizes?
- [ ] Does your routing policy combine top probability, margin, *and* entropy — not just one?
- [ ] Do high-entropy tickets (H_norm ≥ 0.80) have an explicit fast-escalation path?
- [ ] Has someone tuned your entropy threshold against real historical outcomes, not just held-out test data?

--8<-- "_abbreviations.md"