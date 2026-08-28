# Retail Analytics Suite

**A four-part analytics pipeline — EDA, churn prediction, customer lifetime value, and revenue forecasting — that turns 13 months of raw UK e-commerce transactions into a dollar-quantified retention and revenue strategy.**

---

## What this project does

Built on the UCI Online Retail II dataset (~525K transaction line items, Dec 2009–Dec 2010), this suite answers four connected business questions for an e-commerce retailer: _where is revenue actually coming from, which customers are about to leave, what is each customer worth going forward, and how much revenue should we expect next month?_

It's structured as a pipeline, not four disconnected notebooks. The EDA stage cleans the raw transaction log and engineers a 27-feature customer profile (RFM, spend trajectory, product diversity, return behavior). That feature set feeds a churn model, whose output feeds a CLV model, whose combined output produces a four-quadrant action matrix telling a retention team exactly who to contact and how much to spend on them. Revenue forecasting runs as a parallel, standalone module on the monthly aggregates.

## Key results

- **£44,168 net expected return per intervention period** from a combined churn-retention system (£2,140/month spend), by routing the top-20%-revenue segment to a dedicated high-value model and everyone else to a general model
- **84.6% of future high-value revenue captured** by the CLV model (CatBoost, R² = 0.68, MAE = £1,429) when targeting the top ~19% of customers by predicted spend
- **£10.27M in transaction revenue** profiled across 4,383 registered customers and 4,276 products, quantifying a 90/10 (not 80/20) revenue concentration — the single finding that shaped every downstream modeling decision
- Honest caveat, stated up front: churn labels are behaviorally inferred (3-month inactivity), not confirmed cancellations, and the revenue forecast explicitly flags its own confidence interval as statistically unreliable below 24 months of history — both are treated as findings, not hidden

## How it works

```
Raw transactions (UCI Online Retail II, ~525K rows)
        │
        ▼
1. Retail Analysis  →  cleaning, KPIs, diagnostics, RFM feature engineering
        │  (outputs a 27-feature customer table)
        ▼
2. Churn Prediction →  general model (LR) + dedicated high-value model
        │  (outputs churn probability per customer)
        ▼
3. CLV Prediction   →  predicts 6-month forward spend, blends with churn
        │  probability into a 4-quadrant action matrix
        ▼
   Ranked, dollar-weighted retention target list

4. Revenue Forecasting → runs independently on monthly revenue aggregates
```

- **Data in:** raw invoice-level transactions (Invoice, StockCode, Quantity, Price, CustomerID, Country, InvoiceDate)
- **Feature engineering:** recency/frequency/monetary, early vs. late-window spend, spend trend, return rate, product diversity, activity flags, country
- **Modeling:** classification (churn), regression (CLV), time-series (revenue) — each with a documented baseline-to-complex progression, not a single model dropped in
- **Output:** per-customer churn risk + predicted CLV + action-matrix segment, plus a monthly revenue forecast with explicit scenario ranges

## The interesting part

The most useful finding in this project wasn't a metric — it was a null result that changed the architecture. Three algorithms (Logistic Regression, Random Forest, XGBoost) were run against the churn problem in increasing order of complexity, and **all three converged at ~0.77 ROC-AUC regardless of tuning.** That's not a modeling failure — it's diagnostic. When a linear model and a gradient-boosted ensemble land on the same score, the remaining signal in the data has been exhausted; more algorithmic firepower can't extract information that isn't there.

The actual fix wasn't a better model — it was noticing that the aggregate 0.77 AUC was hiding a structural problem: every general model, tree-based or linear, achieved 0–25% Pareto capture on high-value churners specifically, because that segment is only 2.5% of the dataset and gets statistically outvoted during training. **Splitting the top-20%-revenue customers into their own dedicated model** (still just Logistic Regression) pushed CV ROC-AUC from 0.74 to 0.83 and Pareto capture to 100% at the chosen threshold — same algorithm, same features, just trained on the right population. That segmentation decision is what the £44K/period figure above is actually built on.

The same instinct shows up in the CLV and forecasting stages: a two-stage churn-gate-plus-regression CLV model was tried first and scored **R² = –0.832** (worse than guessing the mean) because the churn model's 3-month inactivity label didn't align with the CLV window's 6-month definition — a label-misalignment bug that a single-stage CatBoost model sidestepped entirely. In the forecasting notebook, a 95% confidence interval was computed, then explicitly flagged in the same output as _not a real confidence interval_ — with only 6 validation points, it's just the min/max of observed errors, not a statistically robust bound. Reporting a number as unreliable, in a client-facing analysis, is a decision worth as much as any model choice.

## Results

**Churn (general model — Logistic Regression):**

| Metric                            | Value         |
| --------------------------------- | ------------- |
| ROC-AUC                           | 0.7736        |
| PR-AUC                            | 0.6369        |
| Precision / Recall                | 52.5% / 76.2% |
| Revenue capture                   | 54.27%        |
| Pareto capture (top 20% churners) | 25%           |

**Churn (dedicated high-value model, top-20%-revenue segment):**

| Metric                                      | Value          |
| ------------------------------------------- | -------------- |
| CV ROC-AUC                                  | 0.83           |
| Revenue capture                             | 71.44%         |
| High-value churners caught @ threshold 0.25 | 100%           |
| Intervention cost / revenue protected       | £360 / £73,506 |

**CLV (CatBoost, single-stage):**

| Metric                                 | Value     |
| -------------------------------------- | --------- |
| MAE                                    | £1,428.58 |
| RMSE                                   | £4,320.35 |
| R²                                     | 0.684     |
| Revenue capture (top ~19% predicted)   | 84.63%    |
| Pareto capture (top 20% past spenders) | 90.70%    |

**Revenue forecasting:**

| Scenario                        | Point forecast | 95% range             |
| ------------------------------- | -------------- | --------------------- |
| Conservative (outlier-adjusted) | £840,410       | £407,019 – £1,464,293 |
| Growth (last observation)       | £1,464,293     | —                     |

Coefficient of variation on monthly revenue: 30.3% (high volatility, explicitly reported rather than smoothed over).

**Retail EDA highlights:** $10.27M total revenue, 20,951 orders, $490 average order value, 67% repeat-customer rate, top 1% of customers driving a disproportionate share of revenue, and a product-return analysis showing return rate is driven by fulfillment/fit rather than price or price volatility.

## Tech stack

- **Data & EDA:** pandas, numpy, matplotlib
- **Statistics:** scipy (Mann-Whitney U, Spearman/rank-biserial correlation), Yeo-Johnson power transformation
- **Modeling:** scikit-learn (Logistic Regression, Random Forest, Pipeline/ColumnTransformer), XGBoost, CatBoost
- **Forecasting:** statsmodels (Holt exponential smoothing, moving-average baselines, ARIMA)
- **Evaluation:** custom business-metric functions — revenue capture, Pareto capture, Expected Revenue Saved, intervention cost efficiency, tier-level revenue-at-risk coverage

## Current status & next steps

All four notebooks are complete and each ends in a business-facing conclusion with concrete recommendations. Retail Analysis was delivered as a 33-page client report (PDF — see below). Nothing in this suite is deployed as a live service yet; it currently runs as a manual monthly analysis.

Real next steps, in order:

1. Wire churn scoring + CLV scoring into a single scheduled job (the architecture is already designed as a monthly workflow in the churn notebook — it just isn't automated yet)
2. Replace the behaviorally-inferred churn label with an explicit cancellation/opt-out event once the platform supports it — expected to close most of the 0.77 AUC ceiling
3. Collect 200+ confirmed high-value churn events so the dedicated high-value model stops being CV-fragile (currently trained on 34 churners)
4. Extend the revenue history past 24 months so the forecast's confidence interval becomes statistically meaningful instead of directional

---

### A note on the Retail Analysis report

The EDA/diagnostics notebook doesn't have a live dashboard — its deliverable is a 33-page PDF report written for a non-technical stakeholder. Rather than a "Try it live" section, link it directly, e.g.:

> **Read the full report:** [`reports/retail_analysis_report.pdf`](./reports/retail_analysis_report.pdf)

Drop the PDF in a `/reports` folder in the repo and point the link there — GitHub will render it inline when someone clicks through, so it functions the same way a demo link would for the deployed projects.
