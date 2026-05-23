# Module 7: Regression for AI Operations

## Classification answered "which queue?" — regression answers "how long?"

Every module so far has been about the same decision: where does the Monday ticket go?

You now have a routing pipeline that:
- Tokenizes the ticket (Module 1)
- Produces calibrated probabilities (Modules 2–3)
- Measures uncertainty with entropy (Module 4)
- Tracks output stability (Module 5)
- Logs deterministic decisions and stochastic drafts (Module 6)

The ticket gets routed correctly. But your team lead has a new question:

> *"Great — it went to the right queue. Now how long will it take to resolve?
> I need to know whether to assign it to someone who has 20 minutes free
> or someone who has 2 hours free."*

That's not a classification question. There's no "account_unlock" or "vpn_issue" bucket
for time. The answer is a number — and the tool for predicting numbers is **regression**.

---

## The intuition before the formula

Look at four weeks of resolved tickets from your service desk:

```
Ticket token count    Resolution time (minutes)
─────────────────     ─────────────────────────
 30  tokens            8  min   ← "Reset my password"
 60  tokens           22  min   ← Monday's ambiguous ticket (body only)
140  tokens           31  min
210  tokens           47  min
280  tokens           61  min
350  tokens           78  min
420  tokens           94  min
500  tokens          108  min
```

Plot these and they form a rough line: more tokens, more time. Not perfectly linear,
but close enough to be useful. Regression **finds the line that fits this pattern best**
and lets you predict resolution time for any new ticket.

---

## The math, piece by piece

The simplest regression model is:

$$
\hat{y} = \beta_0 + \beta_1 x
$$

- $x$ — your input, the thing you already know (token count of the incoming ticket)
- $\hat{y}$ — your prediction, the thing you want to estimate (resolution time in minutes)
- $\beta_1$ — the **slope**: how many extra minutes per additional token
- $\beta_0$ — the **intercept**: baseline time even for a near-empty ticket

The model learns $\beta_0$ and $\beta_1$ from historical data by minimising the
**residuals** — the gaps between what it predicted and what actually happened:

$$
\text{residual}_i = y_i - \hat{y}_i
$$

The fitting process (ordinary least squares) minimises the sum of squared residuals:

$$
\text{minimise} \sum_{i=1}^{n} (y_i - \hat{y}_i)^2
$$

Same squaring logic as variance in Module 5: penalise large misses more than small
ones, and eliminate the sign so positive and negative errors don't cancel out.

---

## Fitted to our service desk data

Running the regression on the table above gives approximately:

$$
\hat{y} = 2.1 + 0.21 \times x
$$

**Interpreting the coefficients:**

- $\beta_0 = 2.1$: even the simplest ticket needs at least 2 minutes (login, read, close)
- $\beta_1 = 0.21$: each additional token adds roughly **0.21 minutes** (~13 seconds)

**Applied to our running tickets:**

| Ticket | Token count | Predicted resolution | Actual (if known) | Residual |
|---|---|---|---|---|
| "Reset my password" | 30 tokens | 2.1 + (0.21 × 30) = **8.4 min** | 8 min | −0.4 min |
| Monday ticket (body only) | 60 tokens | 2.1 + (0.21 × 60) = **14.7 min** | 22 min | **+7.3 min** |
| Monday ticket (full context) | 220 tokens | 2.1 + (0.21 × 220) = **48.3 min** | — | — |

Notice the Monday ticket with full context (220 tokens) predicts 48 minutes —
nearly a full analyst hour. That's not just a routing call anymore.
That's a staffing decision.

---

## What the residual is telling you

The Monday ticket body-only residual is **+7.3 minutes** — the model predicted 14.7
but actual resolution took 22 minutes. The model underestimated.

Why? Because token count **captures *length*, not *complexity*.** The Monday ticket is
short but involves three ambiguous intents, a potential security angle, and a
time-sensitive client call. Those factors aren't in the token count.

This is the honest limitation of simple linear regression: **it can only see
what you put in as inputs.** A model that only sees token count will systematically
underestimate complex short tickets and overestimate verbose-but-simple ones.

The fix isn't to throw out regression — it's to add better input features:

```
Better model:
ŷ = β₀ + β₁(token_count) + β₂(H_norm) + β₃(top_prob) + β₄(intent_class)
```

You're already calculating H_norm and top_prob in your pipeline.
The ticket that stumped Modules 2–4 will now also flag as high-effort in Module 7.

---

## Multiple regression: using everything you already know

Extending to multiple inputs:

$$
\hat{y} = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \beta_3 x_3 + \cdots
$$

For the Monday ticket with the full feature set:

```
x₁ = token count:    220
x₂ = H_norm:         0.74   (high entropy — from Module 4)
x₃ = top_prob:       0.576  (moderate confidence — from Module 2)
x₄ = intent_class:   0      (account_unlock encoded as 0)

ŷ = 2.1 + (0.21 × 220) + (18.4 × 0.74) + (−12.3 × 0.576) + (3.5 × 0)
  = 2.1 + 46.2 + 13.6 − 7.1 + 0
  = 54.8 minutes
```

The high entropy term adds **13.6 minutes** to the estimate — the model has learned
that confused classifications (high H_norm) correlate with harder tickets.
The prediction jumps from 48.3 to 54.8 minutes just by including what you
already knew from the entropy calculation.

---

## The staffing decision this enables

Your team has three analysts available at 9:00am Monday:

```
Analyst A:  free for 25 minutes before a meeting
Analyst B:  free for 90 minutes
Analyst C:  free all morning, but handling two other open tickets
```

With a regression estimate of ~55 minutes for the Monday ticket:

- Analyst A is out — not enough runway
- Analyst C is out — already stretched
- Analyst B gets the ticket, with a heads-up that it's likely 45–65 minutes

Without the estimate, you're assigning blind. Someone gets stuck mid-ticket
when their calendar blocks. The user's 10am client call gets missed anyway.

**Regression turns a routing system into a capacity planning system.**

---

## The trap: mistaking prediction for cause

Your regression shows that high-entropy tickets (H_norm ≥ 0.74) take
on average 22 minutes longer to resolve. Someone on the team proposes:

> *"Let's just reduce entropy by lowering the temperature — then tickets
> will resolve faster."*

This is the classic correlation/causation mistake. High entropy doesn't
*cause* slow resolution — both are symptoms of ticket complexity. Hiding
the entropy signal (via temperature) doesn't make the ticket simpler.
It just means you stop measuring the symptom while the underlying problem persists.

The regression model is a forecasting instrument, not a control lever.
Use it to anticipate. Don't use it to manipulate the inputs.

---

## Monitoring after deployment: residuals over time

After you deploy the regression model, track the average residual week by week:

```
Week 1:  avg residual = +1.2 min   (slight underestimate — acceptable)
Week 2:  avg residual = +2.8 min   (growing — worth watching)
Week 3:  avg residual = +6.4 min   (model is drifting — investigate)
Week 4:  avg residual = +11.1 min  (model is broken — retrain now)
```

Growing positive residuals mean the model is consistently underestimating —
real tickets are getting harder than historical patterns predicted. This might
mean ticket complexity is increasing, your workforce changed, or a wave of
patch-related incidents is dominating the queue (sound familiar?).

A model trained in January may be wrong by March if the ticket landscape shifts.
**Residual monitoring is your early warning system.**

---

## How to practice this

Open `notebooks/math-foundations/07_regression.ipynb` and work through:

1. Fit the simple token-count model to the provided dataset and read the
   coefficients — what does the slope mean in plain English for your service desk?
2. Calculate residuals for the Monday ticket at 60 tokens vs 220 tokens —
   why does the fuller context change the estimate so much?
3. Add H_norm and top_prob as features and measure whether prediction error drops
4. Simulate 4 weeks of residual drift and identify the week the model needs retraining

---

## Checklist before moving on

- [ ] Can you explain what the slope coefficient means in your specific operational context?
- [ ] Are you feeding in features you already compute (H_norm, top_prob) — not just token count?
- [ ] Do you track residuals week-over-week to catch model drift early?
- [ ] Is the regression output being used as a signal for staffing, not a hard rule that bypasses human judgment?
- [ ] Have you investigated large-residual outliers — they often reveal a ticket type your model hasn't seen before?

--8<-- "_abbreviations.md"