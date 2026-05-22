# Module 15 — Adversarial Testing: what if someone tries to trick the model?

Every module so far has assumed good-faith users writing honest tickets. Module 15 drops that assumption.

Adversarial testing is the practice of deliberately trying to break your own system before someone else does. In a service desk context, the threat is specific: a user — or an attacker who has compromised a user account — crafts a ticket designed to manipulate the classifier into routing it incorrectly.

The stakes are not symmetric. Tricking the model into *over-escalating* a routine ticket wastes analyst time. Tricking the model into *under-escalating* a security incident leaves an attacker inside the network with a head start.

---

## Three attack types on the Monday ticket scenario

### Attack 1 — Signal suppression

The attacker knows the model routes to `security_incident` when it sees words like "patch", "phishing", "suspicious". So they write around them.

```
Original (honest):
"I've been locked out since this morning. Also my VPN keeps dropping.
This happened right after the security team sent out that patch notice last Friday."

Adversarial (suppressed):
"Morning login issue. Network also unstable. Timing coincides with
recent IT communications."
```

The second ticket describes the same situation. "Patch notice" becomes "recent IT communications." "Security team" disappears. "VPN dropping" becomes "network unstable."

Run both through the classifier:

```
Honest ticket:      account_unlock 0.576 / vpn_issue 0.386 / security_incident 0.039
Suppressed ticket:  account_unlock 0.641 / vpn_issue 0.301 / security_incident 0.018
```

The security signal drops from already-weak 0.039 to near-zero 0.018. Attribution (Module 14) finds nothing to flag. The ticket auto-routes to tier-1.

### Attack 2 — Signal injection

The inverse attack: flood a routine ticket with security-adjacent language to trigger escalation and waste senior analyst time — a denial-of-service against your security review queue.

```
"Locked out of my account. Possibly phishing related? Suspicious activity?
Patch issue? Security incident? Please escalate urgently."
```

The classifier sees every high-weight security token at once:

```
account_unlock 0.31 / vpn_issue 0.09 / security_incident 0.60
```

The ticket escalates. A senior engineer reviews it. It's a forgotten password.

### Attack 3 — Confidence manipulation

A subtler attack targeting the routing thresholds from Module 2. The attacker doesn't try to change the classification — they try to push the entropy into the auto-route zone by writing a ticket that removes ambiguity in a controlled direction.

```
"Password reset needed. Not urgent. No VPN issues. No security concerns.
Standard account access request only."
```

Explicit negations of every competing signal:

```
account_unlock 0.91 / vpn_issue 0.05 / security_incident 0.04
H_norm = 0.28
```

The ticket clears the auto-route threshold. No human reviews it. The attacker's account unlock happens without scrutiny — even if the underlying account was compromised.

---

## How to test for these vulnerabilities

**Red-teaming** is the structured practice of having a team attempt to break the system before deployment. For a service desk classifier, a basic red-team exercise looks like this:

```
Round 1 — Suppression attacks
  → Take 20 confirmed security incidents from history
  → Rewrite each one removing all high-attribution tokens
  → Measure how many now route below the escalation threshold
  → Target: fewer than 2/20 should escape escalation

Round 2 — Injection attacks
  → Take 20 confirmed routine tickets
  → Add security vocabulary to each
  → Measure false escalation rate
  → Target: fewer than 3/20 should escalate

Round 3 — Threshold attacks
  → Take 20 ambiguous tickets (H_norm > 0.60)
  → Rewrite to explicitly negate competing signals
  → Measure how many clear the auto-route threshold
  → Target: none should auto-route after negation rewriting
```

Run this before every model update, and after any significant change to the routing thresholds.

---

## Defences

### Defence 1 — Negation detection

Explicit negations of security signals should *increase* suspicion, not decrease it. A ticket that says "no security concerns" when no one asked is unusual. Add a negation flag:

```
[M15] Adversarial check
    → scan for explicit negation of security tokens
    → "no security issues", "not phishing", "nothing suspicious"
    → if negation count ≥ 2: flag for human review regardless of top_prob
```

This directly blocks Attack 3.

### Defence 2 — Baseline rate anchoring

Even if the classifier assigns low probability to `security_incident`, the routing policy can apply a floor based on the known base rate of security incidents in your environment.

```
Historical base rate of security incidents: 3.2% of all tickets

If P(security_incident) > 0.5 × base_rate (i.e. > 0.016):
    do not auto-route — send to clarification queue
```

Under this rule, the suppressed ticket (0.018) still triggers a review. The signal doesn't have to be strong — it just has to be present above chance.

### Defence 3 — Behavioural consistency check

A ticket that perfectly matches a known adversarial pattern is itself a signal. If the vocabulary of a ticket is suspiciously clean — no contractions, no filler words, no natural variation — flag it:

```
Naturalness score: measure perplexity of the ticket under a language model
Low perplexity (too clean): may indicate constructed/adversarial input
High perplexity (too unusual): may indicate obfuscation attempt
```

This is not a hard block — it's an additional flag for the analyst.

### Defence 4 — Rate limiting on escalation triggers

For injection attacks, the simplest defence is volume-based. If one user account generates three escalation-triggering tickets in 24 hours:

```
[M15] Rate limit check
    → user: account_X
    → escalations in last 24h: 3
    → threshold: 2
    → action: flag account for security review, route all future tickets to human queue
```

---

## Connecting to the Monday ticket

The Monday ticket is honest. But now consider a variant where an attacker who has just compromised an employee account wants to change the password quietly:

```
Adversarial Monday ticket:
"Morning login issue. Network connectivity intermittent.
Not related to any security matters. Standard access request.
No patch issues. Routine reset needed only."
```

Without adversarial defences:
```
account_unlock 0.88  → auto-routes → attacker gets password reset unreviewed
```

With Module 15 defences active:
```
[M15] Negation check: "Not related to any security matters" detected
      "No patch issues" detected
      Negation count: 2 → threshold met
      → override: send to human review regardless of top_prob
```

The auto-route is blocked. An analyst sees the ticket. The unusual phrasing is visible. The account unlock gets scrutiny.

---

## The updated pipeline

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
[M15] Adversarial pre-check  ← NEW
    → negation detection
    → base rate floor check
    → naturalness scoring
    → rate limit check on submitter account
    → any flag: route to human review, bypass auto-route
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
[M14] Attribution flags
    │
    ▼
[M06] Deterministic routing + stochastic drafts
    │
    ▼
[M07] Resolution time estimate
    │
    ▼
Analyst reviews
```

---

## Checklist

- [ ] Have you run suppression, injection, and threshold red-team exercises before deployment?
- [ ] Is negation detection active for explicit negations of security tokens?
- [ ] Is a base rate floor applied so near-zero security probability still triggers review?
- [ ] Is rate limiting active on per-account escalation frequency?
- [ ] Are red-team results logged and compared across model versions?
- [ ] Does your red-team include people who didn't build the system?

---

> The best adversarial testers think like attackers, not engineers.
> The goal is not to find bugs — it's to find the gaps the model doesn't know it has.
>
> Module 16 addresses what happens after the model flags something for human review: how analysts interact with the system, when they should override it, and how their decisions feed back into making the model better.

--8<-- "_abbreviations.md"
