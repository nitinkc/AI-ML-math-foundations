# Module 16 — Human-in-the-Loop: when does the model hand off, and what happens next?

Every previous module has built toward a moment: the model finishes its work and a human takes over. But "human review" is not a safety net you bolt on at the end. It's a designed interface between the model's probabilistic reasoning and a human's contextual judgment — and how you design it determines whether the human adds value or just rubber-stamps the machine.

---

## Three reasons the Monday ticket needed a human

By the time the pipeline finished processing, three separate modules had independently flagged the Monday ticket for human review:

```
Module 02:  top_prob = 0.576  (below 0.80 auto-route threshold)
Module 04:  H_norm = 0.74     (above 0.45 clarification ceiling)
Module 14:  security attribution tokens detected despite low security probability
```

Any one of these would have been enough. Together they make the case unambiguous: the model is uncertain, the uncertainty is spread across a dangerous class boundary, and the classification evidence partially contradicts the classification decision.

The human is not a fallback. The human is the intended outcome for this ticket type.

---

## What the analyst actually sees

Good human-in-the-loop design means the analyst sees everything they need to make a better decision than the model — not just the model's answer.

```
┌─────────────────────────────────────────────────────────────┐
│ TICKET #4821 — Clarification Queue                          │
│ Submitted: Monday 09:47 — HQ employee                       │
├─────────────────────────────────────────────────────────────┤
│ "Can't log in since this morning. VPN keeps dropping every  │
│  few minutes. Happened right after the security team sent   │
│  out that patch notice last Friday."                        │
├─────────────────────────────────────────────────────────────┤
│ Model classification:  account_unlock  (57.6%)              │
│ Confidence:            LOW  ▓▓░░░░░░░░                      │
│ Entropy:               HIGH (0.74)                          │
│                                                             │
│ Attribution flags:                                          │
│   ⚠ "patch notice" — security signal detected              │
│   ⚠ "security team" — security signal detected             │
│                                                             │
│ Predicted resolution: 54.8 min                              │
├─────────────────────────────────────────────────────────────┤
│ Suggested responses:                                        │
│  [A] Password reset sent — you'll receive an email shortly  │
│  [B] Can you confirm whether this started after Friday's    │
│      update?                                                │
│  [C] Flagging this for our security team given the patch    │
│      notice you mentioned                       ← RECOMMENDED│
├─────────────────────────────────────────────────────────────┤
│ Your decision:  [Route: account_unlock] [Route: vpn_issue]  │
│                 [Escalate: security]   [Request more info]  │
└─────────────────────────────────────────────────────────────┘
```

The model recommended Draft C and pre-selected escalation. The analyst can agree, override, or ask for more information. Every choice is one click and every choice is logged.

---

## The four analyst actions and what each means

**Agree with model** — analyst confirms the model's routing. Logged as a positive signal. If the model was correct, this strengthens the training data for that ticket type.

**Override routing** — analyst changes the destination. This is the most valuable signal in the system. An override says: *the model saw what I saw and reached the wrong conclusion.* Every override should be logged with a reason code:

```
Override reason codes:
  R1 — Context not in ticket (analyst has prior knowledge of user/situation)
  R2 — Model missed a signal (attribution didn't surface something obvious)
  R3 — Model over-weighted a signal (false positive on security tokens)
  R4 — Policy change (routing rules updated since model was trained)
  R5 — Adversarial suspicion (ticket seems constructed)
```

**Edit and send draft** — analyst modifies one of the suggested responses. The diff between the suggested draft and the sent response is logged. Repeated edits to the same phrasing indicate the model's language needs updating.

**Request more information** — analyst sends a clarifying question before routing. The user's response is appended to the ticket and re-classified. This is the correct action when the model's entropy is high and the routing consequence is severe.

---

## Override logging: the feedback loop

Overrides are the highest-quality training signal you have. A human expert looked at the same evidence the model saw and reached a different conclusion. That disagreement is valuable — if you capture it.

```
Weekly override report:
  Total tickets reviewed:    847
  Model agreed:              731  (86.3%)
  Overridden:                116  (13.7%)

  Override breakdown:
    R1 (context not in ticket):   44  (38%)
    R2 (model missed signal):     31  (27%)
    R3 (false positive):          22  (19%)
    R4 (policy change):           14  (12%)
    R5 (adversarial suspicion):    5   (4%)
```

R2 overrides — model missed a signal — go directly to the interpretability team (Module 14). What tokens were present that the model underweighted? Add them to the attribution review.

R3 overrides — false positives — go to the threshold calibration review (Module 8). Is the security escalation threshold too sensitive?

R4 overrides — policy changes — trigger an immediate review of whether the model needs retraining or just a threshold update.

---

## When humans should not override

Human-in-the-loop does not mean human-always-right. Analysts have biases too. An override policy without safeguards creates new failure modes:

**Automation bias in reverse** — analysts who distrust the model override it reflexively, even when it's correct. Track override accuracy: what percentage of overrides turn out to be correct vs the original model decision?

**Inconsistency across analysts** — two analysts seeing the same ticket reach different conclusions. This is a calibration problem for humans, not just models. Track inter-analyst agreement on identical tickets.

**Override fatigue** — if analysts review too many tickets, decision quality drops. The clarification queue size is itself a safety metric. If it exceeds a threshold, it's time to retrain the model or adjust thresholds.

```
Queue health metrics (reviewed weekly):
  Avg tickets per analyst per day:   target ≤ 40
  Avg review time per ticket:        target ≤ 4 min
  Override accuracy rate:            target ≥ 78%
  Inter-analyst agreement:           target ≥ 85%
```

---

## The escalation path for the Monday ticket

The analyst sees the ticket, notes the two ⚠️ attribution flags, reads the phrase "security team sent out that patch notice", and selects Draft C with a minor edit:

```
Sent response:
"Thanks for reaching out. Given you mentioned the patch notice from
Friday, I've flagged this for our security team to review alongside
the account access issue. You'll hear from them shortly — estimated
response within the hour."
```

Routing: `security_incident`
Override reason: R2 — model missed combined signal weight
Resolution time (actual): 41 minutes

The model predicted 54.8 minutes (Module 7). Actual was 41 minutes. That residual — 13.8 minutes shorter — gets logged. The regression model updates.

---

## The full pipeline, complete

```
Ticket arrives
    │
    ▼
[M01] Tokenize
    │
    ▼
[M03] Logits → Softmax
    │
    ▼
[M15] Adversarial pre-check
    │
    ▼
[M13] Segment check
    │
    ▼
[M02] Probability + margin check
    │
    ▼
[M04] Entropy → routing decision
    │
    ├── auto-route (high confidence, low entropy, no flags)
    │       └── logged, monitored, periodic audit sample
    │
    └── human review (low confidence OR high entropy OR flags)
            │
            ▼
        [M14] Attribution flags surfaced to analyst
        [M06] Draft responses presented
        [M07] Resolution estimate shown
        [M16] Analyst decides  ← THIS MODULE
            → agree / override (with reason code) / edit / clarify
            │
            ▼
        Decision logged
        Outcome tracked
        Residuals → M07 regression update
        Overrides → M14 attribution review
                  → M08 calibration review
                  → M20 drift monitoring
```

---

## Checklist

- [ ] Does the analyst interface surface confidence, entropy, and attribution flags — not just the top classification?
- [ ] Are all four analyst actions available: agree, override, edit, clarify?
- [ ] Are overrides logged with reason codes?
- [ ] Is override accuracy tracked to catch reverse automation bias?
- [ ] Is inter-analyst agreement measured on shared test tickets?
- [ ] Is queue size monitored as a safety metric?
- [ ] Are R2 overrides feeding back to the interpretability review?
- [ ] Are R4 overrides triggering a policy-model alignment check?

---

> The model is not trying to replace the analyst. It's trying to make sure the analyst's attention goes where it's needed most.
>
> Module 17 asks: what happens when the model keeps needing correction in the same ways? At some point, prompting and threshold tuning stop being enough — and you need to change the model itself.

--8<-- "_abbreviations.md"
