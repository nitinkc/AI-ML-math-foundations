# Module 3: Logits and Softmax

## The problem with raw scores

Cast your mind back to Monday morning's ticket:

> *"Hi, I've been locked out of my account... Also my VPN keeps dropping... This happened
> right after the security team sent out that patch notice last Friday."*

Your model doesn't immediately output `0.55 / 0.30 / 0.15`. Before those clean
probabilities exist, the model produces something messier — raw scores called **logits**:

```
account_unlock:    3.2
vpn_issue:         2.8
security_incident: 0.5
```

Logits are "preference votes." A higher number means the model leans that way more
strongly — but there's no scale, no ceiling, no guarantee they sum to anything useful.
A logit of 3.2 doesn't mean 3.2%. It just means *more preferred than 2.8*.

To get the probabilities you actually route with, you need to convert them.
That conversion is called **Softmax**.

---

## What Softmax does (and why it's not magic)

The formula:

$$
P(i) = \frac{e^{z_i}}{\sum_{j=1}^{n} e^{z_j}}
$$

In plain English: exponentiate every logit, then divide each one by the total.
The result is a set of numbers that all sit between 0 and 1, and sum to exactly 1.

Let's run this on our Monday ticket's logits `[3.2, 2.8, 0.5]`:

**Step 1 — Exponentiate each logit:**
```
e^3.2 = 24.53
e^2.8 = 16.44
e^0.5 =  1.65
        ──────
Total = 42.62
```

**Step 2 — Divide each by the total:**
```
account_unlock:    24.53 / 42.62 = 0.576  (~58%)
vpn_issue:         16.44 / 42.62 = 0.386  (~39%)
security_incident:  1.65 / 42.62 = 0.039  ( ~4%)
```

Now they sum to 1.0 ✅

Notice something important: the logit gap between `account_unlock` and `vpn_issue`
was only **0.4** (3.2 vs 2.8), but after softmax that translates to a probability
gap of **0.19** (0.576 vs 0.386). The probabilities are still uncomfortably close —
nowhere near your 0.80 auto-route threshold. This ticket is genuinely ambiguous.

---

## What happens when the logits are decisive?

Now imagine the ticket said only: *"Reset my password please."*
No VPN mention. No patch notice. Logits might look like:

```
account_unlock:    5.0
vpn_issue:         1.2
security_incident: -0.4
```

Run the same softmax:
```
e^5.0  = 148.41
e^1.2  =   3.32
e^-0.4 =   0.67
           ──────
Total  = 152.40

account_unlock:    148.41 / 152.40 = 0.974  (97%)
vpn_issue:           3.32 / 152.40 = 0.022  ( 2%)
security_incident:   0.67 / 152.40 = 0.004  (<1%)
```

A logit gap of **3.8** produced a probability gap of **0.95**. Safe to auto-route.

**The pattern to memorise:** Large logit gaps → peaked distributions → confident routing.
Small logit gaps → flat distributions → send to human or clarification queue.

---

## The temperature dial: sharpening or softening the distribution

You can adjust how decisive the softmax output is by adding a **temperature** parameter $T$:

$$
P_T(i) = \frac{e^{z_i / T}}{\sum_{j=1}^{n} e^{z_j / T}}, \quad T > 0
$$

Think of temperature like a volume knob on the model's confidence:

| Temperature | Effect | When to use it |
|---|---|---|
| $T = 1.0$ | Default — no change | Normal routing decisions |
| $T < 1.0$ (e.g. 0.3) | Sharpens the winner — more decisive | When you need crisp single-intent routing |
| $T > 1.0$ (e.g. 2.0) | Flattens the distribution — more options | When generating diverse suggestions |

### Applied to our Monday ticket (logits `[3.2, 2.8, 0.5]`):

**At T = 0.3 (low temperature):**
```
account_unlock:    0.91
vpn_issue:         0.09
security_incident: 0.00
```
Looks decisive now — but nothing changed in the underlying evidence.
The model didn't get smarter. You just turned up the confidence dial artificially.

**At T = 2.0 (high temperature):**
```
account_unlock:    0.42
vpn_issue:         0.38
security_incident: 0.20
```
Even flatter. Security incident is now 20% — nearly indistinguishable from the others.

**The critical rule:** Low temperature creates the *appearance* of certainty, not real certainty.
If you use it to push an ambiguous ticket past your routing threshold, you're hiding
the ambiguity, not resolving it. For our ticket — where a misrouted security incident
means a potential attacker stays on the network — that's a dangerous move.

---

## The metric that matters more than the top score

Stop looking at just the top probability. Start tracking the **confidence margin**:

$$
\text{margin} = P(\text{top-1}) - P(\text{top-2})
$$

For our Monday ticket: `0.576 - 0.386 = 0.190` — a margin of only 0.19.
For the "Reset my password" ticket: `0.974 - 0.022 = 0.952` — a margin of 0.95.

A routing policy built on margin is more robust than one built on threshold alone:

| Margin | Routing decision |
|---|---|
| ≥ 0.60 | Auto-route to specialist |
| 0.30 – 0.59 | Route with human-review flag |
| < 0.30 | Clarification queue or escalate |

Our Monday ticket has a margin of 0.19 — it goes straight to escalation.
No amount of temperature tuning changes that. The evidence is genuinely split.

---

## One implementation trap: numerical instability

Exponentiating large logits causes overflow. `e^1000` crashes most systems.

The fix is one line — subtract the max logit before exponentiating:

```python
# ❌ Naive — breaks on large logits
probs = np.exp(logits) / np.sum(np.exp(logits))

# ✅ Numerically stable — always use this
logits_shifted = logits - np.max(logits)
probs = np.exp(logits_shifted) / np.sum(np.exp(logits_shifted))
```

The math is identical (subtracting a constant from all logits doesn't change the
ratios), but it prevents your pipeline from silently producing `NaN` on edge-case tickets.

---

## How to practice this

Open `notebooks/math-foundations/03_logits_softmax.ipynb` and work through:

1. Convert the Monday ticket's logits `[3.2, 2.8, 0.5]` to probabilities by hand, then verify with code
2. Run the same logits at T = 0.3, 1.0, and 2.0 — watch what happens to the margin
3. Find the logit gap threshold where auto-routing becomes safe for your policy
4. Add the numerically stable softmax to your pipeline and test it on extreme logit values

---

## Checklist before moving on

- [ ] Can you explain why a logit of 3.2 is not the same as 32% confidence?
- [ ] Do you track the margin (top-1 minus top-2), not just the top score?
- [ ] Is your softmax implementation numerically stable (max subtraction)?
- [ ] Do you know what temperature setting your system uses — and why?
- [ ] Is your routing policy validated against *real* outcomes, not just held-out test data?

--8<-- "_abbreviations.md"