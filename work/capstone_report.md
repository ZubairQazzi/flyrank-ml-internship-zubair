# Capstone Report — Content Refresh Opportunity Scoring

- **Author:** Zubair Tariq
- **Lane:** Content Refresh Opportunity Scoring
- **Repo:** https://github.com/ZubairQazzi/flyrank-ml-internship-zubair
- **Date:** August 2026

## 0. Abstract

This project asks whether historical search-performance signals can help prioritize which published content pages deserve human review for possible refresh work.

Using the FlyRank ML Internship dataset, I analyzed 85,967 eligible pages and evaluated models on a grouped client holdout of 4,247 pages.

I compared a transparent staleness-and-visibility baseline with Logistic Regression and Random Forest using five pre-decision search features.

Logistic Regression produced the strongest fresh result, reaching 55% precision@20 compared with 20% for the rule baseline, while overall ROC-AUC remained modest at 0.5537.

The final output is a human-review action queue designed to prioritize editorial investigation rather than automate content changes.

## 1. Problem framing

This project asks whether historical search-performance signals can help prioritize which content pages deserve human review for possible refresh work.

The unit of analysis is a published content page. The analysis produces model scores and a ranked queue that helps an editor decide which pages to review first.

The intended action is not automatic rewriting or publishing. A human reviewer uses the ranking to decide whether a page should be reviewed for freshness, checked for weak click capture, monitored, or held.

A wrong high-priority call can waste editorial time or encourage unnecessary changes to a page that does not need them. A wrong low-priority call can delay review of a page that may deserve attention.

Data and machine learning are useful here because the portfolio contains far more content-performance observations than a person can review manually. The models are used as decision support to prioritize attention, not as proof that refreshing a page will improve performance.

## 2. Data safety

This project uses pseudonymized content-performance data from the FlyRank ML Internship dataset. The warehouse release used for the project is build `20260703`.

The final analysis uses:

- daily content performance data from `fact_content_daily_performance`
- content metadata from `dim_content`

Historical search signals are measured from March 1–21, 2026. The later March 22–31, 2026 window is used only to define the evaluation outcome.

The model uses five measured features:

- `past_impressions`
- `past_clicks`
- `past_ctr`
- `past_avg_position`
- `gsc_observed_days`

Pseudonymous `client_hash_id` and `content_hash_id` values are used only for grouping, joining, evaluation, and tracing rows. They are not model features.

Label-derived fields such as `trend_direction` and `trend_pct` are deliberately excluded because they could leak information about the outcome into the model. Future-window information is also excluded from the model inputs.

The analysis excludes pages without sufficient observed search data. The final eligible population requires:

- at least 7 observed GSC days in the feature window
- at least 5 observed GSC days in the outcome window
- at least 100 historical impressions
- published and non-deleted content

No client names, page URLs, private search queries, credentials, or other identifying information are included in the report or public outputs.

## 3. Baseline

The baseline is a transparent rule-based ranking built before the machine-learning models.

It uses the same historical feature frame and the same held-out client split as the learned models so the comparison is fair.

The baseline prioritizes pages using simple observable signals based on existing visibility and page staleness. Its purpose is to provide a clear reference point: a learned model should only be considered useful if it improves ranking quality beyond this simpler rule.

The baseline and learned models are evaluated on the same held-out population and with the same ranking metrics. The final comparison also reports the positive base rate so top-K precision is interpreted in context.

The final baseline metrics are reproduced directly in the capstone notebook on the same held-out rows used for the learned models.

## 4. Model / analysis

The analysis compares Logistic Regression with a Random Forest classifier. Logistic Regression produced the strongest fresh benchmark result, while Random Forest is retained as the scoring model used by the Week 7 action playbook.

Both learned models use five historical search-performance features:

- `past_impressions`
- `past_clicks`
- `past_ctr`
- `past_avg_position`
- `gsc_observed_days`

These features are measured only from the historical feature window. Pseudonymous IDs, future-period information, label-derived trend fields, page URLs, client names, and private query data are not used as model features.

The target is a binary `future_decline` proxy. A page is labeled positive when its average daily impressions during the March 22–31, 2026 outcome window are less than 80% of its average daily impressions during the March 1–21, 2026 feature window.

Logistic Regression provides a simple and interpretable learned reference. Random Forest provides a non-linear comparison that can capture interactions between the measured search signals.

The Logistic Regression configuration uses standardized inputs, balanced class weights, `max_iter=1000`, and `random_state=42`.

The Random Forest configuration uses 300 trees, maximum depth 8, minimum leaf size 20, balanced class weights, and `random_state=42`.

Both models are used for ranking and decision support. Their scores are not interpreted as proof that a page should be refreshed or that a refresh will improve future performance.

## 5. Evaluation

The final evaluation uses a grouped client holdout so pages from the same client cannot appear in both training and test data.

The deterministic split contains:

- 81,720 training rows
- 4,247 held-out test rows
- 31 training clients
- 8 test clients
- 0 overlapping clients
- 41.96% positive base rate in the held-out test set

All three approaches are evaluated on exactly the same held-out rows.

| Method | Precision@10 | Precision@20 | ROC-AUC |
|---|---:|---:|---:|
| Rule baseline | 10% | 20% | 0.4760 |
| Logistic Regression | 40% | 55% | 0.5537 |
| Random Forest | 40% | 50% | 0.5529 |

Logistic Regression produced the strongest fresh result. It matched Random Forest at precision@10, exceeded it at precision@20, and had a slightly higher ROC-AUC.

Both learned models improved substantially over the transparent baseline at the top of the ranking. However, overall ROC-AUC remained close to 0.55, so the available five historical features provide only modest discrimination.

One recurring error pattern is that pages with weak historical engagement can receive high risk scores even when they do not decline later. The opposite also occurs: some pages with apparently healthy historical impressions and clicks still decline in the outcome window.

These errors support treating the learned ranking as a prioritization aid rather than an automatic decision system.

## 6. Interpretation

The learned models found some useful ranking signal, but no single historical feature explains future decline strongly.

The strongest practical result came from Logistic Regression, which reached 55% precision@20 on the held-out client set. Random Forest was close, but slightly weaker in the fresh capstone comparison.

Earlier permutation-importance analysis showed that `gsc_observed_days`, `past_clicks`, and `past_ctr` carried more useful signal than the other features. `past_impressions` contributed less, while `past_avg_position` did not clearly improve generalization in the held-out clients.

The error analysis also showed two important patterns.

First, pages with very weak historical engagement can receive high decline-risk scores even when their later performance remains stable. This means weak CTR or clicks should not automatically trigger a refresh.

Second, some pages with healthy-looking historical impressions and clicks still decline later. Static historical totals cannot capture every temporal, seasonal, or content-specific factor affecting future search performance.

The main finding is therefore directional rather than definitive: historical search-performance signals can improve prioritization over a simple rule baseline, but they are not strong enough to support automatic content decisions.

## 7. Recommendation

The model output should be used as a human-review queue, not as an automated instruction to modify content.

The final held-out queue contains 4,247 pages and assigns four actions:

- 276 pages to `REVIEW_FOR_REFRESH`
- 538 pages to `CHECK_CTR`
- 1,608 pages to `MONITOR`
- 1,825 pages to `HOLD`

The recommended workflow is:

1. Review `REVIEW_FOR_REFRESH` pages first because they combine higher measured decline risk with enough existing visibility to justify editorial attention.
2. For `CHECK_CTR`, inspect title, snippet, SERP context, and search-intent alignment before considering a broader content rewrite.
3. For `MONITOR`, wait for another measurement window unless other business evidence suggests urgent review.
4. For `HOLD`, take no immediate action unless new evidence appears.

Each recommendation carries a reason code so the reviewer can understand why the page entered the queue:

- `HIGH_RISK_VISIBLE`
- `LOW_CTR_VISIBLE`
- `STALE_VISIBLE`
- `MODERATE_RISK`
- `LOW_EVIDENCE`

Before changing content, a reviewer should check freshness, seasonality, search intent, content overlap, SERP context, and business value.

Confidence in the ranking is moderate rather than high. The learned models improve top-of-queue precision over the transparent rule baseline, but overall ROC-AUC remains close to 0.55.

The system should not automatically rewrite or publish content, delete pages, change redirects or canonicals, modify metadata, or make revenue or spending decisions from the model score alone.

## 8. Reproducibility

The complete analysis is available in the public repository:

https://github.com/ZubairQazzi/flyrank-ml-internship-zubair

The final capstone notebook is:

`work/notebooks/capstone.ipynb`

The notebook rebuilds the analysis from the FlyRank warehouse, recreates the grouped client holdout, trains the baseline and learned models, evaluates them on the same held-out rows, regenerates the ranked recommendation queue, and writes the paper artifacts.

The main reproducibility settings are:

- random seed: `42`
- feature window: March 1–21, 2026
- outcome window: March 22–31, 2026
- grouped test size: 20% of clients
- training clients: 31
- held-out clients: 8
- client overlap: 0
- held-out rows: 4,247

The principal Python dependencies used in the notebook are:

- `duckdb`
- `pandas`
- `numpy`
- `scikit-learn`
- `matplotlib`

To inspect the project from a fresh clone:

```bash
git clone https://github.com/ZubairQazzi/flyrank-ml-internship-zubair.git
cd flyrank-ml-internship-zubair
