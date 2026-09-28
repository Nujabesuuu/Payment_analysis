# Payment Transactions Analysis

**English** | [Українська](README.uk.md)

An end-to-end analysis of card payment data: 236,918 payment attempts from 40,210 users over two months (January–February 2026). The project covers data quality, approval rate, chargebacks, revenue, LTV, and calendar and cohort analysis. Every difference found is tested for statistical significance and sized in money, and the findings end in a prioritised list of recommendations.

**Stack:** Python 3.13 · DuckDB (SQL) · pandas · SciPy · Matplotlib · Jupyter

> The notebook and the chart labels are in Ukrainian. This README summarises the work in English.

---

## Contents

- [Overview](#overview)
- [Key metrics](#key-metrics)
- [Findings](#findings)
- [Recommendations](#recommendations)
- [Methodology](#methodology)
- [Limitations](#limitations)
- [Repository structure](#repository-structure)
- [Data](#data)
- [How to run](#how-to-run)

---

## Overview

The analysis answers three groups of questions:

1. **User-level SQL metrics.** For each user it finds total spend, the status of the last transaction, and whether the user ever paid successfully. Every metric is computed in at least two ways and the results are cross-checked. The notebook also shows how to answer all three questions in a single query with no subqueries or CTEs.
2. **Payment performance.** Approval rate, chargeback rate, revenue and LTV are broken down by card brand, country, platform, payment amount, user age and user activity.
3. **What drives the losses and what can be recovered.** This covers the structure of decline codes, calendar anomalies, cohort behaviour, and the upper bound of revenue that could be won back.

All aggregation is done in SQL on top of a single cleaned layer. pandas and SciPy are used only for statistical tests and charts.

## Key metrics

| Metric | Value |
|---|---|
| Payment attempts | 236,918 |
| Users | 40,210 |
| Approval rate, per attempt | 74.70% |
| Approval rate, per user-day | 83.96% |
| Chargeback rate (of successful transactions) | 0.544% |
| Net revenue | 2,325,365 |
| Net revenue per active user | 57.83 |
| LTV(30), mean / median | 26.62 / 5.0 |

The gap between approval per attempt and approval per user-day (9.3 pp) shows how much same-day retries recover: roughly one decline in four ends in a successful payment on the same day.

## Findings

### 1. Platform is the main driver of approval

Android approves 7.4 pp worse than desktop (71.34% vs 78.72%). Android is also the largest channel, with nearly half of all attempts. Lifting it to the desktop level would add about 4.2% to net revenue. The 95% confidence intervals of the three platforms do not overlap.

![Approval rate by platform](images/01_platform_approval.png)

The gap does not come from a single error. All four decline codes are elevated on Android at the same time (1.23–1.54× relative to desktop), which points to the audience or the end-to-end payment flow rather than a bug in one integration step.

![Decline codes by platform](images/02_errors_by_platform.png)

### 2. The cheapest tier approves worst

Large payments are usually declined more often. Here it is the opposite: the cheapest tier accounts for 55% of volume and approves at 70.47%, against 82.19% for the most expensive tiers. Meanwhile, 3.5% of attempts in the top tiers bring almost as much revenue as the 55% in the cheapest one.

![Approval rate by amount tier](images/03_amount_tiers.png)

### 3. Strong hour-of-day effect, no trend over time

Approval drops to 69.22% at 7 am and peaks at 76.76% in the afternoon, a 7.5 pp range that is wider than the gap between platforms. The dip falls on the hours with the fewest attempts, which is consistent with scheduled recurring charges hitting insufficient funds.

![Approval rate by hour of day](images/05_hour_of_day.png)

Daily approval shows no trend over the two months. Three episodes stand out, each with a different cause: a logging failure on January 4 (a spike in declines with no error code), a spike in one specific decline code on January 30 – February 1, and the best day of the period on February 24.

![Daily approval rate and anomalies](images/04_calendar.png)

### 4. Cohorts point to a roughly two-week trial period

Cohort activity does not peak in the first week after registration but in the second, and the first payment comes on day 11 on average. Unobserved cells are left blank rather than zero so that censoring is not mistaken for churn.

![Weekly cohort retention](images/06_cohort_retention.png)

Cumulative LTV grows almost linearly. The first full cohort earns about 16% more over 30 days than the next two, but only three cohorts have a complete 30-day window, so this is a signal to monitor rather than a confirmed trend.

![Cumulative LTV by cohort](images/07_cohort_ltv.png)

### 5. Card brand makes no difference

Visa and Mastercard show the same approval rate (74.76% vs 74.64%, p = 0.51), chargeback rate (p = 0.81) and 30-day LTV (Mann–Whitney, p = 0.75). The difference does not appear inside any country or platform either.

### 6. Recoverable revenue

About a quarter of declined amounts are recovered by users retrying on the same day. The rest sets an upper bound on what better retry logic could win back: 480,864, or 20.7% of net revenue. It is a ceiling, not a target, because part of the declines come from insufficient funds or blocked cards.

![Recoverable revenue by decline code](images/08_recoverable_revenue.png)

## Recommendations

Ordered by expected impact:

1. **Investigate why Android underperforms.** Up to 4.2% of net revenue is at stake. Start with differences in payment methods, the 3DS flow and traffic sources rather than looking for a single bug.
2. **Move scheduled charges from the morning to the evening.** The effect is modest, but testing it only means changing a schedule, not the product.
3. **Investigate the cheapest tier.** Up to 3.3% of net revenue is at stake. It most likely hides a separate scenario, such as trial or minimum charges, with lower-quality cards.
4. **Build retry logic around decline code e0.** It accounts for 33.3% of declines, recovers only 24.2% of the time, and is the last transaction for 37.2% of users who left after a decline.
5. **Keep both card brands.** There is no measurable difference between them.
6. **Fix decline-code logging.** 659 declines have no code, and a third of them happened on a single day.
7. **Find out what the channel with no platform recorded is.** Its 413 users show the best numbers in the dataset: 87.39% approval, LTV(30) of 51.42, and 93.4% paying users.

## Methodology

- **Data audit before any calculations.** The audit checks nulls, duplicate order IDs, chronology, whether chargebacks sit only on successful transactions, and the two different meanings of a missing error code.
- **One cleaned SQL layer.** `transactions_clean` changes only types and adds derived columns. No rows are dropped, and every exclusion is stated in the calculation that uses it.
- **Metrics defined up front.** Chargeback rate is measured against successful transactions, as the card schemes do. Revenue is reported gross and net of chargebacks. LTV is truncated to 7, 14 and 30 days and counts all users in a cohort, paying or not.
- **Selection bias in LTV.** A user appears in the data only after their first attempt, so late cohorts are dominated by fast payers. LTV is reported on a consistent base, and the naive base is shown alongside it.
- **Segment analysis.** A single parametrised SQL helper produces every breakdown, and every breakdown is reconciled back to the baseline totals.
- **Significance testing.** Two-proportion z-tests with Wilson intervals are cross-checked with χ². Skewed LTV is compared with the Mann–Whitney test. A Bonferroni check covers multiple testing. Effect sizes are expressed as a share of net revenue.
- **Decline structure.** Each error code is scored on how often it is recovered the same day and how often it is the user's last transaction.
- **Calendar anomalies.** A day is flagged when it deviates from the 7-day rolling median by more than 2.5 pp.

## Limitations

- The data covers only two months, so full LTV cannot be calculated. All cohort conclusions rely on truncated horizons.
- Users who registered but never attempted a payment are not in the data, so revenue per user is per active user.
- There is no issuing bank, acquirer, product type or description of the decline codes. The causes of declines are inferred from behaviour, and those conclusions are marked as hypotheses.
- No currency is given, so monetary effects are read as shares of revenue rather than absolute amounts.

## Repository structure

```
.
├── payments_analysis.ipynb        # full analysis with saved outputs
├── sql/
│   └── user_metrics_bigquery.md   # user-level queries in BigQuery syntax + dialect notes
├── data/
│   └── transactions.parquet       # dataset
├── images/                        # charts
├── requirements.txt
└── LICENSE
```

## Data

An anonymised dataset of card payment attempts, one row per attempt. IDs are surrogate keys, and amounts are in conventional units with no currency.

| Column | Description |
|---|---|
| `id_order` | Order ID |
| `id_user` | User ID |
| `status` | `success` or `fail` |
| `date_created` | Attempt timestamp |
| `amount` | Payment amount |
| `card_brand` | `VISA` or `MC` |
| `error` | Decline code (0–3), empty for successful attempts |
| `country` | User country |
| `platform` | `android`, `ios`, `desktop`, `other` |
| `date_reg` | User registration timestamp |
| `is_chargeback` | Whether the payment was charged back |

## How to run

```bash
python -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Open `payments_analysis.ipynb` in JupyterLab, VS Code or PyCharm and run all cells. The first cell builds a local `payments.duckdb` from the parquet file. A full run takes about a minute.
