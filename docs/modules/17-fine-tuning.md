# Module 17 — Fine-Tuning: when prompting stops being enough

Every module so far has worked with the model as a fixed object — something you route around, threshold against, and interpret after the fact. Fine-tuning is the first module where you change the model itself.

The question this module answers: *how do you know when it's time to fine-tune, and what does fine-tuning actually do?*

---

## The signal that something needs to change

After 60 days of production, your override log from Module 16 shows a persistent pattern:

```
R2 overrides (model missed a signal) — week-over-week:
  Week 1:   31
  Week 2:   29
  Week 3:   33
  Week 4:   34
  Week 5:   38
  Week 6:   41
```

The number is not declining. If prompting and threshold adjustments were going to fix this, they would have by now. The model is systematically missing something that human analysts consistently catch.

Drill into the R2 override tickets. The interpretability analysis (Module 14) finds a pattern:

```
Most common missed signal in R2 overrides:
  → tickets mentioning "MFA" or "authenticator app" failures
    combined with account lockout
  → these are being routed as account_unlock
  → analysts are escalating them as security_incident
  → the model has low attribution weight on "MFA" + "authenticator"
```

The model learned on data from before your organisation rolled out MFA. "Authenticator" wasn't in the training corpus at meaningful frequency. The model doesn't know what it means in this context.

This is a fine-tuning problem, not a prompt problem.

---

## What fine-tuning actually is

A base language model learns general patterns from vast amounts of text. When you use it for service desk classification, you're asking it to apply general language understanding to a specific domain.

Fine-tuning takes that base model and continues training it — but only on *your* data, for *your* task. The model's weights update to reflect the patterns in your tickets specifically.

```
Before fine-tuning:
  Model has seen: general internet text, general IT discussions
  Weight on "authenticator" → security_incident: low (0.07)

After fine-tuning on 6 weeks of labelled overrides:
  Model has seen: your tickets, your analyst decisions, your domain
  Weight on "authenticator" + "lockout" → security_incident: higher (0.34)
```

The model is no longer applying generic knowledge. It's applying knowledge shaped by your specific environment, your specific incident patterns, and your specific analyst judgments.

---

## What fine-tuning is not

Fine-tuning does not:

- Fix fundamental capability gaps — if the base model can't reason about a concept, fine-tuning on a few hundred examples won't teach it
- Replace calibration — a fine-tuned model can still be overconfident or underconfident (Module 8 still applies)
- Eliminate the need for human review — fine-tuning reduces the frequency of certain errors, it does not eliminate uncertainty
- Protect against adversarial inputs — a fine-tuned model can still be manipulated (Module 15 still applies)

Think of fine-tuning as updating the model's domain vocabulary and pattern weights — not rebuilding its reasoning from scratch.

---

## The fine-tuning data requirement

Fine-tuning requires labelled examples: tickets paired with the correct classification as determined by your analysts.

For the MFA problem, you need:

```
Training examples needed:
  → Confirmed MFA-related security incidents: target ≥ 150
  → Confirmed routine lockouts (no MFA signal): target ≥ 150
  → Confirmed MFA-related VPN issues: target ≥ 50
  → Total: ~350 labelled examples minimum

Sources:
  → R2 overrides from the past 6 weeks: 41 × 6 = ~246 usable
  → Historical tickets manually reviewed: ~100 additional
```

Quality matters more than quantity. A mislabelled example teaches the model the wrong thing. Every training example should be reviewed by at least one analyst before inclusion.

---

## The fine-tuning decision gate

Fine-tuning has a cost: compute time, engineering effort, and the risk of introducing new errors while fixing old ones. Apply a gate before committing:

```
Fine-tuning gate:
  Condition 1: R2 overrides stable or increasing for ≥ 4 weeks      ✅
  Condition 2: Root cause identified via attribution analysis         ✅
  Condition 3: ≥ 200 high-quality labelled examples available        ✅
  Condition 4: Prompt engineering has been attempted and failed       ✅
  Condition 5: Regression baseline established for comparison         ✅

All conditions met → proceed to fine-tuning
```

Condition 4 matters. Before fine-tuning, try adding explicit instructions to the system prompt:

```
"Tickets mentioning MFA failures, authenticator app errors, or
 two-factor authentication issues alongside account lockouts should
 be treated as potential security incidents, not routine unlocks."
```

If this reduces R2 overrides, you don't need to fine-tune — you needed better prompting. Fine-tune only when the model can't be corrected from the outside.

---

## Evaluating the fine-tuned model

After fine-tuning, run the same evaluation suite from Module 12 (Evaluation in Production) before deploying:

```
Evaluation on held-out test set:

Metric                    Before fine-tune    After fine-tune
─────────────────────────────────────────────────────────────
Overall accuracy          91.4%               93.1%
Security incident FNR     0.12                0.07
MFA-related FNR           0.61                0.09  ← target metric
Night-shift FNR           0.455               0.41  ← slight improvement
Calibration ECE           0.08                0.09  ← watch this
Variance σ (Config A)     0.13                0.15  ← slight increase
```

The target metric (MFA FNR) improved dramatically. But notice two things:

- Calibration ECE increased slightly — the fine-tuned model is marginally less well-calibrated. Run Module 8 checks again.
- Variance increased slightly — fine-tuning on a smaller domain dataset can introduce instability. Run Module 5 checks again.

Fine-tuning fixes one thing and can loosen others. Always re-run the full evaluation suite.

---

## Plugging back into the Monday ticket

The Monday ticket doesn't mention MFA. Fine-tuning doesn't change its classification. But a ticket that arrives two weeks after deployment does:

```
New ticket: "Locked out since this morning. Authenticator app giving
             error code 500. Happened right after the Friday patch."
```

Before fine-tuning:
```
account_unlock 0.71 / vpn_issue 0.12 / security_incident 0.17
→ clarification queue (H_norm = 0.68)
→ analyst required
```

After fine-tuning:
```
account_unlock 0.31 / vpn_issue 0.09 / security_incident 0.60
→ escalate immediately (top class: security_incident, H_norm = 0.71)
→ analyst notified: high-confidence security escalation
```

The model now recognises the pattern. The analyst's six weeks of overrides became training data. The system learned.

---

## Checklist

- [ ] Have you attempted prompt engineering before deciding to fine-tune?
- [ ] Is the R2 override pattern stable for ≥ 4 weeks before committing?
- [ ] Do you have ≥ 200 high-quality labelled examples with analyst review?
- [ ] Have you established a pre-fine-tune baseline for all key metrics?
- [ ] After fine-tuning, have you re-run calibration (M08) and variance (M05) checks?
- [ ] Is the fine-tuned model's evaluation on a held-out set, not the training data?
- [ ] Are you tracking which override patterns remain after fine-tuning — to identify the next gap?

---

> Fine-tuning is how the system learns from its mistakes at the model level, not just the threshold level.
> But it requires data, and data takes time to accumulate.
>
> Module 18 addresses a different gap: what if the model needs information it could never have learned during training — real-time knowledge base content, current policy documents, live ticket history?

--8<-- "_abbreviations.md"
