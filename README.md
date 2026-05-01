# Home Credit Default Risk: Full Expected Loss Pipeline

**Author:** Shubham Malpani  
**Background:** Quantitative Credit Risk Analyst | MSc Banking and International Finance, Bayes Business School | CFA Level II  
**Contact:** https://www.linkedin.com/in/shubham-malpani97/ | shubham.malpani97@gmail.com

---

## Project Overview

This project builds a production-style end-to-end Expected Loss pipeline using the Home Credit Default Risk dataset, combining Probability of Default (PD), Loss Given Default (LGD), and Exposure at Default (EAD) models into a single framework:
Expected Loss = PD × LGD × EAD

---

## Dataset

**Source:** [Home Credit Default Risk | Kaggle](https://www.kaggle.com/c/home-credit-default-risk/data)

| File | Description |
|------|-------------|
| application_train.csv | 307,511 loan applications with default outcome |
| bureau.csv | Credit bureau records per borrower |
| bureau_balance.csv | Monthly bureau DPD status snapshots |
| installments_payments.csv | Scheduled vs actual payment history |
| previous_application.csv | Previous Home Credit loan applications |

**Note:** Raw data files are not included in this repository due to size constraints. Download directly from Kaggle using the link above and place in the project root directory.

---

---

## Summary

### Data Understanding
- 307,511 loan applications, 122 features
- Default rate: 8.07%, class imbalance ratio 11:1
- 106 numeric features, 16 categorical features
- 50 columns with more than 30% missing values identified and documented

### Exploratory Data Analysis
- Target variable distribution confirming class imbalance
- Default rate analysis across 7 categorical variables
- KDE distribution analysis across 5 numeric variables
- EXT_SOURCE_2 and EXT_SOURCE_3 identified as strongest visual separators
- DAYS_EMPLOYED anomaly (value 365,243) identified as unemployed placeholder

### Preprocessing and Feature Selection
- Stratified 80/20 train-test split preserving 8.07% default rate
- Feature engineering: DAYS_EMPLOYED anomaly flag, age conversion, debt burden ratio
- Missing value imputation with EXT_SOURCE missingness flags and BUILDING_INFO_MISSING flag
- Manual WOE/IV pipeline built from scratch across 91 candidate features
- 43 features selected via IV threshold of 0.02

### PD Model

| Metric | Train | Test |
|--------|-------|------|
| AUROC | 0.7381 | 0.7406 |
| Gini | 0.4762 | 0.4812 |
| KS Statistic | 0.3552 | 0.3641 |
| Gini Gap | -0.0051 | |

- Logistic regression with L2 regularisation, C=0.001 selected via 5-fold stratified cross-validation
- No overfitting detected: Gini gap of -0.005
- Scorecard built: PDO=20, Base Score=600, Base Odds=1
- Non-defaulter mean score: 678 | Defaulter mean score: 659

### LGD Model

**Engineering Approach:**  
LGD engineered as unpaid principal fraction using AMT_CREDIT as denominator:
LGD = max(0, AMT_CREDIT - Total Payments) / AMT_CREDIT
This avoids interest conflation present in AMT_INSTALMENT-based approaches.

**Two-Stage Model:**

| Stage | Description | Train | Test |
|-------|-------------|-------|------|
| Stage 1 | Binary: P(LGD > 0) via Logistic Regression | AUROC 0.624 | AUROC 0.625 |
| Stage 2 | Continuous: E(LGD \| LGD > 0) via OLS on logit-transformed LGD | RMSE 0.278 | RMSE 0.279 |

**Combined:** Final LGD = P(LGD > 0) × E(LGD | LGD > 0)  
Mean actual LGD: 43.7% | Mean predicted LGD: 47.9%

**Known Limitation:**  
Bureau-based default date identification covered only 1.1% of defaulters due to incomplete bureau reporting in Home Credit's operating markets. AMT_CREDIT denominator approach adopted as the most robust available proxy.

### EAD Model

**Engineering Approach:**  
CCF (Credit Conversion Factor) is engineered as the remaining balance fraction:
CCF = max(0, AMT_CREDIT - Total Payments) / AMT_CREDIT
OLS regression on logit-transformed CCF.  
Mean actual CCF: 39.9% | Mean predicted CCF: 16.9%

**Known Limitation:**  
AMT_PAYMENT includes both principal and interest components, overstating repayment of principal and understating EAD. A dedicated principal balance field would be required for production-grade EAD modelling.

### Expected Loss

| Metric | Original LGD | Improved LGD |
|--------|-------------|--------------|
| Mean LGD | 8.9% | 43.1% |
| Total ECL Provision | 22,790,908 | 133,872,222 |
| EL Rate (EL/EAD) | 0.13% | 0.77% |
| Mean EL per borrower | 370.57 | 2,176.68 |

**Score Band EL Rates:**

| Score Band | Borrowers | Mean PD | Mean LGD | EL Rate |
|------------|-----------|---------|----------|---------|
| 600-630 | 1,119 | 30.3% | 18.3% | 9.36% |
| 630-650 | 6,544 | 18.5% | 9.6% | 3.56% |
| 650-665 | 11,452 | 11.5% | 5.1% | 1.16% |
| 665-680 | 15,206 | 7.3% | 2.7% | 0.40% |
| 680-700 | 18,104 | 4.3% | 1.5% | 0.14% |
| 700-760 | 9,078 | 2.1% | 0.7% | 0.03% |

**IFRS 9 Staging:**
- Stage 1 (PD < 1%): 213 borrowers
- Stage 2 (1% ≤ PD < 20%): 58,239 borrowers
- Stage 3 (PD ≥ 20%): 3,051 borrowers

**Basel III:** Model Gini of 0.481 exceeds minimum 0.35 threshold. Model passes Basel IRB validation requirements.

---

## Key Methodological Decisions

| Decision | Rationale |
|----------|-----------|
| WOE/IV encoding built from scratch | Transparency and deeper understanding over black-box libraries |
| Stratified train-test split | Preserves 8.07% default rate in both sets given 11:1 class imbalance |
| SimpleImputer fitted on training set only | Prevents data leakage from test set into preprocessing |
| L2 regularisation with C=0.001 | Most conservative choice when all C values produce identical AUROC on WOE-encoded features |
| AMT_CREDIT denominator for LGD | Avoids interest conflation in AMT_INSTALMENT, produces economically realistic mean LGD of 43% |
| Two-stage LGD model | Separates zero-loss from positive-loss borrowers before modelling loss magnitude |

---

## Results Summary

| Model | Metric | Value |
|-------|--------|-------|
| PD | Test Gini | 0.481 |
| PD | Test AUROC | 0.741 |
| PD | Test KS | 0.364 |
| LGD | Stage 1 AUROC | 0.625 |
| LGD | Stage 2 RMSE | 0.279 |
| EL | Portfolio EL Rate | 0.77% |
| EL | Total ECL Provision | 133,872,222 |

---
