# Predicting Early Customer Churn and Optimal Timing of Personalized Communication in a Multi-Format Loyalty Program

**Bachelor's thesis** — Sviatoslav Sharak, Ukrainian Catholic University, 2026

---

## Overview

This repository contains the full analytical pipeline for a churn prediction study conducted on the !FEST loyalty program — a Ukrainian holding that operates restaurants, bars, coffee shops, cafés, bakeries, delivery outlets, and retail stores.

The study addresses two questions simultaneously:

- **WHO** — which newly registered clients are at risk of churning during the 90-day onboarding window?
- **WHEN** — at what point in the onboarding trajectory should a retention message be sent to maximize its effect?

Both questions are studied on real transactional and CRM data: 4.2M transactions, 12M marketing messages, and 2.1M recorded conversions across 174 business units.

---

## Key Results

| Metric | Current system | Model (RF) | Delta |
|---|---|---|---|
| Targeting precision | 78.4% | 90.5% | +12.1pp |
| Recall | 57.5% | 62.6% | +5.1pp |
| False positives / 1,000 msgs | 216 | 95 | −56% |
| Lift vs. random | 1.04× | 1.20× | +0.16× |
| Retention rate (IR=10%) | 28.86% | 29.24% | +0.40pp |
| Additionally retained clients | — | +277 / cohort | — |

**Timing finding**: conversion rate in week 1 (days 0–7) is 31.6%, dropping to 16.5% in week 2 and 9.2% after week 5. The current system already concentrates 62.3% of retention messages in week 1, leaving timing improvement modest but measurable (+0.48pp to +1.43pp at 10–30% IR gain).

---

## Research Design

### Churn Definition

Non-contractual churn — the client does not generate any transaction during days 91–120 after their first purchase. This 30-day observation buffer avoids right-censoring artifacts.

Onboarding window: **90 days from first transaction** (not from registration). 58% of clients register at a physical POS terminal at the moment of their first purchase, making registration date a noisy proxy.

### Churn Taxonomy

Hazard function analysis revealed two structurally different fader groups:

- **Acute faders** (days 0–7): 70.2% of all churners. Hazard peaks at h(0)=0.381 and drops to h(7)≈0.03. These clients disengage before any communication can realistically reach them.
- **Chronic faders** (days 8–90): 29.8% of churners. Hazard stabilizes at 0.013–0.030. Optimal intervention window: days 30–45 (68.5% of chronic faders are still active at day 30, 51.7% at day 45).

### Working Population

```
New clients registered Dec 2030 – Sep 2031   125,742
Clients with at least one transaction        118,813   (−6,929 = 5.6%)
Clients with ≥118 days of observation        117,296   (−1,517)
```

Post-onboarding observation: min 118 days, max 408 days, median 268 days.

### Models

Three classifiers trained on a **temporal train/val/test split** (no data leakage from future):

| Model | AUC-ROC | Precision | Recall | F1 |
|---|---|---|---|---|
| Random Forest | best | 90.5% | 62.6% | — |
| XGBoost | — | — | — | — |
| Logistic Regression | — | — | — | — |

The main deployment model is **Random Forest** (best AUC). Two additional models are trained for the early-snapshot (30-day) and multi-visit (30-day, tx≥2) targeting scenarios — see NB08 and NB09.

---

## Feature Engineering

**17 transactional features** — RFM-based signals normalized per format:
- `tx_count`, `total_spend`, `avg_check`, `median_gap`, `days_active`
- `recency_normalized` — days since last visit relative to format's median inter-visit gap
- `visit_fulfillment` — fraction of expected visits actually completed
- `format_diversity` — number of distinct business unit types visited

**11 communication features** — CRM engagement signals:
- `msg_count`, `msg_seg_retention`, `msg_seg_incentive`, `msg_seg_engagement`
- `conversions`, `conv_rate`, `days_since_last_msg`, etc.

**2 profile features**: `reg_day_of_week`, `reg_month` (registration date excluded to prevent leakage)

**Normalization note**: `median_gap` was computed on the full dataset rather than train-only — leakage impact measured at <1.5% performance difference, acknowledged in the thesis as negligible.

---

## Repository Structure

```
notebooks/                   Analysis pipeline — run in order 00 → 09
  00_eda.ipynb               Exploratory data analysis: dataset sizes, distributions, data quality
  00_validation.ipynb        Data integrity checks: FK consistency, zero-amount transactions
  01_onboarding_window.ipynb 90-day window derivation from gap distributions and cohort validation
  02_churn_label.ipynb       Churn labelling: 30-day buffer, class distribution by segment
  03_feature_engineering.ipynb  RFM + format-normalised features, leakage prevention
  04_modeling.ipynb          RF / XGBoost / LR training, temporal split, hyperparameter tuning
  05_interpretation.ipynb    SHAP feature importance: beeswarm, dependence, waterfall plots
  06_timing_analysis.ipynb   Hazard function, chronic phase zoom, Kaplan-Meier curves by timing group
  07_business_results.ipynb  WHO + WHEN business comparison, retention rate simulation
  08_early_model.ipynb       30-day all-clients snapshot model
  09_multivisit_model.ipynb  30-day multi-visit deployment model (tx≥2 clients only)

reports/figures/             Final figures exported for the thesis (PNG)
  hazard_function.png        Dual-panel hazard + cumulative fading chart
  kaplan_meier_timing.png    KM survival curves by message timing group
  conversion_by_week.png     Message conversion rate by week (weeks 1–13)
  targeting_quality_comparison.png  Precision / Lift / FP comparison
  roc_pr_curves.png          ROC and Precision-Recall curves for all 3 models
  shap_beeswarm.png          SHAP beeswarm plot (global feature importance)
  shap_dependence.png        SHAP dependence plots for top features
  shap_waterfall.png         SHAP waterfall for individual client profiles
  client_type_breakdown.png  Three client segments: retained / fading / one-timers
  rate_of_return.png         Chronic faders alive % by day (day 8–90)
  model_mv_comparison.png    Multi-visit model metrics comparison

src/
  config/                    Shared path constants (DATA, INTERIM, REPORTS)
  data/                      Data preparation scripts

data/                        Raw and intermediate data — gitignored
  raw/                       Original BigQuery CSV exports
  interim/                   Processed CSVs, trained models (.pkl), SHAP values (.npy)
  processed/                 Final feature matrix and labels
```

---

## Dataset (not included)

Raw data is not published due to a confidentiality agreement with !FEST. To reproduce the pipeline, place the following BigQuery exports under `data/raw/`:

| File | Rows | Description |
|---|---|---|
| `clients.csv` | 841,434 | Client registry with dual ID system |
| `transactions.csv` | 4,246,735 | All transactions Dec 2030 – Jan 2032 |
| `messages.csv` | 12,039,701 | CRM messages (Lokal, Email, Viber, SMS) |
| `conversions.csv` | 2,167,100 | Message → transaction conversion links |
| `message_templates.csv` | 1,087 | Template metadata with 5 segment types |
| `business_units.csv` | 174 | Location metadata with 12 format types |

Dates in the dataset are shifted by a constant offset for anonymization purposes.

---

## Reproducing the Pipeline

```bash
# 1. Clone and set up environment
git clone <repo>
cd thesis-churn-sharak
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2. Place raw exports
mkdir -p data/raw
# copy your BigQuery CSVs to data/raw/

# 3. Run notebooks in order
jupyter lab
# Execute: 00_eda → 00_validation → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09
```

All intermediate outputs (CSVs, `.pkl` models, `.npy` SHAP arrays, intermediate plots) are written to `data/interim/` and gitignored. Final thesis figures are exported to `reports/figures/`.

---

## Dependencies

```
pandas        tabular data processing
numpy         numerical operations
matplotlib    all visualizations (no seaborn)
scikit-learn  Random Forest, Logistic Regression, metrics
xgboost       XGBoost classifier
shap          model interpretation
jupyter       notebook environment
ipykernel     kernel for Jupyter
tqdm          progress bars
pyarrow       fast CSV/Parquet I/O
python-dotenv environment variable management
```

Install: `pip install -r requirements.txt`

Python version: see `.python-version`.

---

## Methodological Notes

**Why temporal split instead of random split**: random splitting would allow the model to train on clients whose outcomes are temporally correlated with test clients, inflating AUC estimates. All clients with `first_tx_date` before the cutoff form train/val; the rest form the test set.

**Why no survival library**: the Kaplan-Meier estimator in NB06 is implemented from scratch using the standard product-limit formula. This was a deliberate choice to avoid treating the KM curve as a black box in an academic context.

**Why the 1.3× timing multiplier was rejected**: an earlier draft estimated a 1.32× lift from timing optimization based on the hazard curve (68.5%/51.7%). This was invalidated when message-level analysis showed that 62.3% of retention messages to chronic faders are already sent in week 1 (median message day = 1.0). The difference between the model's timing and the current system's timing is too small to justify a point estimate. A sensitivity analysis (±10/20/30% IR) is used instead.

**Kaplan-Meier selection bias**: the "Days 30+" timing group shows S(90)=0.263 (lower survival = more churn), which looks counterintuitive. The reason is selection bias: only clients who are already active survive to day 30 and receive a late message. The valid comparison is Days 0-7 (56.9% return rate) vs. No messages (53.7% return rate) = +3.2pp from early messaging.

---

## Author

Sviatoslav Sharak — Bachelor's student, Ukrainian Catholic University, Faculty of Applied Sciences

Thesis supervisor: [supervisor name]

Academic year: 2025–2026
