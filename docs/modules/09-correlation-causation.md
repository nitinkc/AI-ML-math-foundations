# Module 9 — Correlation vs Causation: are you fixing the right thing?

Your regression model from Module 7 is live. Resolution time estimates are running. And then an analyst notices something:

> "Tickets with higher token counts are taking longer to resolve AND getting escalated more often. Should we cap prompt length?"

It's a reasonable instinct. The pattern is real. But acting on it directly might make things worse. This module is about knowing the difference between a signal worth investigating and a cause worth fixing.

---

## The pattern that triggered the question

You pull the data:

```
Token count < 100:    avg resolution  18.2 min    escalation rate  8.1%
Token count 100–200:  avg resolution  34.7 min    escalation rate  14.3%
Token count > 200:    avg resolution  61.4 min    escalation rate  23.9%
```

The correlation is clear. As token count goes up, both resolution time and escalation rate go up with it. Plotted on a chart it looks almost linear.

So the proposed fix: **cap tickets at 150 tokens.**

Before you ship that policy, you need to ask one question: *why does this pattern exist?*

---

## The math in plain terms

Correlation measures how consistently two variables move together:

$$
r = \frac{\sum_i (x_i - \bar{x})(y_i - \bar{y})}{\sqrt{\sum_i (x_i - \bar{x})^2} \cdot \sqrt{\sum_i (y_i - \bar{y})^2}}
$$

- $r = 1.0$: perfect positive correlation — both always increase together
- $r = -1.0$: perfect negative correlation — one increases as the other decreases
- $r = 0$: no linear relationship

For token count vs resolution time in your data: $r = 0.71$. Strong. But correlation says nothing about *why* the pattern exists. That requires a different kind of thinking.

---

## The hidden variable: ticket complexity

Here's an alternative explanation. Segment the same data by ticket type:

| Ticket type | Avg tokens | Avg resolution | Escalation rate |
|:---|---:|---:|---:|
| Password reset | 68 | 4.1 min | 2.3% |
| VPN issue | 142 | 18.6 min | 11.4% |
| Security incident | 231 | 47.2 min | 61.8% |

The pattern isn't driven by token count. It's driven by **ticket complexity** — which happens to produce longer descriptions AND longer resolution times AND more escalations, all at once.

Token count is innocent. It's a symptom of complexity, not a cause of delays.

This hidden third variable — ticket complexity — is called a **confounder**. It causes both things you're measuring simultaneously, creating the illusion of a direct relationship between them.

```
                    Ticket Complexity
                    ↙               ↘
           Token Count          Resolution Time
               ↕
          (apparent correlation — not causal)
```

---

## What happens if you act on the correlation anyway

You cap tickets at 150 tokens. Users with complex issues start truncating their descriptions to fit. Now your classifier gets less signal — exactly the signal that separates `vpn_issue` from `security_incident`.

Recall Module 1: the Monday ticket contained the phrase "this happened right after the security team sent out that patch notice last Friday." That phrase is 17 tokens. Under a 150-token cap on a 160-token ticket, context like that is exactly what gets cut.

The result:

```
Before token cap:   security incident FNR  =  0.12
After token cap:    security incident FNR  =  0.31  ← nearly tripled
```

You tried to reduce resolution time. You made misrouting significantly worse. This is what acting on correlation without understanding causation costs in production.

---

## The three questions before any policy change

### 1. Could a confounder explain both variables?

List every plausible third variable that might cause both sides of the correlation. For token count and resolution time:

- Ticket complexity (most likely)
- Submitter verbosity habits (possible — some users always write long tickets regardless of issue type)
- Multi-issue tickets (a single ticket reporting both a VPN drop and a lockout is longer and harder)

Until you've ruled these out, you don't have a cause.

### 2. Does the correlation hold within each segment?

If token count *causes* longer resolution times, the pattern should hold within each ticket type. Check it:

| Ticket type | Token–resolution correlation |
|:---|---:|
| Password reset | r = 0.09 |
| VPN issue | r = 0.14 |
| Security incident | r = 0.11 |

Within each category, the correlation nearly vanishes. That's the confounder signature: strong aggregate correlation, weak within-segment correlation. Complexity was doing all the work.

### 3. Can you test the causal claim with a controlled pilot?

If you still believe token count has some causal effect after checking for confounders, test it:

- Take one queue
- Randomly assign tickets to a capped and uncapped condition
- Measure resolution time and FNR after 2 weeks
- Compare the outcomes

If resolution time drops in the capped condition without increasing FNR, you have evidence. If FNR rises, the confounder hypothesis was right and the cap is harmful.

This is the only reliable path from correlation to confident action.

---

## Connecting back to the regression model

Module 7 produced this estimate for the Monday ticket:

$$
\hat{y} = 2.1 + (0.21 \times 220) + (18.4 \times 0.74) + (-12.3 \times 0.576) = 54.8 \text{ min}
$$

Token count (220) has a coefficient of 0.21 in that model. Does that mean tokens *cause* longer resolution? Not necessarily. The regression model learned that longer tickets correlate with longer resolution — and uses that correlation to produce useful predictions. That's fine for forecasting. It is not fine for intervention.

The distinction matters the moment you consider acting on a coefficient:

| Use case | Correlation sufficient? | Causation required? |
|:---|:---:|:---:|
| Predicting resolution time | ✅ Yes | ❌ No |
| Staffing decisions from predictions | ✅ Yes | ❌ No |
| Changing ticket submission policy | ❌ No | ✅ Yes |
| Retraining the model on filtered data | ❌ No | ✅ Yes |

Regression is a prediction tool. It is not a policy tool unless you've done the causal work first.

---

## Plugging back into the Monday ticket

The Monday ticket is 220 tokens. Under the naive policy it would have been flagged for truncation. Instead, the pipeline now applies the causation check before any length-based intervention is considered:

```
[M09] Correlation check (at policy review time, not per ticket)
    → token count correlates with resolution time: r = 0.71
    → segmented by ticket type: r = 0.09–0.14  ← confounder detected
    → proposed cap policy: BLOCKED pending controlled pilot
    → token count retained as regression feature (predictive, not causal)
    → Monday ticket: 220 tokens, no truncation applied
```

The Monday ticket keeps its full context. The phrase about the patch notice stays in. The classifier has the signal it needs.

---

## The updated pipeline position

This module sits between calibration (Module 8) and sampling controls (Module 10). Its job is to prevent you from misusing the outputs of your regression and calibration models before you've checked whether the patterns you're seeing are real causes or just correlated symptoms.

```
[M08] Calibration → are the probabilities trustworthy?
    │
    ▼
[M09] Correlation vs Causation  ← THIS MODULE
    → before acting on any pattern:
       segment the data, check for confounders, run a pilot
    │
    ▼
[M10] Sampling Controls → temperature, top-p, top-k
```

---

## Checklist before acting on any observed pattern

- [ ] Have you listed every plausible confounder that could explain both variables?
- [ ] Have you segmented the data by ticket type, queue, and submitter profile — does the correlation hold within segments?
- [ ] Have you checked whether your regression coefficients are being used for prediction (fine) or for policy changes (requires causal evidence)?
- [ ] Have you designed a controlled pilot before shipping any intervention?
- [ ] Have you documented the distinction between correlation and causation in your decision record so future analysts don't repeat the mistake?

---

> Strong correlation means: *this is worth investigating.*
> It does not mean: *this is worth changing.*
>
> Module 13 will revisit this discipline in a specific context: when you segment accuracy by user group and find a gap, the temptation is to act immediately. The right move is to ask the same question first — confounder or cause?

--8<-- "_abbreviations.md"
