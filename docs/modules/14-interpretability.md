# Module 14 — Interpretability: which words are driving the classification?

Your bias audit in Module 13 found that night-shift tickets have a 45.5% false-negative rate for security incidents. The system is failing a specific group of users in a dangerous way. But knowing *that* it's failing is not the same as knowing *why*.

Interpretability is how you look inside the classification decision and find out which parts of the ticket the model weighted most heavily — and which parts it ignored.

---

## The question interpretability answers

When the Monday ticket was classified as `account_unlock` with probability 0.576, what drove that decision?

Was it:
- "locked out of my account" → likely `account_unlock`
- "VPN keeps dropping" → likely `vpn_issue`
- "patch notice last Friday" → likely `security_incident`

The model saw all three signals. Interpretability tells you which one won, and by how much.

---

## The technique: token attribution

The most practical interpretability method for a classifier is **token attribution** — assigning a score to each token that represents how much it pushed the prediction toward or away from each class.

For the Monday ticket, a simplified attribution might look like:

```
Token                  account_unlock   vpn_issue   security_incident
─────────────────────────────────────────────────────────────────────
"locked"               +0.31            -0.08        -0.04
"out"                  +0.18            -0.03        -0.02
"account"              +0.41            -0.11        -0.07
"VPN"                  -0.09            +0.38        +0.06
"dropping"             -0.04            +0.29        +0.03
"patch"                -0.02            -0.05        +0.44
"notice"               -0.01            -0.03        +0.31
"Friday"               +0.01            +0.02        +0.08
─────────────────────────────────────────────────────────────────────
Net score (logit):      3.2              2.8          0.5
```

This matches the logits you saw in Module 3. Now you can see *why* those logits came out that way:

- "locked" and "account" together pushed hard toward `account_unlock`
- "VPN" and "dropping" pushed toward `vpn_issue`
- "patch" and "notice" pushed toward `security_incident` — but their combined weight was not enough to overcome the first two signals

The model was not ignoring the security signal. It was outweighed by louder signals earlier in the ticket.

---

## Why night-shift tickets fail: the attribution difference

Now run the same attribution on a night-shift version of the same underlying incident:

```
Night-shift ticket:  "locked out again + vpn down. same as last week"
```

```
Token                  account_unlock   vpn_issue   security_incident
─────────────────────────────────────────────────────────────────────
"locked"               +0.31            -0.08        -0.04
"out"                  +0.18            -0.03        -0.02
"vpn"                  -0.09            +0.38        +0.06
"down"                 -0.03            +0.21        +0.02
─────────────────────────────────────────────────────────────────────
Net score (logit):      2.6              2.4          0.1
```

The security signal is almost entirely absent. Not because the incident is different — it's the same situation — but because the phrases that carry security weight ("patch notice", "security team", "last Friday") were never written. The writer was tired, brief, and assumed context the model cannot see.

This is a **vocabulary gap**, not a data volume problem. The fix is not just more night-shift training data. It's teaching the model to recognise that brief terse tickets from certain contexts deserve elevated uncertainty — which feeds back into Module 4 (entropy) and Module 13 (segment thresholds).

---

## The two levels of interpretability

### Level 1 — Per-ticket explanation (real-time)

Every time an analyst reviews a ticket in the clarification queue, they should see which tokens drove the classification:

```
Classification: account_unlock (57.6%)

Key signals:
  → "locked out of my account"   strongly supports account_unlock
  → "VPN keeps dropping"         moderately supports vpn_issue
  → "patch notice last Friday"   weakly supports security_incident  ⚠️

Note: security signal present but outweighed. Review carefully.
```

The ⚠️ flag is not the model changing its decision. It's the model surfacing that it noticed the signal and couldn't act on it with confidence. That's exactly the information the analyst needs.

### Level 2 — Aggregate attribution (weekly review)

Across all tickets in the past week, which tokens most consistently drove `security_incident` classifications?

```
Top 10 tokens by security_incident attribution weight:
  1. "phishing"          +0.61
  2. "patch"             +0.44
  3. "notice"            +0.31
  4. "suspicious"        +0.29
  5. "link"              +0.27
  6. "clicked"           +0.24
  7. "forwarded"         +0.21
  8. "email"             +0.19
  9. "Friday"            +0.08
 10. "update"            +0.07
```

This list is useful for three things:

- **Confirming the model is using the right signals** — "phishing", "patch", "suspicious" are exactly what you'd want
- **Spotting unexpected signals** — if "Friday" has significant weight, the model may have learned a spurious day-of-week pattern
- **Identifying gaps** — if "vpn" and "locked" are appearing in the top security attribution tokens, the model is confusing routine IT issues with security incidents

---

## Connecting to the night-shift gap

You now have everything you need to diagnose Module 13's finding:

```
Step 1 (M13):  Night-shift FNR = 0.455  ← the gap is real
Step 2 (M09):  Is this causal or a confounder?
Step 3 (M14):  Run attribution on night-shift vs day-shift security incidents
               → night-shift tickets missing "patch", "notice", "suspicious"
               → attribution score for security_incident drops accordingly
               → root cause: vocabulary gap, not model bias against shift pattern
Step 4:        Intervention: lower entropy threshold for short tickets
               + add analyst flag for tickets under 80 tokens with VPN + lockout combo
```

Interpretability turned a statistical finding into an actionable root cause.

---

## The limits of interpretability

Token attribution tells you what the model weighted. It does not tell you:

- Whether the model's weighting is *correct* — a token can have high attribution and still be a spurious correlation
- What the model would have decided if a token were different — that requires counterfactual analysis
- Why the model assigned a particular weight to a token — that goes deeper into the architecture than attribution can reach

Treat attribution as a diagnostic tool, not a complete explanation. It narrows the search space. It does not close the case.

---

## Plugging back into the Monday ticket

```
[M14] Attribution check (per ticket, at analyst review)
    → top attribution tokens: "locked" (+0.41), "account" (+0.38), "patch" (+0.44)
    → security signal present: "patch notice" visible in attribution
    → flag generated: "security tokens detected — review before routing"
    → analyst sees flag alongside draft responses from M06
    → analyst selects Draft 3: escalates to security team
```

The model's classification said `account_unlock`. The attribution said *but look at this*. The analyst made the right call because the system surfaced what the model noticed but couldn't act on alone.

---

## The full pipeline, updated

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
[M13] Segment check
    │
    ▼
[M02] Probability + margin check
    │
    ▼
[M04] Entropy check → routing decision
    │
    ▼
[M14] Attribution  ← NEW
    → score each token's contribution per class
    → surface top security tokens even if class lost
    → attach attribution summary to analyst view
    │
    ▼
[M05] Stability gate (deploy-time)
    │
    ▼
[M06] Deterministic routing + stochastic drafts
    │
    ▼
[M07] Resolution time estimate
    │
    ▼
Analyst reviews — with attribution flags visible
```

---

## Checklist

- [ ] Is token attribution running on every ticket that enters the clarification queue?
- [ ] Are security-class attribution signals surfaced to analysts even when `security_incident` is not the top class?
- [ ] Are you running weekly aggregate attribution reviews to check for spurious tokens?
- [ ] Have you used attribution to diagnose your segment FNR gaps before choosing an intervention?
- [ ] Are you treating attribution as a diagnostic, not a complete explanation?

---

> The model noticed "patch notice last Friday." It just couldn't act on it alone.
> Interpretability makes sure a human can.
>
> Module 15 flips this around: what if someone deliberately crafts a ticket to *hide* those signals — or to inject false ones?

--8<-- "_abbreviations.md"
