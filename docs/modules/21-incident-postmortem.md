# Module 21 — Incident & Postmortem: when the pipeline fails

This is the module you hoped you'd never need to use.

Somewhere in the real world, the Monday ticket — or one exactly like it — got misrouted. The security incident was classified as a routine account unlock. A tier-1 agent sent a password reset link. An attacker used it to re-enter the network. The breach was discovered 11 days later.

Every capability in this course exists to prevent that outcome. This module is about what happens when prevention fails: how you investigate it, document it, communicate it, and fix it so it doesn't happen again.

---

## The incident timeline (a hypothetical)

```
Friday 17:32   Security team sends patch notice KB-2847 to all users
Monday 09:47   Ticket #4821 arrives: lockout + VPN + patch mention
Monday 09:47   Pipeline runs: account_unlock (0.576) → clarification queue
Monday 09:51   Analyst A opens ticket, reviews for 90 seconds
Monday 09:52   Analyst A selects Draft A ("password reset sent") — does not read attribution flags
Monday 09:52   Password reset email sent. Ticket closed.
Monday 09:54   Attacker uses reset link. Account re-accessed.
Monday 10:00–  Attacker moves laterally. 3 additional accounts accessed.
Tuesday 19:14  Security monitoring detects anomalous access pattern
Tuesday 19:31  Incident declared. Forensics begins.
Wednesday 08:00 Root cause identified: ticket #4821 misrouted.
```

The model did its job. The pipeline flagged the ticket for human review, generated attribution flags for "patch notice" and "security team", and recommended Draft C (escalation). The human overrode the recommendation without reading the flags.

This is not a model failure. It's a system failure — the interface between the model and the human didn't work as designed.

---

## The postmortem structure

A postmortem is not a blame exercise. It is a structured investigation of what happened, why it happened, and what changes will prevent recurrence.

### 1. Timeline reconstruction

Reproduce the exact pipeline state at the moment of the incident. Every module in this course was designed with this moment in mind — because every decision was logged.

```
Postmortem reconstruction for ticket #4821:

[M01] Tokens: 68 (ticket) + 680 (system + history) = 748 total
[M03] Logits: account_unlock 3.2 / vpn_issue 2.8 / security_incident 0.5
[M02] top_prob: 0.576 (below 0.80 → human review triggered ✅)
[M04] H_norm: 0.74 (above 0.45 → clarification queue ✅)
[M14] Attribution flags: "patch notice" +0.44, "security team" +0.31 ✅
[M06] Draft C recommended: escalation to security team ✅
[M16] Analyst action: Draft A selected, no override reason logged
[M15] Adversarial check: no flags (honest ticket)
[M13] Segment: HQ employee, standard hours, FNR 0.053 (no override triggered)
```

The pipeline performed correctly at every step. The failure occurred at the analyst decision point — Draft C was recommended and Draft A was chosen.

### 2. Contributing factors

A postmortem looks for all contributing factors, not just the proximate cause.

**Contributing factor 1 — Interface design (primary)**
Attribution flags were displayed but not visually prominent. The analyst interface showed a small ⚠️ icon below the fold. Analyst A did not scroll to see it. The recommended draft was not visually distinguished from the other options.

**Contributing factor 2 — Queue pressure**
At 09:51 Monday, the clarification queue contained 23 tickets. Analyst A had a 10 AM meeting. Average review time in the previous hour: 87 seconds per ticket. The 90-second review time was within normal range — but normal was too fast for a high-stakes ticket.

**Contributing factor 3 — Training gap**
Analyst A had been onboarded 6 weeks earlier. Attribution flags and their significance were covered in week 1 training but not reinforced. There was no policy requiring analysts to confirm they had reviewed flags before closing a security-adjacent ticket.

**Contributing factor 4 — Threshold gap**
The adversarial check (Module 15) did not flag this ticket — correctly, because it wasn't adversarial. But the combination of high-FNR-adjacent signals (patch mention + lockout + VPN) did not trigger an additional confirmation step. The routing policy had no "two signals present = mandatory escalation check" rule.

### 3. What did not cause the incident

Equally important: what the postmortem ruled out.

- The model was not miscalibrated — attribution was correct
- The pipeline did not fail — all checks ran as designed
- The ticket was not adversarially crafted — honest user, honest content
- The RAG system did not surface incorrect documents — RAG was not active at this time (pre-Module 18 deployment)

Ruling out non-causes prevents the wrong fixes from being applied.

---

## The remediation plan

Each contributing factor gets a specific fix:

**Fix 1 — Interface redesign (addresses CF1)**
Attribution flags with security tokens are moved above the fold. The recommended draft is visually distinguished. A confirmation dialog appears if the analyst selects a lower-priority route than recommended when security flags are present:
```
"⚠️ Security signals detected in this ticket.
 You are routing to account_unlock instead of the recommended
 security_incident escalation. Confirm and provide reason:"
 [Confirm with reason]  [Review flags]
```

**Fix 2 — Queue throttling (addresses CF2)**
When the clarification queue exceeds 15 tickets, a supervisor alert fires. Tickets with security attribution flags are automatically deprioritised *upward* — moved to the top of the queue and flagged for senior analyst review.

**Fix 3 — Training reinforcement (addresses CF3)**
Attribution flag review is added as a mandatory step in the analyst workflow, not just onboarding. Monthly refresher: 30-minute scenario exercise on high-stakes ticket types. Completion tracked.

**Fix 4 — Routing rule addition (addresses CF4)**
New routing rule added to the pipeline:

```
[M04 + M14 combined rule]
if security_attribution_tokens ≥ 2 AND (account_unlock OR vpn_issue is top class):
    → override to: mandatory human review with senior analyst
    → block auto-route and block tier-1 routing
    → require explicit escalation decision with reason code
```

This rule would have caught ticket #4821. It catches every ticket where the model's top class disagrees with the security attribution evidence.

---

## The 30-day follow-up

A postmortem is not complete at the remediation plan. Each fix has a verification step.

```
30-day postmortem follow-up:

Fix 1 (interface):      A/B test new vs old interface
                        Metric: flag acknowledgment rate
                        Target: ≥95% of flagged tickets show flag reviewed
                        Result at 30 days: 97.3% ✅

Fix 2 (queue throttle): Monitor queue size and alert frequency
                        Metric: tickets reviewed in <60s with security flags
                        Target: 0 per week
                        Result at 30 days: 1 (investigated, edge case) ✅

Fix 3 (training):       Track completion of monthly scenario exercise
                        Metric: 100% completion rate
                        Target: 100%
                        Result at 30 days: 94% (6 analysts on leave) ⚠️

Fix 4 (routing rule):   Count tickets caught by new rule
                        Metric: tickets blocked from tier-1 by new rule
                        Target: >0 (rule is firing)
                        Result at 30 days: 14 tickets intercepted ✅
```

Fix 3 is incomplete. The remediation plan stays open. The postmortem is not closed until every fix is verified.

---

## What the Monday ticket looks like now

Six months after the incident, the same ticket arrives again:

```
Ticket: "Can't log in since this morning. VPN keeps dropping. 
         Happened right after the security team sent out that patch notice."
```

```
[M03]  account_unlock 0.576 (unchanged — model hasn't been retrained yet)
[M14]  "patch notice" +0.44 ← security token
       "security team" +0.31 ← security token
       security_attribution_tokens = 2

[NEW RULE — Fix 4]
       security_attribution_tokens ≥ 2 AND top class = account_unlock
       → mandatory senior analyst review
       → tier-1 routing blocked
       → confirmation dialog required

[M16]  Senior analyst sees: attribution flags above the fold (Fix 1)
                            recommended draft: escalation
                            confirmation required to downgrade
       Decision: escalate to security team
       Reason code: R1 (context + policy awareness)
```

The outcome is different. Not because the model changed — it didn't. Because the system around the model changed. That's the lesson of this entire course.

---

## The full pipeline, final state

```
Ticket arrives
    │
    ▼
[M01] Tokenize → token budget check
    │
    ▼
[M18] RAG retrieval → augmented context
    │
    ▼
[M03] Logits → Softmax
    │
    ▼
[M15] Adversarial pre-check
    │
    ▼
[M13] Segment check → threshold adjustment
    │
    ▼
[M02] Probability + margin check
    │
    ▼
[M04] Entropy check
    │
    ▼
[M14] Attribution scoring → security token count
    │
    ▼
[COMBINED RULE] security tokens ≥ 2 + class mismatch → senior review
    │
    ├── Tier 1: auto-route (high confidence, no flags)
    │
    ├── Tier 2: clarification queue (standard human review)
    │
    └── Tier 3: mandatory senior review (escalation flags present)
            │
            ▼
        [M06] Draft responses + confirmation dialog if downgrading
        [M07] Resolution estimate
        [M16] Analyst decision — logged with reason code
            │
            ▼
        Outcome logged
        → M07 residual tracking
        → M08 calibration monitoring
        → M13 segment FNR update
        → M17 fine-tune candidate if R2 override
        → M20 drift monitoring
        → M21 postmortem if SLA breach or security misroute
```

---

## Checklist

- [ ] Is every pipeline decision logged with enough detail to reconstruct it later?
- [ ] Is the postmortem process documented before an incident occurs, not during one?
- [ ] Does the postmortem template include timeline, contributing factors, non-causes, remediations, and 30-day verification?
- [ ] Are interface design and queue pressure treated as pipeline risks, not just human factors?
- [ ] Is the combined attribution + class mismatch rule active in your routing policy?
- [ ] Are postmortem findings feeding back into training data, routing rules, and interface design?
- [ ] Is every open remediation tracked until verified closed?

---

> The Monday ticket was a test this course has been running since Module 1.
> Every number — the 0.576 probability, the 0.74 entropy, the +0.44 attribution weight on "patch notice" — was a signal.
> The system's job was to make sure a human saw the right signals at the right time.
>
> That is what all of this is for.

--8<-- "_abbreviations.md"
