# Module 19 — Cost & Latency: what does this actually cost to run?

Every capability added in the previous modules — RAG retrieval, attribution scoring, adversarial checks, draft generation — consumes time and money. Module 19 is where the engineering decisions of the whole course get priced.

The question is not whether these capabilities are worth having. They are. The question is: *at what point does the cost of running them exceed the value they deliver, and how do you find that point before it finds you?*

---

## Token cost: the Monday ticket end to end

At full pipeline capability (Modules 1–18 active), the Monday ticket consumes:

```
Component                           Tokens (input)    Tokens (output)
────────────────────────────────────────────────────────────────────
System prompt                            600               —
RAG retrieved documents (k=3)            600               —
Conversation history                      80               —
Ticket text                               70               —
Classification output                     —                30
Attribution scores                        —                80
Draft responses (×3)                      —               240
Resolution time estimate                  —                20
────────────────────────────────────────────────────────────────────
Total                                  1,350              370
Grand total: ~1,720 tokens per ticket
```

At current API pricing, that's a known cost per ticket. Multiply by volume:

```
Daily ticket volume:          500
Tokens per ticket:          1,720
Daily token consumption:  860,000

Monthly (22 working days): ~18,900,000 tokens
```

That number needs to sit in front of whoever approves the infrastructure budget before the system goes live — not after.

---

## Latency: where the time goes

Token cost is the budget question. Latency is the user experience question. Your analyst needs a routed ticket with draft responses before their 10 AM call. The pipeline has to finish in time.

```
Step                          Typical latency
────────────────────────────────────────────
Tokenization [M01]                  <10ms
RAG retrieval [M18]               80–200ms
Model inference (classification)  400–900ms
Attribution scoring [M14]         100–300ms
Draft generation (×3) [M06]       600–1200ms
Resolution estimate [M07]           <10ms
────────────────────────────────────────────
Total pipeline:               ~1.2–2.6 seconds
```

For a ticket arriving at 9:47 AM with a 10 AM deadline, 2.6 seconds is fine. For a high-volume queue processing 500 tickets simultaneously, 2.6 seconds per ticket with sequential processing means the last ticket waits 21 minutes. Parallelise or the SLA breaks.

---

## The cost-capability trade-off matrix

Not every ticket needs the full pipeline. Build a tiered execution model:

```
Tier 1 — Fast path (high confidence tickets):
  Conditions: top_prob ≥ 0.80, H_norm ≤ 0.35, no adversarial flags
  Steps run:  M01 tokenize → M03 logits → M02/M04 checks → M06 auto-route
  Skipped:    RAG, attribution, draft generation, resolution estimate
  Tokens:     ~750 per ticket
  Latency:    ~500ms
  % of volume: ~45% of tickets

Tier 2 — Standard path (clarification queue):
  Conditions: top_prob < 0.80 OR H_norm > 0.35
  Steps run:  Full pipeline
  Tokens:     ~1,720 per ticket
  Latency:    ~2.0s
  % of volume: ~48% of tickets

Tier 3 — Full path (escalation + adversarial flags):
  Conditions: H_norm ≥ 0.80 OR adversarial flag OR high-FNR segment
  Steps run:  Full pipeline + extended RAG (k=5) + senior analyst alert
  Tokens:     ~2,200 per ticket
  Latency:    ~2.8s
  % of volume: ~7% of tickets
```

Blended cost across tiers:
```
(0.45 × 750) + (0.48 × 1,720) + (0.07 × 2,200)
= 337.5 + 825.6 + 154
= ~1,317 tokens per ticket (average)
```

vs 1,720 tokens if every ticket ran the full pipeline. A 23% cost reduction by routing tickets to the appropriate processing tier.

---

## The latency budget for the Monday ticket

The Monday ticket is Tier 2 — standard path. At 9:47 AM:

```
09:47:00.000  Ticket arrives
09:47:00.010  Tokenization complete
09:47:00.190  RAG retrieval complete (3 documents returned)
09:47:00.890  Classification complete (logits → softmax → routing decision)
09:47:01.140  Attribution scoring complete (flags generated)
09:47:02.340  Draft responses generated (×3)
09:47:02.350  Resolution estimate calculated
09:47:02.350  Ticket appears in analyst queue with full context
```

Total: **2.35 seconds** from arrival to analyst view. Analyst has 12 minutes and 57 seconds before the 10 AM call.

---

## Where costs grow unexpectedly

**Conversation history accumulation** (Module 1): as users reply to tickets, history grows. A ticket with 10 back-and-forth messages adds ~400 tokens of history. At scale, implement the summarisation strategy from Module 1 — don't let history grow unbounded.

**RAG document length**: if policy documents are retrieved in full rather than as chunks, a single document can be 800–1,200 tokens. Three such documents fill the context window. Enforce chunk size at indexing time (Module 18).

**Draft generation at temperature 1.0**: stochastic generation is inherently slower than deterministic inference. Generating three drafts costs approximately 3× the output token budget. If latency is critical, generate one draft deterministically and offer "generate alternatives" as an on-demand action.

**Concurrent volume spikes**: Monday mornings, post-patch deployments, and incident-adjacent periods all cause volume spikes. A 3× spike with sequential processing triples your latency. Design for burst capacity, not average load.

---

## Monitoring cost and latency in production

These are not set-and-forget metrics. Add them to your weekly operations review alongside accuracy and FNR:

```
Weekly cost and latency report:

Metric                          Target      This week
──────────────────────────────────────────────────────
Avg tokens per ticket            ≤1,400      1,317 ✅
Monthly token projection      ≤20M/mo    ~14.5M ✅
Tier 1 ticket share              ≥40%        45% ✅
P95 latency (Tier 2)              ≤3.0s       2.4s ✅
P95 latency (Tier 3)              ≤4.0s       3.1s ✅
Latency SLA breaches              0            0 ✅
RAG retrieval timeout rate        <0.5%       0.3% ✅
```

If Tier 1 share drops significantly, your ticket population is getting more ambiguous — investigate whether that's real (harder tickets) or a model degradation signal (Module 20).

---

## Checklist

- [ ] Have you calculated total token consumption at expected daily and monthly volume?
- [ ] Is the tiered execution model implemented — are high-confidence tickets skipping expensive steps?
- [ ] Have you parallelised pipeline execution where steps are independent?
- [ ] Are conversation histories being summarised to prevent unbounded token growth?
- [ ] Are RAG documents chunked to enforce maximum retrieval token cost?
- [ ] Are cost and latency metrics in your weekly operations review?
- [ ] Have you capacity-planned for volume spikes (Monday mornings, post-patch periods)?

---

> Cost and latency are not obstacles to building the right system.
> They are constraints that force you to be precise about which capabilities matter most, for which tickets, under which conditions.
>
> Module 20 addresses what happens over time: ticket patterns shift, language evolves, new incident types emerge — and a model that was well-calibrated in week one starts to drift.

--8<-- "_abbreviations.md"
