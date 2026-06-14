# UBA Kenya Retail Credit — Early-Warning Model

**Author:** Vincent Mbira Muiruri  
**Assessment:** UBA Kenya Retail Credit Early-Warning Model  
**Objective:** Predict `default_next_90d` using only information available as of `snapshot_date`, enabling the collections team to prioritize the **top 5% highest-risk customers** each month for early intervention.

---

## Repository Structure

```
├── uba_kenya_early_warning_model.ipynb   # Full modeling workflow
├── predictions.csv                       # Final scored output for submission
└── README.md                             # Project documentation
```

---

## How to Run

1. Place the datasets in the same directory as the notebook:
   - `uba_kenya_candidate_train.csv`
   - `uba_kenya_candidate_scoring.csv`

2. Open `uba_kenya_early_warning_model.ipynb` in Jupyter Notebook or JupyterLab.

3. Run all cells sequentially from top to bottom. The notebook will:
   - perform data quality checks and drop leakage columns
   - train the models
   - evaluate performance
   - generate predictions for the scoring dataset

4. The final predictions will be saved as `predictions.csv`.

**Environment Requirements**

- Python 3.9+
- pandas, numpy, scikit-learn, xgboost, matplotlib

No internet access is required.

---

## Validation Design

A **strict time-based validation strategy** was used to mimic real-world deployment conditions, where a model is trained on historical data and applied to future observations.

| Split      | Date Range              | Rows   |
|------------|-------------------------|--------|
| Training   | 2025-01-01 → 2025-10-31 | 15,225 |
| Validation | 2025-11-01 → 2025-12-27 | 2,756  |

This prevents temporal leakage and ensures predictions are based only on information available as of `snapshot_date`.

Cross-validation used **GroupKFold on `customer_id`**, ensuring the same customer did not appear in both training and validation folds, which eliminates customer-level data leakage.

---

## Data Quality Issues

Four data issues were identified during exploratory analysis.

### Issue 1: Structural Missing Values

Several financial fields contain high rates of missing values that are structural in nature — they are missing because the customer does not hold the relevant product, not due to data collection errors.

| Column               | Missing Count | Missing % |
|----------------------|---------------|-----------|
| `credit_limit_kes`   | 12,599        | 70.07%    |
| `utilization_ratio`  | 12,599        | 70.07%    |
| `collateral_value_kes` | 3,637       | 20.23%    |
| `loan_to_value`      | 3,637         | 20.23%    |

**Resolution:** These fields were filled with **0** to indicate the absence of a product feature, rather than imputed with a central tendency measure which would misrepresent the underlying data.

---

### Issue 2: Leakage Columns

Three columns were identified as future leakage — they encode information that would not be available at the time of prediction (`snapshot_date`) and were therefore **excluded from all models**.

#### (a) `collection_contacted_after_snapshot`

This variable indicates whether the collections team contacted the customer **after** the snapshot date. At prediction time, this action has not yet occurred, making it a future value. The column also shows an unrealistically high correlation of **0.509** with the target variable `default_next_90d`, which further confirms it encodes outcome information.

#### (b) `chargeoff_indicator`

This variable is an accounting action recording the outcome of the default process. A charge-off can only occur after a default has been confirmed, meaning it is a downstream consequence of the very event being predicted. Its correlation with the target variable is **0.492**. Including this feature would cause the model to learn from the outcome it is trying to predict.

#### (c) `days_to_next_payment`

This variable indicates when the customer will make their next payment and is derived from a **future payment date**. At `snapshot_date`, this information is not known. Its correlation with the target variable is **0.962** — near-perfect — which is a strong indicator of direct leakage from the future.

**Resolution:** All three columns were dropped before model training.

---

### Issue 3: Extreme Skewness in Financial Variables

Several continuous financial variables exhibit significant right skew, which can distort distance-based or linear models.

| Column                    | Skewness |
|---------------------------|----------|
| `loan_balance_kes`        | 1.06     |
| `avg_daily_balance_30d_kes` | 2.09   |
| `inflow_30d_kes`          | 1.20     |

**Resolution:** These variables were log-transformed prior to model training to reduce the influence of extreme values.

---

### Issue 4: Missing February 2025 Snapshots

The snapshot distribution reveals that **February 2025 is entirely absent** from the training data, while all other months are present.

| Month   | Snapshot Count |
|---------|----------------|
| 2025-01 | 2,710          |
| 2025-02 | *missing*      |
| 2025-03 | 1,384          |
| 2025-04 | 1,397          |
| 2025-05 | 2,762          |
| 2025-06 | 1,403          |
| 2025-07 | 1,414          |
| 2025-08 | 1,385          |
| 2025-09 | 1,415          |
| 2025-10 | 1,355          |
| 2025-11 | 1,346          |
| 2025-12 | 1,410          |

This gap may reflect a data pipeline failure or extraction error. It introduces a temporal blind spot in training, and any seasonal default patterns occurring in February will not be captured by the model.

**Monitoring recommendation:** Investigate the root cause of the February gap before production deployment. If the gap reflects a systemic pipeline issue, validate that the scoring dataset and future monthly extracts are complete.

---

## Modeling Approach

Three models were trained and evaluated:

| Model               | Mean CV AUC | Validation AUC |
|---------------------|-------------|----------------|
| Logistic Regression | **0.7091**  | **0.7506**     |
| Gradient Boosting   | 0.7023      | 0.7179         |
| XGBoost             | 0.6748      | 0.7281         |

**Selected Model: Logistic Regression**

Reasons for selection:
- Highest validation AUC and most stable cross-validation results
- Strong interpretability in a regulated financial environment — coefficients can be audited and explained to risk committees and regulators
- Lower computational complexity, making monthly retraining straightforward
- Gradient Boosting and XGBoost underperformed on validation despite higher model complexity, suggesting the linear model better captures the signal in this dataset

---

## Operational Performance

The model is designed to support a collections team that can contact only the **top 5% highest-risk customers each month**.

### Validation Results

| Metric         | Value      |
|----------------|------------|
| ROC AUC        | **0.7506** |
| PR-AUC         | **0.3694** |
| Recall@Top 5%  | **20.35%** |

### Threshold Selection

The top 5% threshold was selected to match the operational capacity constraint stated in the brief. In the validation period, this corresponds to the 95th percentile of predicted probabilities. The tradeoff is as follows:

- **Benefit:** Collections resources are concentrated on the customers most likely to default, avoiding wasted outreach on low-risk accounts.
- **Cost:** 79.65% of future defaulters fall outside the top 5% and will not receive proactive intervention. Expanding the contact list to the top 10% would increase recall but may exceed collections capacity.

### Interpretation

Out of **339 future defaulters** in the validation period, **69 were identified within the top 5% highest-risk customers**. This represents a **4× improvement over random selection**, which would be expected to capture approximately 5% of defaulters in a 5% sample.

> "If we contact the top 5% each month, we expect to catch roughly 1 in 5 customers who will default in the next 90 days — while limiting outreach to only the highest-risk segment."

---

## Operational Recommendations

The predicted risk scores support three operational actions within UBA Kenya's retail credit portfolio:

1. **Collections prioritization** — Customers ranked in the top 5% risk segment are prioritized for proactive outreach by collections or relationship managers before accounts enter serious delinquency.

2. **Credit limit management** — For revolving credit products, high-risk customers may have their credit limits reviewed or temporarily reduced to limit additional exposure.

3. **Early restructuring discussions** — Customers flagged as high risk may benefit from early engagement to explore repayment plans or restructuring options before accounts deteriorate further.

---

## Failure Modes & Monitoring

### Failure Mode 1: Data Drift

Customer behavior patterns may shift due to macroeconomic shocks, changes in lending policy, or sector-specific income disruptions. If the feature distributions at scoring time diverge significantly from those seen during training, model predictions will degrade.

**Monitoring approach:**
- Track monthly model metrics: AUC and Recall@Top5%
- Monitor feature distribution shifts using **Population Stability Index (PSI)** on key variables, particularly `utilization_ratio`, `days_past_due`, and `missed_payments_3m`
- Flag and investigate any month where PSI exceeds 0.2 for a key feature

### Failure Mode 2: Product Mix Changes

New credit products or changes in underwriting policy may alter the relationship between observable features and default risk. For example, a new secured loan product would introduce collateral-related features with no historical default data to learn from.

**Monitoring approach:**
- Monitor score distribution changes month-over-month
- Track drift in key product-level variables: `utilization_ratio`, `loan_to_value`, `collateral_value_kes`
- Retrain the model when a significant product or policy change is introduced, rather than waiting for performance degradation to occur

---

## Predictions Output

The scoring dataset contains **4,659 rows**. The model generates probability predictions stored in `predictions.csv`.

**Format:**

```
customer_id,snapshot_date,pred_default_next_90d
C00001,2026-01-01,0.34
C00002,2026-01-01,0.12
...
```

**Score distribution summary:**

| Metric | Value |
|--------|-------|
| Min    | 0.09  |
| Mean   | 0.50  |
| Max    | 1.00  |

These probabilities are used to rank customers each month and identify the top-risk segment for collections outreach.

---

## Conclusion

The developed early-warning model successfully identifies customers likely to default within the next 90 days using only information available at the snapshot date. By removing leakage columns, addressing structural missing values, and applying a rigorous time-based validation design, the model is built to reflect real deployment conditions.

By focusing on the **top 5% highest-risk customers**, UBA Kenya can detect over **20% of future defaults**, enabling proactive collections strategies and improved credit risk management.