# Capstone Report — <your lane>

- **Author:** Mariam Arif
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/ma5029blp-wq/ML-flyrank-internship
- **Date:** 4 September 2026

## 0. Abstract

This project asks whether a simple Logistic Regression model can help identify content items showing observed search-performance decline and improve content-refresh prioritization compared with a transparent baseline. The analysis uses 30,000 anonymized content-level observations from the FlyRank content-refresh dataset, with historical search-performance, freshness, content, and intent features. A Logistic Regression model was evaluated using a grouped client-level test split, with Precision@20 and Precision@50 used to measure ranking quality. On the selected test split, the model achieved Precision@20 of 0.50 and Precision@50 of 0.62, compared with 0.30 and 0.32 for the baseline. The resulting ranked queue is intended as human-reviewed decision support for prioritizing which content pages should be examined first, not as an automatic content-editing or publishing system.

## 1. Problem framing

The goal of this project is to help SEO and content teams decide which content pages should be reviewed first when there are limited resources for content maintenance.
The unit of analysis is one content item. The model produces a decline-risk score and a ranked review queue, with higher-scored items placed earlier in the queue.
The output supports a human decision: which pages should be reviewed first for a possible refresh or other appropriate content action. The model does not automatically rewrite, publish, delete, or redirect content.
A wrong decision has a practical cost. Reviewing a page that does not need attention uses limited editorial time, while failing to review a genuinely declining and valuable page can delay a useful content opportunity. Ranking helps focus review effort on a smaller set of higher-priority pages.
Machine learning is useful here because it combines multiple historical performance and content signals into one ranking instead of relying on a single rule such as content age or search visibility.

## 2. Data safety

The analysis uses the anonymized FlyRank content-refresh dataset used during the internship. The modeling frame contains 30,000 content-level observations.
The model features are:
- `impressions_prev_30d`
- `clicks_prev_30d`
- `sessions_prev_30d`
- `days_since_last_update`
- `content_age_days`
- `avg_position`
- `search_volume`
- `competition`
- `word_count`
- `char_count`
- `content_type`
- `main_intent`
The target is derived from `trend_direction`, where `down` is treated as observed decline. Label-derived fields such as `trend_direction`, `trend_pct`, `is_declining_label`, and `declining_observed` are not used as model features.
Decision-derived fields such as `action`, `reason_code`, `score`, and `baseline_score` are also excluded from model training.
Pseudonymous IDs such as `client_id` and content identifiers are used for grouping or joining only. They are not predictive features.

## 3. Baseline

The baseline is a simple freshness-and-visibility rule used to prioritize content for review. A content item receives a positive baseline score when it has previous-30-day impressions and has not been updated for at least 91 days.
The baseline is intentionally transparent and easy to reproduce. It provides a simple comparison for the Logistic Regression model because both approaches produce a ranking of content items for review and are evaluated using the same test set and Precision@K metrics.
On the grouped client-level test split, the baseline achieved Precision@20 of 0.30 and Precision@50 of 0.32. The observed decline base rate on the same test set was 51.65%.
The baseline is a useful reference point rather than a claim that freshness alone explains content decline.

## 4. Model / analysis

The analysis uses Logistic Regression as a binary classification model. It was selected because the task has a binary observed outcome and the model provides a relatively interpretable ranking signal.
The model uses 12 features.
The numeric features are:

- `impressions_prev_30d`
- `clicks_prev_30d`
- `sessions_prev_30d`
- `days_since_last_update`
- `content_age_days`
- `avg_position`
- `search_volume`
- `competition`
- `word_count`
- `char_count`
The categorical features are:
- `content_type`
- `main_intent`
Numeric missing values are median-imputed and standardized. Categorical missing values are filled with the most frequent category and then one-hot encoded.
The target is `declining_observed`, defined as 1 when `trend_direction` is `down` and 0 otherwise.
The model deliberately excludes label-derived fields such as `trend_direction`, `trend_pct`, `is_declining_label`, and `declining_observed`, as well as decision-derived fields such as `action`, `reason_code`, `score`, and `baseline_score`.
Client IDs are used to create the grouped validation split and are not used as predictive features.
The model produces a probability score representing the estimated likelihood of the observed decline label. Content items are ranked from highest to lowest score for review prioritization.

## 5. Evaluation

The model was evaluated using a grouped client-level split. `GroupShuffleSplit` was used with `client_id` as the grouping variable, a 25% test size, and random state 42. This produced 22,885 training observations from 24 clients and 7,115 test observations from 8 clients, with no client overlap between the two sets.
The observed decline base rate in the test set was 51.65%. This base rate is reported alongside Precision@K so that the ranking results are not interpreted without context.
The model and baseline were evaluated on the same test split:

| Method | Precision@20 | Precision@50 |
|---|---:|---:|
| Baseline | 0.30 | 0.32 |
| Logistic Regression | 0.50 | 0.62 |

The Logistic Regression model therefore showed higher measured Precision@20 and Precision@50 than the baseline on this test split.
A short error analysis from the validation work found both false positives and false negatives. This means the model's ranking is useful for prioritization but is not a perfect classifier. The errors also reinforce the need for human review before taking content actions.
These results are measured on the selected grouped test split. They should not be interpreted as guaranteed performance on future clients or future time periods.

## 6. Interpretation

The model combines historical search performance, freshness, content characteristics, and search intent into a single ranking signal.
The strongest measured result is the improvement in ranking precision over the simple baseline. Precision@20 increased from 0.30 for the baseline to 0.50 for Logistic Regression, while Precision@50 increased from 0.32 to 0.62. This suggests that the model can provide a more useful prioritization signal than the baseline on the tested clients.
The model should not be interpreted as identifying a single cause of content decline. Different content characteristics can be associated with the observed label, and the model combines these signals rather than providing a causal explanation.
The earlier signal audit also showed that staleness alone was a mixed signal. This is why the action playbook combines the model ranking with other context, such as previous search visibility and freshness, rather than treating page age as a standalone decision rule.
A useful interpretation of the output is therefore: the model helps decide which pages deserve attention first, while the reason code and underlying content signals help a human reviewer understand the context.
The results are directional and observational. They do not show that any individual feature causes decline or that refreshing a page will improve its future performance.

## 7. Recommendation

The main recommendation is to use the model output as a ranked review queue for content maintenance. Higher-scored content items should be reviewed earlier because they received a higher estimated probability of the observed decline label.
An editor can use the queue in the following way:

1. Start with the highest-ranked content items.
2. Review the underlying search-performance and content signals.
3. For items marked `VISIBLE_STALE`, pay particular attention to whether the page has meaningful previous search visibility and has not been updated for at least 91 days.
4. For items marked `HIGH_MODEL_SCORE`, review the page context, search intent, content quality, and business value before deciding on an action.
5. Select the appropriate human-reviewed action, such as refreshing, monitoring, or taking no action.

The exported queue uses `REVIEW_REFRESH` as the recommended review action, but this is not an instruction to automatically refresh every page. The final decision remains with the content or SEO reviewer.
The model provides useful decision-support evidence because it achieved measured Precision@20 of 0.50 and Precision@50 of 0.62 on the grouped test split, compared with 0.30 and 0.32 for the baseline. However, these results come from one grouped test split and should not be treated as guaranteed performance on future data.
The queue should therefore be treated as a prioritization tool rather than an automatic content-management system. A high score indicates that an item deserves earlier review; it does not prove that the page will decline in the future or that refreshing it will improve performance.

## 8. Reproducibility

The analysis can be reproduced from the project repository using the capstone notebook and the anonymized starter dataset.
The main notebook is:
`work/notebooks/capstone.ipynb`
The modeling data is read from:
`data/raw/content_refresh_anonymized.csv`
The grouped train/test split uses `GroupShuffleSplit` with:
- `test_size=0.25`
- `random_state=42`
- grouping variable: `client_id`
The model uses the preprocessing and Logistic Regression pipeline defined in the capstone notebook. Numeric features are median-imputed and standardized, while categorical features are most-frequent imputed and one-hot encoded.
A fresh run should execute the capstone notebook from top to bottom and recreate the reported model, evaluation table, ranked recommendations, and exported figures.
The main reproducibility outputs are:
- `work/outputs/ml11_results_table.csv`
- `work/figures/ml11_model_vs_baseline.png`
- `work/figures/ml11_score_distribution.png`
The evaluation is a grouped client-level validation rather than a sealed future-time holdout. Therefore, the reported metrics should be treated as measured results for the selected test split, not as guaranteed future production performance.
The project uses a fixed random seed of 42 for the grouped split and Logistic Regression model. The notebook contains the preprocessing, feature definitions, training procedure, evaluation metrics, and export steps needed to reproduce the analysis.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. See [FlyRank](https://flyrank.ai).
This project was completed as part of the FlyRank ML Internship and uses the anonymized dataset provided for the internship work.
All findings in this report are presented as observed, measured, directional, and decision-support evidence. The reported Precision@20 and Precision@50 values are presented alongside the test-set base rate and baseline results.
The analysis does not make causal claims, does not make claims about Google search algorithms, and does not include client-identifying details or private search queries.
The results should be interpreted as evidence from the tested dataset and validation split rather than as a guarantee of future traffic, rankings, or business outcomes.

---rt
> match a fresh re-run.
