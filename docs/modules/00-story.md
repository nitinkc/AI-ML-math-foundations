#  The anchor scenario

It's Monday morning. A ticket lands in your service desk queue:

> **Subject:** Can't log in — urgent
> 
> **Body:** Hi, I've been locked out of my account since this morning. Also my VPN keeps
> dropping every few minutes. Not sure if these are related but I need access ASAP for
> a client call at 10am. This happened right after the security team sent out that
> patch notice last Friday.

Every module connects back to the same Monday morning ticket:

One ticket. Three possible things going on:

| Intent              | What it would mean                                                  | Resolution time | Who handles it                              |
|:--------------------|:--------------------------------------------------------------------|:----------------|:--------------------------------------------|
| `account_unlock`    | User is locked out — password reset or re-enablement needed         | ~4 min          | Tier-1 agent                                |
| `vpn_issue`         | VPN client broken after the Friday patch                            | ~18 min         | Tier-2 network team                         |
| `security_incident` | The "patch notice" was a phishing email; account may be compromised | ~45 min         | Senior engineer + audit trail + possibly HR |

**The stakes are not equal.** Misrouting a security incident as a password reset
doesn't just waste time — it leaves an active attacker inside the network while
a tier-1 agent cheerfully sends a password reset link.

This is the problem your AI copilot has to solve correctly, **every time, at scale**.

Every module adds one more layer of precision to that solution.

---

## modules

| # | File                             | Topic                          | What it teaches                                                                           |
|:---|:---------------------------------|:-------------------------------|:------------------------------------------------------------------------------------------|
| 00 | 00-story.md                      | The Monday Ticket              | The scenario, the stakes, the three possible intents                                      |
| 01 | 01-tokenization.md               | Tokenization                   | How the model reads the ticket; token budgets; what gets cut                              |
| 02 | 02-probability.md                | Probability                    | Classifier confidence; expected handle time; routing thresholds                           |
| 03 | 03-logits-softmax.md             | Logits & Softmax               | Where probabilities come from; why margin matters                                         |
| 04 | 04-entropy.md                    | Entropy                        | Measuring spread of uncertainty; the three-tier routing policy                            |
| 05 | 05-variance-stddev.md            | Variance & Std Dev             | System stability; the go/no-go deployment gate                                            |
| 06 | 06-determinism.md                | Determinism vs Stochastic      | Which steps need reproducibility; which benefit from variation                            |
| 07 | 07-regression.md                 | Regression                     | Predicting resolution time; residual tracking                                             |
| 08 | 08-classification-calibration.md | Calibration                    | Are the probabilities actually trustworthy                                                |
| 09 | 09-correlation-causation.md      | Correlation vs Causation       | Are you fixing the right thing; confounder detection; pilot design                        |
| 10 | 10-sampling-controls.md          | Sampling Controls              | Temperature, top-p, top-k; when to constrain generation                                   |
| 11 | 11-guardrails-thresholds.md      | Guardrails & Thresholds        | Hard rules layered on top of probabilistic outputs                                        |
| 12 | 12-evaluation-in-production.md   | Evaluation in Production       | How to measure a live system; metrics that matter                                         |
| 13 | 13-bias-fairness.md              | Bias & Fairness                | Disaggregated accuracy; FNR by segment; segment-specific thresholds; proxy discrimination |
| 14 | 14-interpretability.md           | Interpretability               | Which words drove the classification; why the night-shift gap exists at the token level   |
| 15 | 15-adversarial-testing.md        | Adversarial Testing            | What if a user crafts a ticket to trick the classifier; red-teaming your pipeline         |
| 16 | 16-human-in-the-loop.md          | Human-in-the-loop              | When the model defers; how analyst decisions feed back; override logging                  |
| 17 | 17-fine-tuning.md                | Fine-Tuning                    | When prompt engineering stops being enough; what fine-tuning actually changes             |
| 18 | 18-rag.md                        | Retrieval-Augmented Generation | Giving the model access to your knowledge base at inference time                          |
| 19 | 19-cost-latency.md               | Cost & Latency                 | Token costs at scale; latency budgets; the accuracy vs speed trade-off                    |
| 20 | 20-drift-retraining.md           | Drift & Retraining             | How production data shifts over time; when and how to retrain                             |
| 21 | 21-incident-postmortem.md        | Incident & Postmortem          | When the pipeline fails; how to investigate, document, and fix it                         |

---

## How the modules connect

```
FOUNDATION (what the model does)
01 Tokenization → 02 Probability → 03 Logits & Softmax → 04 Entropy

STABILITY (can you trust the system)
05 Variance → 06 Determinism → 08 Calibration → 09 Correlation vs Causation

CAPABILITY (what the system produces)
07 Regression → 10 Sampling Controls → 11 Guardrails → 17 Fine-Tuning → 18 RAG

EVALUATION (is it working)
12 Evaluation in Production → 13 Bias & Fairness → 14 Interpretability → 20 Drift & Retraining

SAFETY (what can go wrong)
15 Adversarial Testing → 16 Human-in-the-loop → 21 Incident & Postmortem

OPERATIONS (real-world constraints)
19 Cost & Latency → runs as a thread through all the above
```
