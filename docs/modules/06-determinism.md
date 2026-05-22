# Module 6: Deterministic vs Stochastic Behavior

## A tale of the same ticket, run twice

It's Tuesday. Your team re-runs Monday's ambiguous ticket through the system
to double-check the routing decision before a post-incident review:

> *"Hi, I've been locked out of my account... Also my VPN keeps dropping...
> This happened right after the security team sent out that patch notice last Friday."*

**First run:**
```
Routed to: account_unlock queue
Suggested response: "Hi, I've raised a password reset request for your account.
                    You should receive an email within 5 minutes."
```

**Second run, same ticket, 10 minutes later:**
```
Routed to: vpn_issue queue
Suggested response: "Thanks for reaching out. Can you confirm which VPN client
                    you're using and whether this started after Friday's update?"
```

Different queue. Different response. Same ticket.

Your security lead asks: "Which one was right?" You can't answer — because you
don't know if the difference is meaningful signal or random noise.

This is the core problem Module 6 solves. The question isn't whether your system
is deterministic or stochastic — **it's whether you made that choice deliberately,
and whether you applied it to the right parts of the pipeline.**

---

## Deterministic mode: same input, always same output

The deterministic rule is called **argmax** — always pick the class with the
highest probability, no randomness involved:

$$
\hat{i} = \arg\max_i \, P(i)
$$

Applied to Monday's ticket probabilities:
```
account_unlock:    0.576   ← argmax always picks this
vpn_issue:         0.386
security_incident: 0.039
```

Run it 1,000 times. You get `account_unlock` 1,000 times.

**What you gain:** Consistency. Auditability. Traceability. If someone asks
"why did ticket #4821 go to the account unlock queue on Monday?", you can
replay the exact inputs and get the exact same answer.

**What you give up:** Any chance of catching cases where the second-ranked
class was actually correct — even when the margin was only 0.19.

---

## Stochastic mode: same input, different outputs each time

In stochastic mode, the system doesn't always pick the top class. Instead,
it **samples** — drawing from the probability distribution the way a weighted
die works. Each class gets picked proportional to its probability.

For Monday's ticket:
```
account_unlock:    picked ~57.6% of runs
vpn_issue:         picked ~38.6% of runs
security_incident: picked  ~3.9% of runs
```

Run it 10 times and you might see:
```
Run 1:  account_unlock
Run 2:  account_unlock
Run 3:  vpn_issue        ← different outcome
Run 4:  account_unlock
Run 5:  vpn_issue        ← different outcome again
Run 6:  account_unlock
Run 7:  account_unlock
Run 8:  vpn_issue
Run 9:  account_unlock
Run 10: account_unlock
```

**What you gain:** Variety. When generating response drafts, that variety is
the entire point — you want three meaningfully different phrasings, not the
same sentence three times.

**What you lose:** Consistency. Which means you lose auditability. And in
compliance-critical workflows, that's not a trade-off you can make.

---

## Temperature is the dial between the two

Recall from Module 3: temperature $T$ controls how peaked or flat your
probability distribution is before sampling.

$$
P_T(i) = \frac{e^{z_i / T}}{\sum_{j} e^{z_j / T}}
$$

Now think about what that does to stochastic sampling:

**At T → 0 (approaching deterministic):**
```
account_unlock:    ~1.000
vpn_issue:         ~0.000
security_incident: ~0.000
```
Sampling from this almost always picks `account_unlock`. Effectively deterministic.

**At T = 1.0 (default stochastic):**
```
account_unlock:    0.576
vpn_issue:         0.386
security_incident: 0.039
```
Sampling gives genuine variety — `vpn_issue` comes up roughly 4 in 10 runs.

**At T = 2.0 (high randomness):**
```
account_unlock:    0.42
vpn_issue:         0.38
security_incident: 0.20
```
Now `security_incident` shows up 1 in 5 runs. You're pulling responses from
a nearly flat distribution — mostly noise, very little signal.

The practical takeaway: **temperature isn't just a creativity dial.
It's a consistency dial. Turning it up in a routing step introduces the
exact re-run inconsistency we saw at the top of this module.**

---

## The two jobs of the IT copilot — and why they need different modes

Your copilot does two fundamentally different things with the Monday ticket:

### Job 1 — Intent routing (must be deterministic)

The ticket goes to one queue. One decision. Permanent record.

If your routing step uses stochastic sampling, you get the Tuesday re-run
problem: different answer each time, no way to audit, no way to defend the
decision to a security lead or a compliance officer.

```
Routing step settings:
  mode:        deterministic (argmax)
  temperature: not applicable — argmax ignores temperature
  seed:        N/A
  logged:      queue assignment + top probability + margin + H_norm
```

### Job 2 — Draft response suggestions (stochastic is the point)

The analyst sees 3 suggested first-line responses. They pick one, edit it,
send it. Variety here is a feature — if all 3 drafts are identical,
you've wasted the analyst's time and the model's compute.

```
Draft generation settings:
  mode:        stochastic (sampling)
  temperature: 0.8 – 1.2  (enough variety, not pure noise)
  seed:        logged per request (so drafts can be reproduced if needed)
  logged:      seed value + temperature + all 3 draft outputs
```

The pattern: **deterministic for decisions, stochastic for options.**

---

## Connecting back to variance (Module 5)

Remember Config B from last module — quality scores oscillating between
2.7 and 4.9 day to day, same average as Config A?

Now you have the vocabulary to diagnose it. That oscillation is almost
certainly stochastic behaviour bleeding into a step that should be deterministic.

Run the diagnostic:

```
1. Fix the random seed across all 30 days of Config B runs.
   If variance drops → stochastic sampling was the cause.
   Fix: switch routing step to deterministic mode.

2. If variance stays high with a fixed seed → something else is varying.
   Likely cause: context window truncation (Module 1) or temperature > 0.
   Fix: audit what's actually changing in the input between runs.
```

High variance in a deterministic step is a red flag that the step
isn't actually deterministic. Always verify, never assume.

---

## Reproducibility: the seed contract

When you do need stochastic outputs (draft suggestions, creative rewriting,
generating example tickets for testing), you must log the random seed.

```python
import random
import numpy as np

SEED = 42  # logged with every request

random.seed(SEED)
np.random.seed(SEED)

# Now your stochastic sampling is reproducible on demand
drafts = sample_responses(ticket, n=3, temperature=1.0)
```

The seed contract: **if anyone asks to reproduce a past output,
you can replay the exact seed and get the exact same drafts.**

Without this, your stochastic outputs are forensically invisible.
For a security incident ticket, that's unacceptable — you need to be
able to reconstruct exactly what suggestions your system made and why.

---

## The full pipeline view

Here's how Monday's ticket flows through a correctly configured system:

```
Ticket arrives
    │
    ▼
[DETERMINISTIC] Intent classification
    → argmax over softmax probabilities
    → logged: intent, top_prob, margin, H_norm
    → if H_norm ≥ 0.80: escalate immediately (Module 4 rule)
    │
    ▼
[DETERMINISTIC] Routing decision
    → applies threshold + margin + entropy gates (Modules 2–4)
    → logged: queue assignment, gate values, timestamp
    │
    ▼
[STOCHASTIC] Draft response generation   ← only this step uses sampling
    → temperature: 1.0, seed: logged
    → generates 3 response options for analyst
    → logged: seed, temperature, all 3 outputs
    │
    ▼
Analyst reviews, edits, sends
```

Every decision step is deterministic and auditable.
The one creative step is stochastic but seeded and logged.
Nothing is left to chance without a record.

---

## How to practice this

Open `notebooks/math-foundations/06_determinism.ipynb` and work through:

1. Run the Monday ticket through argmax vs sampling 20 times each — compare the output distributions
2. Run sampling at T = 0.3, 1.0, 2.0 — measure how often the second-ranked class gets picked
3. Add seed logging to a stochastic step and verify you can reproduce any past output exactly
4. Identify which steps in a provided sample pipeline are incorrectly using stochastic mode
   where deterministic is required

---

## Checklist before moving on

- [ ] Can you identify which steps in your pipeline must be deterministic and which can be stochastic?
- [ ] Is your routing step using argmax — not temperature-based sampling?
- [ ] Are all stochastic steps logging their seed and temperature settings?
- [ ] If asked to reproduce any past output, can you do it?
- [ ] Have you checked whether high variance in any step is caused by accidental stochastic behaviour?

--8<-- "_abbreviations.md"