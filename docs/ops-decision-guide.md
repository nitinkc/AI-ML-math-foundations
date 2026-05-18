# Ops Decision Guide (Sprint 4)

This page turns math concepts into repeatable operating controls for IT teams.

## 1) Routing and confidence policy

- Use class probability + [entropy], not top class label alone.
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

| Signal | Threshold | Action |
|---|---:|---|
| Weighted KPI delta | >= +0.01 | Expand rollout |
| Weighted KPI delta | (-0.01, +0.01) | Hold and monitor |
| Weighted KPI delta | <= -0.01 | [rollback] |

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

