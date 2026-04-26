# Predictive Modeling of Early Customer Churn in a Multi-Format Loyalty Program

Bachelor's thesis codebase — Sviatoslav Sharak, 2026.

## Research summary

The study builds a WHO+WHEN framework for early churn management in the !FEST multi-format loyalty program (restaurants, bars, coffee shops, cafés, bakeries, delivery, retail). Key results:

- **90-day onboarding window** derived from three independent analyses of time-to-second-visit distributions.
- **Format-normalised features** (`recency_normalized`, `visit_fulfillment`) encode raw behavioural signals relative to each format's empirical inter-visit rhythm.
- **Two-phase fader decomposition**: 70.2% of churners disengage within 7 days (acute phase, unreachable); 29.8% fade gradually (chronic phase, reachable days 30–45).
- **Deployable 30-day multi-visit model**: AUC 0.731, precision 87.8%, 122 FP per 1,000 messages vs. 218 FP for the current system.
- **Retention simulation**: +2,109 additionally retained clients per cohort at 10% intervention response rate.

## Repository structure

```
notebooks/          Analysis pipeline (run in order 00 → 09)
  00_eda.ipynb          Exploratory data analysis
  00_validation.ipynb   Data quality / FK integrity checks
  01_onboarding_window  90-day window derivation
  02_churn_label        Churn labelling strategy
  03_feature_engineering  RFM + format-normalised features
  04_modeling           Random Forest / XGBoost training
  05_interpretation     SHAP feature importance
  06_timing_analysis    Hazard function + Kaplan-Meier timing
  07_business_results   Business comparison vs. current system
  08_early_model        30-day all-clients snapshot model
  09_multivisit_model   30-day multi-visit deployment model

reports/figures/    Final figures used in the thesis (PNG)

src/
  config/           Shared path constants
  data/             Data preparation scripts

data/               Raw and intermediate data (gitignored)
  raw/              Original BigQuery exports
  interim/          Processed CSVs, model artifacts, interim plots
  processed/        Final processed datasets
```

## Reproducing the pipeline

1. Place raw BigQuery CSV exports under `data/raw/` (see notebook cell 1 for expected filenames).
2. Install dependencies: `pip install -r requirements.txt`
3. Run notebooks in order: `00_eda` → `01` → ... → `09`.

All intermediate outputs (CSVs, `.pkl` models, `.npy` SHAP values) are written to `data/interim/` and are gitignored.

## Dependencies

See `requirements.txt`. Key packages: `pandas`, `scikit-learn`, `xgboost`, `shap`, `lifelines`, `matplotlib`, `seaborn`.
