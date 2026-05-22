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

---

## Ops Controls

This page turns math concepts into repeatable operating controls for IT teams.

## 1) Routing and confidence policy

- Use class probability + entropy, not top class label alone.
- Define confidence bands per risk class (`auto`, `review`, `abstain`).
- Document fallback behavior for every abstain case.

## 2) Decoding profile governance

- Maintain named presets for each use case (safe vs creative).
- Version control profile changes (`temperature`, `top_p`, `top-k`).
- Re-run evaluation checks after each preset update.

## 3) Stability and drift controls

- Track mean and standard deviation for repeated-run quality.
- Alert on sustained variance increase or KPI regression.
- Segment by queue, severity, and ticket type before diagnosis.

## 4) Rollout gate template

Use a simple gate table:

| Signal             |   Threshold    | Action           |
|:-------------------|:--------------:|:-----------------|
| Weighted KPI delta |    >= +0.01    | Expand rollout   |
| Weighted KPI delta | (-0.01, +0.01) | Hold and monitor |
| Weighted KPI delta |    <= -0.01    | [rollback]       |

## 5) Minimum audit log fields

- model version
- prompt template version
- decoding profile
- confidence outputs
- guardrail decision path
- final action and human override flag

## 6) Weekly review checklist

- Did any threshold changes increase incident risk?
- Did calibration drift by class or queue?
- Did any policy increase abstain volume unexpectedly?
- Are rollback triggers still realistic for current traffic?

## 7) Common anti-patterns

- One global threshold for all classes
- Correlation-based policy changes without pilot evidence
- Silent decoding changes outside release process
- KPI dashboards without explicit gate decisions

--8<-- "_abbreviations.md"
