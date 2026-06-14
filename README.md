# Credit Risk PD Modeling: Survival Analysis vs Logistic Regression

A credit risk project modeling **time-to-default** (not just default/no-default) using survival
analysis on ~2 million Lending Club loans, benchmarked against a standard logistic regression
approach, and connected to IFRS 9 expected credit loss (ECL) provisioning.

## Motivation

Most junior credit risk projects use logistic regression to predict whether a loan defaults.
This project instead asks **when** a loan is likely to default — using Kaplan-Meier curves and
a Cox Proportional Hazards model — and shows why this framing produces better discrimination
and maps more naturally onto how banks actually provision for credit losses under IFRS 9.

## Dataset

- Source: [Lending Club Loan Data](https://www.kaggle.com/datasets/wordsforthewise/lending-club) (public, Kaggle)
- ~2.07 million loans after cleaning (from an original ~2.26 million)
- Loan statuses used: Fully Paid, Current, Charged Off, Default

## Methodology

### 1. Data Cleaning & Feature Engineering
- Dropped irrelevant/high-cardinality columns (IDs, URLs, free-text fields)
- Converted `issue_d` and `last_pymnt_d` to dates and derived a `duration` column
  (months between loan issuance and last observed payment)
- Created a binary `event` column: `1` = Charged Off / Default, `0` = Fully Paid / Current (censored)
- Cleaned `term` (e.g. "36 months" → 36) and `emp_length` (mapped text categories to 0–10,
  with a separate `emp_length_missing` flag for the ~39% of borrowers coded as "n/a")
- One-hot encoded categorical variables: `grade`, `home_ownership`, `purpose`, `verification_status`

### 2. Kaplan-Meier Survival Analysis
- Overall survival curve: ~31% cumulative default rate by month 60
- Stratified by loan grade (A–G): clean, monotonic separation — grade A loans default far less
  and far later than grade G loans
- Estimates beyond month 60 are unreliable due to small remaining sample size

### 3. Cox Proportional Hazards Model
- Fit on a 200,000-loan sample with 35 covariates
- **Concordance: 0.69**
- Grade dominates: relative to grade A, hazard ratios rise from **1.94 (grade B)** to
  **5.22 (grade G)** — a grade G loan defaults at roughly 5x the rate of a grade A loan,
  holding other factors constant
- `int_rate` (HR 1.06) and `dti` add incremental predictive power beyond grade
- `home_ownership` is not statistically significant
- `verification_status` shows a counterintuitive **positive** association with default
  (likely a selection effect — Lending Club tends to verify income for riskier applications)

### 4. Logistic Regression Benchmark
- Same 200k sample, same 35 features, binary target (`event`)
- **AUC: 0.631** — notably lower than the Cox model's concordance
- The gap is attributed to how each model treats "Current" (still-active) loans: logistic
  regression labels them identically to "Fully Paid" loans, while the Cox model correctly
  treats them as censored observations that have only partially revealed their outcome

### 5. IFRS 9 / Expected Credit Loss Application
Using the grade-level Kaplan-Meier curves, 12-month and 36-month default probabilities were
extracted directly:

| Grade | PD (12mo) | PD (36mo) | Cliff Ratio (36mo / 12mo) |
|-------|-----------|-----------|----------------------------|
| A     | 1.4%      | 6.9%      | 4.92                       |
| B     | 3.5%      | 14.2%     | 4.07                       |
| C     | 6.4%      | 22.6%     | 3.51                       |
| D     | 10.1%     | 31.0%     | 3.08                       |
| E     | 14.4%     | 39.4%     | 2.75                       |
| F     | 19.7%     | 47.7%     | 2.41                       |
| G     | 25.8%     | 54.4%     | 2.11                       |

**Finding**: the "cliff ratio" — how much larger lifetime PD is compared to 12-month PD —
shrinks as grade worsens. For grade A, lifetime risk is nearly 5x the 12-month figure (risk is
back-loaded); for grade G, it's only ~2x (risk is front-loaded). This implies that under IFRS 9,
a Stage 1 → Stage 2 migration (12-month ECL → lifetime ECL) has a proportionally larger impact
on provisioning for higher-grade loans than for lower-grade loans.

## Limitations & Future Work

- Cox model and logistic regression were fit on a 200k-row sample, not the full 2.07M dataset
- The proportional hazards assumption was not formally tested
- "Fully Paid" and "Current" loans are both treated as censored; a competing-risks framework
  (Fine-Gray or cause-specific hazards) would model payoff and default as distinct outcomes
- Only Probability of Default (PD) is modeled — Loss Given Default (LGD) and Exposure at
  Default (EAD), the other components of ECL, are out of scope
- Tree-based survival models (e.g. Random Survival Forest) were considered as a non-linear
  benchmark but deprioritized in favor of interpretable models, consistent with how credit
  risk models are typically validated in practice


