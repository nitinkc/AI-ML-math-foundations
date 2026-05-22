# Module 18 — Retrieval-Augmented Generation: giving the model live knowledge

Fine-tuning (Module 17) updates the model's weights to reflect patterns in your data. But weights are fixed at training time. By the time a fine-tuned model is deployed, the world has already moved on: new patches have been released, new policies written, new incident patterns emerged.

Retrieval-Augmented Generation (RAG) solves a different problem: it gives the model access to information it could not have been trained on, at the moment it needs it.

---

## The gap that RAG fills

On the Monday ticket, the model had to classify based entirely on the ticket text and whatever patterns it had learned during training. It didn't know:

- What the Friday patch notice actually said
- Whether other users had submitted similar tickets that morning
- What the current policy is for MFA-related lockouts
- Whether there's an active incident already open for the VPN issue

A model without RAG is reasoning in a vacuum. With RAG, it reasons with context.

---

## How RAG works

At inference time — when the ticket arrives, before classification — a retrieval step runs:

```
1. Ticket arrives: "locked out + VPN dropping + patch notice last Friday"

2. Retrieval step:
   → convert ticket to embedding vector
   → search knowledge base for nearest neighbours
   → return top-3 most relevant documents

3. Retrieved documents:
   [Doc 1] Patch notice from Friday:
           "Security patch KB-2847 addresses authentication flow.
            Known side effect: MFA tokens may require re-enrollment.
            Estimated affected users: ~12%"

   [Doc 2] Incident log opened 08:23 Monday:
           "Multiple VPN disconnections reported — under investigation
            by network team. Likely related to KB-2847."

   [Doc 3] Policy: Account lockout + patch correlation:
           "If lockout coincides with a patch deployment within 72h,
            treat as potential security incident pending investigation."

4. Augmented prompt sent to classifier:
   [System] + [Retrieved docs] + [Ticket text] → classification
```

The model now knows what the patch was, that an incident is already open, and that policy mandates escalation. The ticket that previously scored 0.039 on `security_incident` now scores much higher.

---

## The knowledge base: what goes in it

RAG is only as good as what you retrieve. For a service desk copilot, the knowledge base should contain:

```
Document type               Update frequency    Priority
────────────────────────────────────────────────────────
Active incident logs        Real-time           Critical
Patch and change notices    Per deployment      High
Security bulletins          Per release         High
Routing policies            Per policy update   High
Past resolved tickets       Daily               Medium
Known error patterns        Weekly              Medium
User account history        Real-time           Situational
```

The key insight: documents that change frequently are exactly the ones the model can't have learned in training. Real-time incident logs, today's patch notice, this week's security bulletin — these are the highest-value RAG inputs.

---

## Retrieval quality: the precision-recall trade-off

Retrieving too few documents risks missing the relevant one. Retrieving too many floods the context window (Module 1) with noise.

For the Monday ticket, test retrieval quality:

```
Retrieval experiment — top-k documents:

k=1:  Retrieved patch notice (correct) — missed incident log
k=3:  Retrieved patch notice + incident log + policy (all correct)
k=5:  Retrieved patch notice + incident log + policy +
      unrelated ticket from 3 weeks ago +
      general VPN troubleshooting guide (noise added)
k=10: Context window fills with low-relevance documents,
      classifier performance degrades
```

For this ticket type, k=3 is optimal. This will vary by ticket category — run the experiment for each major ticket type and set k per category.

---

## Token budget impact

RAG has a direct cost in Module 1 terms. Each retrieved document consumes tokens:

```
Without RAG:
  System prompt:          ~600 tokens
  Conversation history:    ~80 tokens
  Current ticket:          ~70 tokens
  Reserved output:        ~500 tokens
  ────────────────────────────────────
  Total:                ~1,250 / 4,000 tokens

With RAG (k=3, avg doc length 200 tokens):
  System prompt:          ~600 tokens
  Retrieved documents:    ~600 tokens  ← new
  Conversation history:    ~80 tokens
  Current ticket:          ~70 tokens
  Reserved output:        ~500 tokens
  ────────────────────────────────────
  Total:                ~1,850 / 4,000 tokens
```

Still within budget. But if retrieved documents are long (full policy documents, detailed incident logs), they can push you past the limit. Chunk documents into retrievable segments of ~150-200 tokens before indexing. Retrieve chunks, not whole documents.

---

## RAG and the Monday ticket: before and after

**Without RAG:**
```
Ticket text only → logits [3.2, 2.8, 0.5] → probabilities [0.576, 0.386, 0.039]
H_norm = 0.74 → clarification queue
Analyst required
```

**With RAG (k=3 retrieval):**
```
Ticket + patch notice + incident log + policy →
logits [2.1, 2.6, 3.4] → probabilities [0.198, 0.328, 0.474]
Top class: security_incident (0.474)
H_norm = 0.81 → escalate immediately

Pipeline decision: escalate, do not wait for clarification
Analyst notified: high-entropy security escalation with supporting documents
```

The model didn't get smarter. It got better information. The classification shifted from uncertain-account-unlock to probable-security-incident because the context now included the policy that says *patch + lockout within 72h = treat as security incident*.

---

## What RAG does not fix

RAG retrieves — it does not reason. If the retrieved documents are:

- Outdated: retrieval will surface wrong information confidently
- Irrelevant: noise in context degrades classification
- Contradictory: the model may average across conflicting signals incorrectly
- Missing: if the relevant document isn't in the knowledge base, RAG adds nothing

Maintain the knowledge base as carefully as you maintain the model. A stale knowledge base is worse than no RAG — it gives the model false confidence.

---

## Checklist

- [ ] Is your knowledge base segmented by document type with appropriate update frequencies?
- [ ] Have you chunked documents into retrieval-appropriate sizes (≤200 tokens per chunk)?
- [ ] Have you measured optimal k per ticket category?
- [ ] Have you recalculated token budgets after adding retrieved documents?
- [ ] Do you have a staleness policy — how old can a document be before it's excluded from retrieval?
- [ ] Are retrieved documents shown to analysts alongside the classification, for transparency?
- [ ] Are retrieval misses (no relevant document found) logged for knowledge base gap analysis?

---

> RAG gives the model knowledge it couldn't have been trained on.
> Fine-tuning gives the model patterns it kept getting wrong.
> Together they address two different kinds of gap — and both leave the cost and latency questions for Module 19.

--8<-- "_abbreviations.md"
