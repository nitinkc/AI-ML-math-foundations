# Module 20 — Drift & Retraining: when the world changes faster than your model

A model trained in January on January tickets will not perform the same way in July. The tickets change. The language changes. The infrastructure changes. The threats change.

Drift is the gradual divergence between what the model learned and what the world currently looks like. It is invisible until it becomes a problem — unless you're actively measuring it.

---

## Three types of drift

### Data drift — the inputs change

The vocabulary, length, and structure of incoming tickets shifts over time. This can happen because:

- A new software platform is rolled out (new error messages, new terminology)
- A new type of attack emerges (new phishing patterns, new social engineering language)
- Organisational changes alter who submits tickets (new departments, new users)
- Seasonal patterns shift (new contractors in Q4, remote workers in summer)

**Detection:** Compare the token distribution of this month's tickets against the training corpus. If the overlap is decreasing, the model is seeing increasingly unfamiliar inputs.

```
Token distribution overlap (training vs production):
  Month 1:  94.2%  ✅
  Month 2:  93.1%  ✅
  Month 3:  91.8%  ✅
  Month 4:  88.3%  ⚠️  (new ERP system rollout)
  Month 5:  84.1%  🔴  (retraining trigger)
```

### Label drift — the correct answers change

What counts as a `security_incident` this month may differ from what counted three months ago. Policy changes, new threat intelligence, and updated escalation procedures all change the ground truth.

After a major phishing campaign in month 4, your security team updates the policy: any ticket mentioning unexpected MFA prompts should be treated as a security incident regardless of whether a lockout is present.

The model was trained before that policy existed. Its labels are now wrong by definition.

**Detection:** Track analyst override rate (Module 16) week over week. A sustained increase in R4 overrides (policy change) is the label drift signal.

```
R4 overrides (policy mismatch):
  Week 1:   14
  Week 2:   12
  Week 3:   15
  Week 4:   31  ← policy updated
  Week 5:   38
  Week 6:   42  → retraining trigger
```

### Concept drift — the relationship between inputs and outputs changes

The subtlest form. The same words now mean different things in your environment.

Before the MFA rollout: "verification code" in a ticket meant the user was doing routine 2FA.
After the MFA rollout: "verification code" in a ticket is a potential phishing indicator.

Same token. Different security implication. The model can't know this unless it's retrained.

**Detection:** Track calibration ECE (Module 8) over time. If the model's stated confidence is increasingly divorced from its actual accuracy, concept drift is likely.

---

## The Monday ticket as a drift case study

Imagine the Monday ticket arrives not in week 1 of deployment but in month 6, after:

- A major phishing campaign targeted your organisation
- Policy now flags any patch-notice mention as a security signal
- "VPN dropping" is a known indicator of a specific lateral movement technique

The model was trained on pre-campaign data. Its logits for the Monday ticket are unchanged:

```
Month 1: account_unlock 0.576 / vpn_issue 0.386 / security_incident 0.039
Month 6: account_unlock 0.576 / vpn_issue 0.386 / security_incident 0.039
```

But the correct answer in month 6 is unambiguously `security_incident`. Every analyst is overriding this ticket type. The override rate has been climbing for six weeks.

The model has drifted — not because it changed, but because the world around it did.

---

## Measuring drift: the monitoring stack

Add drift metrics to the weekly operations review alongside cost and latency (Module 19):

```
Weekly drift report:

Metric                              Baseline    This week   Trend
────────────────────────────────────────────────────────────────
Token overlap (training vs prod)    94.2%       88.3%       ↓ ⚠️
Calibration ECE                     0.06        0.11        ↓ ⚠️
R2 override rate (model missed)     31/wk       41/wk       ↑ ⚠️
R4 override rate (policy change)    14/wk       38/wk       ↑ 🔴
Security incident FNR               0.12        0.23        ↑ 🔴
Overall accuracy                    91.4%       87.1%       ↓ ⚠️
```

Any single metric at ⚠️ triggers investigation. Two or more at 🔴 triggers the retraining decision gate.

---

## The retraining decision gate

Retraining is not free. It requires labelled data, engineering time, and a full re-evaluation cycle. Apply a gate:

```
Retraining gate:

Condition 1: ≥2 drift metrics at 🔴 for ≥3 consecutive weeks       ✅
Condition 2: Root cause identified (data drift / label drift /
             concept drift)                                          ✅
Condition 3: ≥300 high-quality labelled examples since last train   ✅
Condition 4: RAG knowledge base updated to reflect current reality  ✅
Condition 5: Prompt engineering adjustments have been attempted     ✅

All conditions met → begin retraining cycle
```

Condition 4 matters. Before retraining, update the RAG knowledge base (Module 18) with current policies, current incident patterns, and current terminology. Some apparent model drift is actually a RAG staleness problem — cheaper to fix.

---

## Retraining vs fine-tuning

Both involve updating the model with new data. The difference is scope:

```
Fine-tuning (Module 17):
  → Fix a specific, identified gap
  → Small dataset (200–500 examples)
  → Fast (hours to days)
  → Risk: may not generalise beyond the specific fix

Retraining:
  → Reset the model's understanding of the full distribution
  → Larger dataset (full recent corpus + historical)
  → Slower (days to weeks)
  → Risk: may introduce regressions on previously stable classes
```

When drift is narrow (one ticket type, one terminology shift), fine-tune.
When drift is broad (overall accuracy declining, multiple classes affected), retrain.

---

## After retraining: the regression test

Before deploying a retrained model, run it against a frozen holdout set from your stable period — the weeks when the original model was performing well. The retrained model should:

```
Regression test (stable period holdout):

Metric                  Original model    Retrained model    Pass?
──────────────────────────────────────────────────────────────────
Overall accuracy        91.4%             92.1%              ✅
Security FNR            0.12              0.08               ✅
Password reset FNR      0.02              0.03               ✅ (within tolerance)
VPN issue accuracy      88.6%             89.2%              ✅
Calibration ECE         0.06              0.07               ✅
Variance σ              0.13              0.16               ⚠️ watch
```

The retrained model should not perform significantly worse on any class it previously handled well. A regression on `password_reset` (which wasn't causing problems) while fixing `security_incident` is a bad trade.

---

## Checklist

- [ ] Are token distribution overlap metrics tracked weekly?
- [ ] Are R2 and R4 override rates tracked as drift signals?
- [ ] Is calibration ECE monitored over time, not just at deployment?
- [ ] Is the retraining decision gate documented and applied consistently?
- [ ] Before retraining, is the RAG knowledge base updated first?
- [ ] Is the distinction between fine-tuning (narrow gap) and retraining (broad drift) applied correctly?
- [ ] Is a regression test on the stable holdout set run before every retraining deployment?

---

> Drift is not a failure. It's the normal behaviour of a system operating in a changing world.
> The failure is not detecting it.
>
> Module 21 closes the course with the question you hope never to need: what do you do when something goes seriously wrong — a security incident was misrouted, an attacker got in, an analyst made a bad call? How do you investigate it, document it, and make sure it doesn't happen again?

--8<-- "_abbreviations.md"
