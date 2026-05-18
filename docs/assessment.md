# Curriculum Assessment (Sprint 4)

Use this page as a practical checkpoint after each phase.

## How to use

- Spend 10 to 15 minutes per section.
- Answer in plain English first.
- Then justify your answer with one formula or metric.

## Phase checks

### Phase 1 (Modules 1 to 4)

1. A ticket request exceeds token budget. What do you trim first and why?
2. A model predicts intent with probability 0.62. Why is this not "truth"?
3. Two intents have close softmax scores. What routing action is safest?
4. Top class is unchanged, but [entropy] rises sharply. What should policy do?

### Phase 2 (Modules 5 to 8)

1. Two prompt profiles have equal mean quality but different standard deviation. Which do you deploy?
2. Where must [determinism] be enforced in a service desk pipeline?
3. Your regression model underestimates high-severity ticket duration. What do you investigate?
4. Precision improves after threshold tuning, but recall drops. When is this acceptable?

### Phase 3 (Modules 9 to 12)

1. You find correlation between long prompts and escalations. What stops you from claiming causation?
2. You changed temperature and top-p in production. What must be monitored immediately?
3. How do you set `tau_low` and `tau_high` for high-risk queues?
4. Weighted KPI dropped by 0.03 after rollout expansion. What should the gate decision be?

## Scoring rubric

- `Green`: answer links math signal to an operational decision and fallback path.
- `Yellow`: answer includes math but lacks clear policy/governance action.
- `Red`: answer relies on intuition only and omits risk controls.

## Team retro prompts

- Which module changed how we make deployment decisions?
- Which metric do we trust least today, and why?
- What one guardrail will reduce risk most in the next sprint?

--8<-- "_abbreviations.md"

