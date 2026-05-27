# AI Math Foundations — Personal Reference

Quick-access index. All content lives in four nav pages below.

---

## What each page is

| Page               | What it covers                                                                                                |
|:-------------------|:--------------------------------------------------------------------------------------------------------------|
| **Modules 01–12**  | All 12 lesson pages — token → logit → probability → entropy → variance → regression → guardrails → evaluation |
| **Reference**      | All formulas, code explanations, and distribution guide in one place                                          |
| **Ops Cheatsheet** | Routing policy, rollout gates, audit fields, review checklist, and self-check questions                       |
| **Notebook Labs**  | Links to all 13 Jupyter notebooks with lab descriptions                                                       |

---

## Concept chain (the single thread tying everything together)

## Phase 1 - Language units and probability basics

1. Ticket text is **tokenized** — budget determines how much history you can keep.
2. Model outputs **logits** — raw preference scores per intent.
3. **Softmax** converts logits to probabilities — normalized confidence values.

## Phase 2 - Uncertainty, variability, and control

4. 4.**Entropy** measures how spread-out those probabilities are — high entropy → escalate.
5. **Temperature / top-p / top-k** control how stochastic the sampling is.

6. **Variance and std dev** catch run-to-run instability across repeated prompts.

## Phase 3 - Prediction models used in AI systems

8. **Regression** predicts numeric outcomes like resolution hours from token count.
8. **Classification + calibration** tunes thresholds so auto-routing is trustworthy.
9. **Correlation ≠ causation** — segment before making policy changes.

## Phase 4 - LLM sampling and decoding controls

11. **Guardrail thresholds** map confidence to auto / review / abstain actions.
11. **Weighted KPI** gates rollout decisions: go / hold / rollback.

## Phase 5 - Measurement and production decisions

---

## Key distribution cheatsheet

| What you observe | Distribution |
|---|---|
| Single routing decision (right/wrong) | Bernoulli |
| Correct routes out of N tickets | Binomial |
| Average quality score across many runs | Normal (bell curve) |
| Time between ticket arrivals | Exponential |
| Incident count per hour | Poisson |
| Token selection at high temperature | Approaches Uniform |

---

## Sprint outcomes at a glance

**Sprint 1 (M1–M4):** tokens → probabilities → softmax → entropy.

**Sprint 2 (M5–M8):** stability → determinism → regression → calibration.

**Sprint 3 (M9–M12):** causation → sampling controls → guardrails → production evaluation.

--8<-- "_abbreviations.md"
