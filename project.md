# Real-Time Fraud Risk Scoring for New User E-Commerce Transactions
---

## 1. Problem Definition

E-commerce platforms lose money to fraudulent orders placed with stolen payment credentials. The loss is not merely the order value: a chargeback typically costs the merchant the goods, the fulfilment and shipping, and a processing fee, so the realised loss on an undetected fraudulent order commonly exceeds twice its face value. Blocking legitimate customers, however, is also costly, forfeiting the margin on the sale and damaging the customer relationship.

**Operational decision.** When a newly registered user attempts their first purchase, which of three experiences should the platform serve in real time: complete the order normally, require a step-up verification such as an SMS code, or hold the order for manual review?

**Goal.** Estimate the probability that a new user's first transaction is fraudulent, using only information available at the instant of purchase.

**Unit of analysis.** One user's first transaction.

## 2. Dataset Description

The project uses the **Fraud E-commerce** dataset published on Kaggle (`vbinh002/fraud-ecommerce`). 

It comprises two tables.

**`Fraud_Data.csv`** — 151,112 rows and 11 columns, one row per user, describing that user's first transaction:

| Variable | Description |
|---|---|
| `user_id` | Unique user identifier |
| `signup_time` | Account creation timestamp (GMT) |
| `purchase_time` | Purchase timestamp (GMT) |
| `purchase_value` | Item cost (USD) |
| `device_id` | Device identifier, unique per physical device |
| `source` | Acquisition channel: Ads, SEO, or Direct |
| `browser` | Browser used |
| `sex`, `age` | User demographics |
| `ip_address` | Numeric IP address |
| `class` | Target: 1 = fraudulent |

**`IpAddress_to_Country.csv`** — 138,846 rows mapping numeric IP ranges (`lower_bound_ip_address`, `upper_bound_ip_address`) to a country, joined by interval lookup rather than equality.

The class distribution is approximately 9.5% fraudulent to 90.5% legitimate — imbalanced, but far less extreme than transaction-level fraud datasets, which keeps a wide range of algorithms viable. The primary table has no missing values, and `user_id` is unique per row.

**Why this dataset suits the brief.** Unlike pre-cleaned benchmark data, the raw columns are not directly modellable. `signup_time`, `purchase_time`, `device_id`, and `ip_address` carry no usable pattern as-is; their predictive content only emerges through derived features such as elapsed time between signup and purchase, and the number of distinct users sharing a device or IP. The analytical work is therefore genuine rather than decorative.

**Limitations.** It covers a single platform in 2015, contains only first transactions so no behavioural history exists, and includes demographic attributes whose operational use raises fairness and legal questions. The IP-to-country table will not resolve every address, producing a residual unmapped category that must be handled explicitly rather than dropped.

## 3. Proposed Methodology

**Data understanding.** Profile distributions, class balance, temporal coverage, and the repetition structure of `device_id` and `ip_address`. Examine the distribution of signup-to-purchase elapsed time by class, which prior analyses of this data identify as a dominant signal.

**Feature engineering.**
- *Elapsed time* between signup and purchase, in seconds, plus a binary indicator for near-instantaneous ("flash") purchases.
- *Temporal context*: hour of day, day of week, and week of year of purchase.
- *Entity velocity*: count of distinct users sharing each `device_id`; the same for `ip_address`.
- *Geography*: country resolved from the IP interval table, plus an explicit `unmapped` level.
- *Value and demographics*: transformed `purchase_value`, age bands, and encoded `source`, `browser`, `sex`.

**Leakage control.** Velocity features computed over the entire dataset use information from the future, since a device's total user count includes users who had not yet appeared at the time of the transaction being scored. The project will compute these features **causally**, counting only prior occurrences relative to each transaction's `purchase_time`, and will additionally compute the naive global version in order to quantify the performance inflation that the shortcut produces. Publicly available analyses of this dataset routinely use the global version without comment, so this comparison is a genuine contribution rather than a formality. Any target encoding of country will likewise be fitted within cross-validation folds only.

**Validation design.** Because the deployed model scores future transactions, the primary evaluation uses a **temporal split**: train on the earlier portion of the period, test on the later. A conventional stratified random split will be reported alongside it, and the divergence between the two treated as a finding about how optimistic random splitting is for time-ordered data. All preprocessing will sit inside a `scikit-learn` Pipeline so that transformers are fitted per fold.

**Implementation note.** The IP-to-country interval join will be implemented with sorted bounds and vectorised binary search (`numpy.searchsorted`), giving log-linear rather than quadratic complexity — a material difference against 151,112 rows and 138,846 ranges, and one that most published notebooks on this dataset handle inefficiently with row-wise loops.

**Modelling.** Three baselines establish the floor: approve-everything, a single rule flagging flash purchases, and a rule flagging devices shared by more than one user. Candidate models then span regularised logistic regression, decision tree, random forest, and gradient boosting (LightGBM or XGBoost), tuned by randomised search.

**Imbalance handling.** Class weighting, SMOTE applied strictly within training folds, and no resampling with post-hoc threshold optimisation will be compared under identical validation. At a 9.5% positive rate, the hypothesis is that threshold optimisation on a calibrated model will match or beat resampling.

**Evaluation.** Precision–recall AUC is the primary ranking metric, with the no-skill baseline at 0.095; ROC-AUC is reported for comparability with published work. The decisive metric is expected value-weighted cost under the two-threshold policy. Calibration will be assessed by reliability curves and Brier score, with isotonic regression applied if required, since the policy bands depend on absolute probabilities. Precision at fixed review capacity will be reported to reflect a realistic manual-review budget. All metrics will carry confidence intervals from repeated resampling.

**Interpretation, fairness, and robustness.** SHAP values and partial dependence will characterise the drivers. A fairness audit will disaggregate true-positive and false-positive rates by sex, age band, and country, with particular attention to country, since routing orders to friction by geography is operationally and ethically sensitive. The report will close with a discussion of adversarial decay: the elapsed-time feature is directly controllable by an attacker, so any deployed model built on it degrades as fraudsters adapt.

## 4. Expected Outcomes

**Artefacts.** A reproducible analysis notebook with fixed seed and pinned dependencies, a final written report, a model card documenting intended use and subgroup performance, and an AI Use Log appendix.

**Anticipated findings.** Elapsed time between signup and purchase and device-sharing counts are expected to dominate feature importance. Discrimination is expected to be strong, with ROC-AUC plausibly in the low-to-mid 0.80s, but a substantial share of that performance is expected to be recoverable by two simple rules — a result that would argue for a hybrid rules-plus-model system rather than a black box.

**Anticipated methodological results.** Causally computed velocity features are expected to underperform globally computed ones, quantifying a leakage effect that is widely overlooked in public work on this dataset. Temporal splitting is expected to yield lower and more honest estimates than random splitting. The two-threshold policy is expected to outperform any single-threshold policy on expected cost, and the optimal thresholds are expected to sit well away from 0.5.

**Possible null result.** It remains possible that gradient boosting delivers negligible gain over the simple rules once cost is properly accounted for. That would be a legitimate finding, reported plainly rather than concealed by selective metric choice.

## 5. Planned AI Tool Usage Strategy

AI tools will be used deliberately and logged in full. An **AI Use Log** will record the date, tool, purpose, what was accepted, what was rejected and why, and how each output was verified.

| Phase | Planned use | Verification |
|---|---|---|
| Framing | Stress-test the decision framing and cost structure | Cross-check against fraud management literature |
| Feature design | Generate candidate derived features beyond my own list | Test empirically; retain only what validates |
| Implementation | Scaffold the interval join and pipeline code | Execute and hand-verify against brute-force results on a sample |
| Critique | Adversarial review targeting leakage and invalid comparisons | Treat as hypotheses to test, not conclusions |
| Writing | Structure and tighten prose | All substantive claims authored and verified by me |

**Explicit boundaries.** AI tools will not choose the evaluation metric, set the cost parameters, produce any numerical result, or supply citations. Every reference will be independently located and read. No figure or statistic will appear in the report unless reproduced by the committed code.

### How AI tools will assist in this project

AI tools will act as a fast, tireless collaborator rather than a substitute for analysis. Their highest-value contribution is expected to be adversarial: asking a model to mount the strongest available critique of my own design surfaces problems such as the temporal leakage in velocity features far earlier than solitary work would, and that particular issue shapes the entire validation strategy of this project. They will also compress low-judgment effort, drafting exploratory plotting code, explaining unfamiliar library behaviour, and proposing a broader set of candidate features than I would generate alone, which preserves time for the evaluation and interpretation work that determines whether the model is actually decision-useful. Every AI contribution enters the project as a hypothesis rather than a result: code is executed and checked against independent implementations, suggested features are retained only if they survive validation, and factual claims are confirmed against primary sources. The rejections recorded in the AI Use Log are intended to be as informative as the acceptances, since they document the points at which independent judgment overrode a fluent but incorrect suggestion.

---

## References

Fraud E-commerce dataset. Kaggle. https://www.kaggle.com/datasets/vbinh002/fraud-ecommerce

[Add: the assigned course text and any fraud-detection or cost-sensitive-learning papers you cite, in the required style.]
