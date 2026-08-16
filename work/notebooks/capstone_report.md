# Capstone Report

- **Author:** 
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** 
- **Date:** 

## 0. Abstract

This project asks whether historical search and engagement signals can rank content pages that are more likely to experience click improvement in the following month. Using the FlyRank internship warehouse, I constructed a client × content dataset from March 2026 historical signals and April 2026 click outcomes. I compared a transparent Week 4 baseline with Logistic Regression, Decision Tree, and Random Forest models using a client-grouped train/test split. On the held-out test set, the Random Forest achieved 41.93% Precision@10% and 2.00× lift, compared with 38.40% and 1.83× for the baseline. The resulting ranked queue is intended as decision support for prioritizing content pages for human review, not as a causal prediction of the effect of refreshing a page.

## 1. Problem framing

The decision supported by this project is which content pages an editor should review first for potential improvement.

The unit of analysis is **client × content**.

The output is a ranked opportunity score for each eligible content page, with higher scores indicating higher priority for review.

A human editor can use the ranked queue to prioritize pages for further investigation, such as reviewing click capture, search-result alignment, and content relevance.

The cost of a wrong call is spending limited editorial time on a page that does not subsequently show click improvement. For that reason, the system is designed as decision support rather than an automatic refresh decision.

Data and ML are useful because the dataset contains multiple historical search and engagement signals that can be combined into a ranking rather than relying on a single transparent rule.

## 2. Data safety

The analysis uses the FlyRank internship warehouse release and focuses on March 2026 historical features with April 2026 outcomes.

The modeling population contains **176,737 client × content rows** with usable March `avg_position`.

The five predictive features are:

- `avg_impressions`
- `avg_clicks`
- `avg_position`
- `avg_pageviews`
- `avg_sessions`

The target is defined as:

> 1 when April average clicks are greater than March average clicks; otherwise 0.

Rows without usable March `avg_position` were excluded because the Week 4 baseline depends on this signal. Missing-position rows represented 46.68% of the initial March-April matched population and had a substantially lower positive rate than rows with usable position, so they were not blindly imputed.

The final modeling population has an **80.43% negative / 19.57% positive** distribution.

Pseudonymous client and content IDs are used for grouping, joining, and identifying rows within the analysis only. They are not predictive features.

Label-derived fields such as `trend_direction` and `trend_pct` were excluded. Future April outcome information was not used as a feature.

No client names, domains, URLs, private queries, credentials, or other client-identifying information are included in the public analysis.

## 3. Baseline

The transparent baseline is the Week 4 score:

> `baseline_score = avg_impressions × avg_position`

The baseline is a fair comparison because it is a simple, transparent ranking rule built from the same historical feature window and evaluated on exactly the same held-out test population as the ML models.

On the grouped test set, the baseline achieved:

- **Precision@10%: 38.40%**
- **Recall@10%: 18.31%**
- **Lift@10%: 1.83×**

The test-set positive base rate was **20.97%**, so the baseline's top 10% ranking captured a substantially higher concentration of positive outcomes than the overall test population.

## 4. Model / analysis

Three supervised models were evaluated:

1. Logistic Regression
2. Decision Tree
3. Random Forest

All models used the same five historical features:

- `avg_impressions`
- `avg_clicks`
- `avg_position`
- `avg_pageviews`
- `avg_sessions`

The target was April average clicks being greater than March average clicks.

Class-balanced training was used because the positive class was a minority of the modeling population.

The Random Forest was selected as the final ranking model because it produced the strongest held-out ranking performance.

Its test-set results were:

- **Accuracy:** 67.94%
- **Precision:** 36.03%
- **Recall:** 68.22%
- **F1:** 47.16%
- **ROC-AUC:** 0.7353
- **PR-AUC:** 0.3713
- **Precision@10%:** 41.93%
- **Lift@10%:** 2.00×

The model is used as a ranking system rather than as a claim that a page will definitely improve.

## 5. Evaluation

The final evaluation used a **client-grouped train/test split** so that clients in the training data could not also appear in the test data.

The split contained:

- **Training:** 103,085 rows across 32 clients
- **Testing:** 73,652 rows across 15 clients
- **Client overlap:** 0

The positive rate was:

- Training: **18.57%**
- Testing: **20.97%**

The main evaluation metric is Precision@10% because the practical decision is to prioritize a limited top portion of pages for editorial review.

| Ranking method | Precision@10% | Recall@10% | Lift@10% |
|---|---:|---:|---:|
| Random Forest | **41.93%** | 19.99% | **2.00×** |
| Decision Tree | 39.93% | 19.04% | 1.90× |
| W04 Baseline | 38.40% | 18.31% | 1.83× |
| Logistic Regression | 34.76% | 16.57% | 1.66× |

The Random Forest improved Precision@10% over the transparent baseline by **3.53 percentage points**, from 38.40% to 41.93%.

The result is an observed improvement on this held-out test population. It does not establish that the model will outperform the baseline in every future deployment.

Error analysis showed that some highly ranked pages did not subsequently improve. These false positives often had measurable search visibility and very low historical click activity. Successful high-ranked pages had similar historical characteristics, showing that the available features do not perfectly separate future outcomes.

## 6. Interpretation

The Random Forest placed the greatest feature importance on historical search visibility and click activity:

| Feature | Importance |
|---|---:|
| `avg_impressions` | 59.17% |
| `avg_clicks` | 20.50% |
| `avg_position` | 7.52% |
| `avg_pageviews` | 6.42% |
| `avg_sessions` | 6.39% |

The fitted model therefore relied most heavily on historical impressions, followed by historical clicks.

This should be interpreted as a property of the fitted model, not as evidence of causality.

A useful pattern in the ranked queue is that many high-scoring pages have measurable search visibility combined with very low historical click activity. These pages can be treated as candidates for review because the model ranks them highly, but the ranking alone does not establish that changing the page will cause clicks to increase.

The comparison also produced a negative result for Logistic Regression: despite being simpler, it produced lower top-10% ranking performance than both the baseline and the tree-based models.

## 7. Recommendation

The recommended workflow is:

1. **Prioritize the highest-ranked pages from the Random Forest queue.**
2. **Review click capture and content/search-result alignment** for pages with measurable visibility and low historical click activity.
3. **Use the score to prioritize human investigation, not to automatically refresh content.**
4. **Compare editorial decisions and later outcomes against the ranked queue** before treating the system as a production decision rule.

The Random Forest is the preferred ranking model for this experiment because its **41.93% Precision@10% and 2.00× lift** exceeded the W04 baseline's **38.40% and 1.83×** on the same grouped test set.

Confidence should remain moderate. The improvement is measurable but modest, the target is only a one-month click-change proxy, and the error analysis shows substantial overlap between successful and unsuccessful high-ranked pages.

## 8. Reproducibility

The complete analysis is implemented in:

`work/notebooks/capstone.ipynb`

The notebook loads the FlyRank warehouse data, constructs the March-April dataset, applies the documented population rule, creates the five features and target, builds the Week 4 baseline, trains the three ML models, performs client-grouped validation, evaluates the rankings, generates recommendations, and saves paper artifacts.

The main saved artifacts are:

- `work/outputs/capstone/ranking_results.csv`
- `work/outputs/capstone/classification_results.csv`
- `work/outputs/capstone/feature_importance.csv`
- `work/outputs/capstone/top_recommendations.csv`
- `work/outputs/capstone/precision_at_10_comparison.png`
- `work/outputs/capstone/random_forest_feature_importance.png`

The modeling random seed is **42**.

The notebook was run successfully from top to bottom without errors.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

[FlyRank](https://flyrank.ai)
