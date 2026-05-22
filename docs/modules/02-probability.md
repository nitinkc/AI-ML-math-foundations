# Module 2: Probability Refresher for AI

## What does "0.72 probability" actually mean?

Picture a traffic light. It doesn't tell you *whether* an accident will happen — it tells you 
*how much to trust* the current flow of traffic before you step on the gas.

Model probabilities work the same way.

When your classifier returns this for a service desk ticket:

```
account_unlock:    0.60
vpn_issue:         0.25
security_incident: 0.15
```

It is **not** saying: *"I am 60% sure this is an account unlock."*

It is saying: *"Given everything I've learned, account_unlock fits the pattern better than the other options — and here's how much better."*

The model is expressing **relative preference**, not absolute truth. That distinction will save you from bad routing decisions.

---

## The two rules that can never break

No matter what model you use, no matter what the ticket says, these two rules must always hold:

$$
\sum_{i=1}^{n} P(i) = 1 \qquad \text{and} \qquad 0 \le P(i) \le 1
$$

**In plain English:**
1. All probabilities across all classes must add up to exactly 1 (100%)
2. No probability can be negative or greater than 1

**Why you should care operationally:** If your model outputs `[0.60, 0.25, 0.30]` — that sums to 1.15, not 1.0. That's a red flag. Either the output wasn't normalized, something broke in post-processing, or you're reading raw logits instead of probabilities. Any downstream routing logic built on that is already wrong.

> **Quick sanity check to run in every pipeline:**
> ```python
> assert abs(sum(probs.values()) - 1.0) < 1e-6, "Probabilities don't sum to 1!"
> ```

---

## The formula that answers "how much time will this cost us?"

Once you have valid probabilities, you can calculate **expected value** — what you should *plan for* on average across many tickets.

$$
\mathbb{E}[X] = \sum_{i=1}^{n} x_i \cdot P(i)
$$

Where $x_i$ is the cost (in minutes, dollars, headcount) for handling intent $i$.

### Real example: staffing your Monday morning queue

Say your model has classified today's incoming 500 tickets with these probabilities:

| Intent | Probability | Avg. Handle Time |
|---|---|---|
| `account_unlock` | 0.60 | 4 min |
| `vpn_issue` | 0.25 | 18 min |
| `security_incident` | 0.15 | 45 min |

Expected handle time per ticket:

$$
\mathbb{E}[X] = (0.60 \times 4) + (0.25 \times 18) + (0.15 \times 45)
$$
$$
= 2.4 + 4.5 + 6.75 = \textbf{13.65 minutes per ticket}
$$

500 tickets × 13.65 min = **6,825 minutes = ~114 agent-hours needed today.**

Now you're not guessing at staffing. You're planning from data.

---

## The routing decision: when to act, when to pause

Back to that same ticket with these scores:

```
account_unlock:    0.60  ← top class
vpn_issue:         0.25
security_incident: 0.15
```

Your policy says: **auto-route only if the top class hits 0.80 confidence.**

This ticket fails. Here's why that matters:

The gap between `account_unlock` (0.60) and `vpn_issue` (0.25) is only 0.35. That's not decisive — it's uncertain. If you auto-route to account unlock and it's actually a VPN issue, you've just sent the ticket to the wrong team, wasted a handler's time, and frustrated the user.

The correct response is to **route to clarification** — ask one confirming question, or send to a human reviewer — before taking action.

### The confidence ladder (build this into your system)

| Top class score | What it signals | What to do |
|---|---|---|
| ≥ 0.80 | High confidence | Auto-route to specialist queue |
| 0.60 – 0.79 | Moderate confidence | Route with a flag: "verify before acting" |
| 0.40 – 0.59 | Low confidence | Send to clarification queue |
| < 0.40 | No clear winner | Escalate to human immediately |

The exact thresholds are yours to tune — but **the ladder structure is non-negotiable.** A system with no middle tier will either over-automate (and make expensive mistakes) or under-automate (and defeat the point).

---

## The trap that catches everyone

> *"0.82 confidence? That's basically confirmed. Route it."*

No. Here's the mental model to replace that instinct:

Imagine you flip a weighted coin that lands heads 82% of the time. If you route 1,000 tickets at 0.82 confidence, **180 of them are wrong.** At 100 tickets a day, that's 18 misrouted tickets daily — silently, with no error message, no alert, no warning.

This is why you need:

1. A **threshold policy** — written down, agreed upon, not just in someone's head
2. A **fallback behavior** — what exactly happens when confidence is too low
3. **Monitoring** — track whether your model's confidence distribution drifts week-over-week

If your average top-class confidence was 0.78 last month and it's 0.61 this month, something changed. Either your ticket language shifted, or your model needs retraining.

---

## How to practice this

Open `notebooks/math-foundations/02_probability.ipynb` and work through:

1. Convert raw model scores (logits) into valid probabilities that sum to 1
2. Spot deliberately broken outputs — probabilities that violate the two rules
3. Calculate expected handle time for a sample Monday queue
4. Tune confidence thresholds and watch how your misroute rate changes

---

## What to lock in before moving on

- [ ] Can you explain why 0.72 probability ≠ "72% confirmed correct"?
- [ ] Do your pipeline's output probabilities always sum to 1? (add the assert check)
- [ ] Is your threshold policy written down with explicit fallback behavior for each tier?
- [ ] Are ambiguous tickets (0.40–0.60) going somewhere useful, not just silently dropped?
- [ ] Are you tracking confidence distribution drift week to week?

--8<-- "_abbreviations.md"