# UBA Kenya DS Assessment — Data Dictionary

Each row is a **monthly customer snapshot**. The goal is to predict `default_next_90d` **using only information available as of `snapshot_date`**.

## Identifiers / Time
- `customer_id`: Unique customer identifier (grouping key; customers have multiple rows).
- `snapshot_date`: Snapshot date (use for time-based validation; beware time leakage).

## Customer attributes
- `county`: County of primary branch/relationship.
- `sector`: Primary income source / business segment.
- `employment_type`: Employment category.
- `gender`: Self-declared gender (optional; treat carefully).
- `age`: Age in years.
- `kyc_level`: KYC tier (Tier1–Tier3).
- `relationship_length_months`: Months since relationship start.
- `salary_inflow_flag`: 1 if salary-like inflow is detected.

## Product / exposure
- `product_type`: Personal_Loan, SME_Loan, Overdraft, Credit_Card
- `loan_balance_kes`: Current outstanding balance.
- `credit_limit_kes`: Limit (only for overdraft / credit card, else null).
- `utilization_ratio`: Balance/limit (only for limit products, else null).
- `interest_rate`: Current nominal rate (decimal).
- `loan_term_months`: Term in months (0 for revolving products).
- `collateral_value_kes`: Collateral value (often missing for revolving products).
- `loan_to_value`: Balance/collateral (null when collateral is null).

## Bureau / obligations
- `bureau_score`: Synthetic bureau score (300–850).
- `num_active_loans`: Number of active obligations (approx).
- `num_closed_loans`: Number of closed obligations (approx).

## Repayment / delinquency
- `days_past_due`: Current DPD (0–180).
- `missed_payments_6m`: Missed payments in last 6 months (0–6).

## Account behavior (last 30 days)
- `avg_daily_balance_30d_kes`
- `inflow_30d_kes`
- `outflow_30d_kes`
- `digital_txn_count_30d`
- `atm_withdrawals_30d`
- `branch_visits_30d`

## ⚠️ Leakage traps (should be excluded in a correct solution)
These are **not reliably available at decision time** and are constructed from future operational outcomes.
- `collection_contacted_after_snapshot`
- `chargeoff_indicator`
- `days_to_next_payment` (derived from a future payment date)

## Label
- `default_next_90d`: 1 if default event occurs within next 90 days after snapshot_date; else 0.
