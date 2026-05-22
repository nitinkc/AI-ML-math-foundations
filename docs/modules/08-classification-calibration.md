# Module 8: Classification and Calibration

## What's the big idea?

Classification is straightforward: predict a category with a confidence number.

Calibration is the hard part: **Does that confidence number actually mean what it says?**

Let's say your model says "I'm 80% confident" on 100 different tickets. In a perfect world, about 80 of those predictions are actually correct. But what if only 60 are? Or 95? That's a problem.

When predictions drift from reality, your thresholds become unreliable, and your decisions backfire.

## The math in plain terms

Here's how you make a decision based on a threshold:

$$
\hat{y} =
\begin{cases}
\text{yes/auto-route}, & p \ge \tau \\
\text{no/escalate}, & p < \tau
\end{cases}
$$

- $p$ is your predicted confidence 
- $\tau$ (tau) is your decision threshold

**The calibration question:** When you set $\tau = 0.80$, and you auto-route 100 tickets, are about 80 of them actually correct?

## Real-world scenario: Incident auto-escalation

You're handling security tickets. You set different thresholds for different risks:

- **Low-risk category (e.g., password reset):** High threshold (0.95). Only auto-action if super confident. Fewer false alarms.
- **High-risk category (e.g., potential breach):** Lower threshold (0.70). Flag more aggressively. Miss fewer incidents.

Over weeks, you monitor: "Of tickets at 0.70 confidence we flagged as security, what % actually were?" If it's not ~70%, your threshold is off. Time to recalibrate.

## How to try this

Run `notebooks/math-foundations/08_classification_calibration.ipynb` and:

- See how different thresholds affect precision and recall
- Build a table: predicted confidence vs actual accuracy rate
- Compare outcomes under stricter vs looser policies

## The traps to avoid

❌ **The trap:** One global threshold for all risk classes  
✅ **The smart move:** Use class-specific thresholds AND recalibrate every month

## Checklist for you

- [ ] Is each threshold tied to your business risk tolerance?
- [ ] Do you monitor precision/recall by class, not just overall?
- [ ] Do you recalibrate on fresh production-like data?
- [ ] Are low-confidence outputs routed to safe fallback paths?

--8<-- "_abbreviations.md"
