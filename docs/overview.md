# Curriculum Overview: AI Math Foundations

The full curriculum now covers Sprint 1 through Sprint 3, plus Sprint 4 polish assets for assessment and governance.

## How to use this track

Use each module in three passes:

1. Plain-English model (what this means in real work)
2. Minimal math (one formula at a time)
3. Notebook lab (small runnable checks)

This sequence is designed for IT professionals who need decision clarity, not deep theory.

## Running scenario used across all modules

We use one story through the whole curriculum:

**IT Service Desk Copilot**

- Inputs: tickets, chat messages, and email requests
- Tasks: intent routing, summary draft, escalation risk hints
- Decisions: auto-route vs route-to-human, confidence thresholds, and stability checks

## Sprint 1 outcomes

By the end of Modules 1 to 4, you should be able to:

- Explain how text becomes tokens and why token budget affects cost/latency
- Interpret model probabilities as confidence signals, not truth
- Convert [logits] to probabilities using [softmax]
- Use [entropy] to detect uncertainty and trigger fallback behavior

## Learning path in Sprint 1

- Module 1: `modules/01-tokenization.md`
- Module 2: `modules/02-probability.md`
- Module 3: `modules/03-logits-softmax.md`
- Module 4: `modules/04-entropy.md`
- Notebook guide: `notebooks.md`

## Notebook labs in Sprint 1

- `notebooks/math-foundations/00_warmup.ipynb`
- `notebooks/math-foundations/01_tokenization.ipynb`
- `notebooks/math-foundations/02_probability.ipynb`
- `notebooks/math-foundations/03_logits_softmax.ipynb`
- `notebooks/math-foundations/04_entropy.ipynb`

## Sprint 2 outcomes

By the end of Modules 5 to 8, you should be able to:

- Measure run-to-run instability with [variance] and standard deviation
- Decide where [determinism] is required vs where stochastic generation is acceptable
- Use [regression] to estimate operational effort like resolution time
- Tune [threshold]s with [classification] metrics and basic [calibration] checks

## Learning path in Sprint 2

- Module 5: `modules/05-variance-stddev.md`
- Module 6: `modules/06-determinism.md`
- Module 7: `modules/07-regression.md`
- Module 8: `modules/08-classification-calibration.md`

## Notebook labs in Sprint 2

- `notebooks/math-foundations/05_variance_stddev.ipynb`
- `notebooks/math-foundations/06_determinism.ipynb`
- `notebooks/math-foundations/07_regression.ipynb`
- `notebooks/math-foundations/08_classification_calibration.ipynb`

## Sprint 3 outcomes

By the end of Modules 9 to 12, you should be able to:

- Separate [correlation] signals from causal claims in incident analysis
- Configure sampling controls by risk profile and reliability goals
- Implement confidence-based guardrails with explicit fallback behavior
- Evaluate production rollouts with weighted [KPI] gates and rollback criteria

## Learning path in Sprint 3

- Module 9: `modules/09-correlation-causation.md`
- Module 10: `modules/10-sampling-controls.md`
- Module 11: `modules/11-guardrails-thresholds.md`
- Module 12: `modules/12-evaluation-in-production.md`

## Notebook labs in Sprint 3

- `notebooks/math-foundations/09_correlation_causation.ipynb`
- `notebooks/math-foundations/10_sampling_controls.ipynb`
- `notebooks/math-foundations/11_guardrails_thresholds.ipynb`
- `notebooks/math-foundations/12_eval_in_production.ipynb`

## Sprint 4 polish deliverables

- `assessment.md` for quick knowledge checks across all phases
- `ops-decision-guide.md` for governance-ready rollout and threshold playbooks

## Practical rules for this sprint

- Keep formulas small and tied to one operational decision.
- If confidence is diffuse, escalate rather than over-automate.
- Use notebook checks to validate intuition before changing production settings.

--8<-- "_abbreviations.md"

