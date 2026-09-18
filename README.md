# Causal AI for Personalised Decision-Making in Marketing

CS958 MSc dissertation project (University of Strathclyde), supervised by Vinod Kumar Chauhan.

## Purpose

Marketing teams often want to know not just "does this promotion work on average?" but "who
should actually receive it?". Naive comparisons between customers who received a promotion and
those who didn't are biased by confounding (e.g. higher-spending customers are more likely to be
targeted in the first place). This project builds and validates an end-to-end **causal inference
pipeline** that:

- discovers likely confounders directly from data instead of assuming them,
- estimates the **Average Treatment Effect (ATE)** of a promotion using causal discovery output,
- stress-tests that estimate with refutation and sensitivity analysis,
- estimates **personalised, per-customer effects (CATE)** using multiple uplift-modelling methods,
- and converts those personalised effects into a practical **targeting policy** (who should
  receive the promotion).

The same pipeline is first validated on synthetic data (where the true effect is known, so the
method can be checked for correctness) and then applied to a large real-world marketing dataset.

## What has been done

The pipeline (`notebook_connected_final.ipynb`) is built as a set of reusable functions so the
exact same code runs on both datasets:

1. **Synthetic data generation** — a marketing scenario with known, built-in confounding (older /
   higher-spend customers are more likely to receive the promotion) and a known personalisation
   effect (younger / lower-spend customers benefit more), so results can be checked against ground
   truth.
2. **Causal discovery** — an ensemble of PC, FCI and GES algorithms is used to discover which
   variables confound the treatment–outcome relationship, with a domain-knowledge fallback when
   discovery is unreliable.
3. **ATE estimation (DoWhy)** — propensity-score matching, using the confounders identified in
   step 2 as `common_causes`, so the discovery and estimation stages are directly connected.
4. **Refutation tests and sensitivity analysis** — placebo-treatment, random-common-cause and
   data-subset refuters, plus a continuous sensitivity analysis (`sensemakr`) to check how strong
   an unobserved confounder would need to be to explain away the estimated effect.
5. **CATE estimation (EconML)** — five per-customer uplift models (Causal Forest, X-Learner,
   S-Learner, T-Learner, Linear DML) are tuned and compared on the same data.
6. **Targeting policy** — CATE estimates are converted into a simple decision rule for who should
   receive the promotion.
7. **Repeat on real data** — the identical pipeline is re-run on the Criteo Uplift dataset (~14M
   rows), including causal discovery, ATE/CATE estimation, refutation and sensitivity analysis.
8. **End-to-end pipeline runner** — a single function chains every step (discovery → ATE →
   refutation → CATE) for any dataset and column mapping.

Causal DAGs for the Criteo dataset are included as images (`criteo_causal_dag_clean.png`,
`criteo_causal_dag_circular.png`).

## Conclusions

**Synthetic data (ground truth available):**
- The naive treated-vs-untreated purchase rate difference (0.487 vs 0.191) is badly biased by
  confounding, as expected by design.
- After adjusting for the discovered confounders (age, past_spend), the estimated ATE was 0.334,
  close to the true simulated effect of 0.299.
- Refutation tests supported the estimate: a placebo treatment pushed the effect to ~0, and random
  common cause / data subsetting left it stable.
- Sensitivity analysis found the estimate is robust to hidden confounding — an unobserved
  confounder would need to explain over 30% of residual variance in both treatment and outcome to
  bring the effect to zero.
- CATE models recovered the true individual-level effect reasonably well: correlation with the
  true individual treatment effect was 0.606 (Causal Forest) and 0.808 (X-Learner).

**Real data (Criteo Uplift dataset):**
- Causal discovery on the raw feature set found no reliable confounders, and a naive ATE across
  all 12 features (0.0067) looked negligible.
- Using two features (f3, f6) flagged by a corrected discovery pass changed the picture
  substantially — the corrected/IPW-adjusted ATE moved to about 0.009, showing how sensitive the
  estimate is to which variables are treated as confounders, and reinforcing the value of a
  disciplined discovery step rather than adjusting for everything by default.
- Refutation tests (placebo, data subset) on the full-feature-set estimate supported it as stable
  (placebo effect ≈ 0, p = 0.9).
- Across CATE models, **Causal Forest** and **T-Learner** produced the strongest empirical uplift
  ranking (highest Qini AUC and Uplift@10%), outperforming X-Learner, S-Learner and Linear DML.
- The targeting policy validated the CATE ranking directly against outcomes: customers in the
  top decile of estimated uplift had an observed visit rate of 29.8%, versus 6.8% in the bottom
  decile — evidence that the estimated personalised effects meaningfully separate customers who
  do and don't respond to the promotion.
- Overall, the pipeline generalises from synthetic to real data: the same discovery → ATE →
  refutation → CATE → targeting sequence produces internally consistent, refutation-tested, and
  practically actionable results on both.

## Dataset

Real-world experiments use the **Criteo Uplift Prediction Dataset**:
https://ailab.criteo.com/criteo-uplift-prediction-dataset/

The raw dataset files are not included in this repository (they are several GB); download them
from the link above and place them in the project folder to reproduce the real-data sections of
the notebook.

## Repository contents

- `notebook_connected_final.ipynb` — the full connected pipeline (synthetic + Criteo).
- `criteo_causal_dag_clean.png`, `criteo_causal_dag_circular.png` — discovered causal graphs for
  the Criteo dataset.

Large raw data files and the local Python virtual environment (`causal_env/`) are excluded via
`.gitignore` — see the Dataset section above for how to obtain the data.
