# Module 12: Evaluation in Production

## What's the big idea?

You built an AI system. Tested it offline. Looks great on the validation set.

So you deploy it... and the real world isn't the test set.

**Production evaluation** is your safety net. It asks: *Is this actually working in the real world, right now?*

Don't just measure accuracy. Measure what matters operationally:
- How fast are tickets getting resolved?
- Are escalations actually accurate?
- How often do analysts override the system?
- Are users happy?

These numbers tell you if you should expand, pause, or rollback.

## The math in plain terms

Most production systems need to balance multiple metrics. Use a weighted KPI:

$$
KPI_{weighted}=\sum_{i=1}^{n} w_i m_i
$$

- $m_i$ is each metric you care about (accuracy, SLA hit-rate, etc.)
- $w_i$ is how much you weight that metric (based on business importance)

**Translation:** Some metrics matter more than others. Accuracy might be 40% of your score, SLA hit-rate 35%, analyst override rate 25%. Sum them up, you get one number that tells you go/no-go.

## Real-world scenario: Phased rollout gate

You roll out your new AI copilot in phases:

**10% traffic:** KPIs look good. Resolution time ↓, overrides ↓, SLA ↑.
→ Go to 25%.

**25% traffic:** Still looking solid.
→ Go to 50%.

**50% traffic:** Oh no. Override rate shoots up. Incident miss-rate climbs. Weighted KPI drops below your gate threshold.
→ HOLD. Don't go further.

Now you investigate, fix the issue, retest. Only then proceed.

If you didn't have this gate, you'd have rolled out broken system to 100% of traffic. Disaster.

## How to try this

Run `notebooks/math-foundations/12_eval_in_production.ipynb` and:

- Combine multiple metrics into a single weighted KPI
- Compare baseline vs new model over weekly windows
- Simulate go/hold/rollback decisions based on KPI thresholds

## The traps to avoid

❌ **The trap:** Optimize one metric (accuracy) while everything else falls apart  
✅ **The smart move:** Monitor a balanced KPI set tied to business outcomes

## Checklist for you

- [ ] Are rollout gates defined *before* launch?
- [ ] Are KPIs monitored by segment (e.g., queue, urgency, geography)?
- [ ] Is metric drift detected with alert thresholds?
- [ ] Are rollback criteria explicit and pre-approved?
- [ ] Do you have a post-incident review process?

--8<-- "_abbreviations.md"
