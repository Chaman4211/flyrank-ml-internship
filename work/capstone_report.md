# Content Refresh Prioritization: A Leakage-Aware Research Note

**Author:** Chaman Afzaal  
**Track:** FlyRank ML Internship — Applied Search Intelligence  
**Updated:** 29 October 2026  
**Status:** Public-safe research note; model benchmark uses the anonymized starter slice.

> **Scope note.** The warehouse profile below comes from the executed full-release exploration notebook. The model-versus-baseline numbers come from a separate 30,000-row starter-slice benchmark. They must not be described as a model trained on all 78.8 million daily rows.

## 1. Title and abstract

**Research question:** Can a learned ranking help people prioritize content items for review better than a transparent freshness-and-visibility rule?

This study examines content-review prioritization using FlyRank's anonymized search-performance resources. The warehouse release contains 78,835,655 daily performance rows from 27 January 2025 through 30 June 2026, while the headline model comparison uses the 30,000-row starter slice. On a client-held-out test set of 7,115 starter-slice rows, Logistic Regression achieved average precision 0.529, Precision@20 0.600, and Precision@50 0.640; the Week-4 rule baseline scored 0.504, 0.450, and 0.440 on the same rows. This is a measured improvement in ranking against an observed decline label on that split, with a 0.517 test-label base rate and eight held-out clients. The results support further human-reviewed testing, not a claim that refreshing content causes recovery or that the starter-slice metrics generalize to the full warehouse.

## 2. Introduction / problem statement

A team may have thousands of content items but limited time to inspect them. The practical question is not “which page should the system automatically rewrite?” It is “which page should a person inspect first, and why?”

The benchmark compares a readable rule—old enough and still visible in search—with Logistic Regression that uses content and keyword-context fields. The desired output is a review priority, not an automatic refresh command. A useful ranking should place more items carrying the observed decline label near the top, while leaving final decisions to a human.

## 3. Data

### Warehouse release profile

The executed full-release exploration notebook reports release **build v20260703** and the following tables:

| Table | Rows | Grain |
|---|---:|---|
| `dim_clients` | 104 | One pseudonymized client |
| `dim_content` | 519,606 | One content item |
| `fact_content_daily_performance` | 78,835,655 | Report date × client × content |
| `fact_content_daily_performance_sample` | 11,694,072 | Same grain; final month only |
| `fact_content_query_90d` | 2,414,248 | Client × content × query hash |

The daily table covers **2025-01-27 to 2026-06-30**. The exploration query derived 155,903 content items with sufficient history under its initial history filter. The warehouse profile and query are documented in the [full-release exploration notebook](../notebooks/03_working_with_the_full_release.ipynb).

### Headline benchmark dataset

The headline model-versus-rule comparison uses the repository's anonymized starter slice: **30,000 content items, 44 source columns, and 32 pseudonymized clients**. Its metrics are aggregated over a trailing 90-day window ending at export time. This is a different analysis layer from the full-release warehouse profile.

### Exclusions and privacy

The public report does not publish client names, domains, page URLs, raw search queries, query hashes, row-level outputs, credentials, or data exports. IDs are used only for grouping, joins, and reproducibility. The final-month warehouse sample was not used to design the label. The starter benchmark excludes target-derived fields, direct trend-window inputs, IDs, current traffic totals, and product-generated flags from model features.

## 4. Methodology

### Target and assumptions

For the starter benchmark, the binary target is `is_declining_label = (trend_direction == "down")`. It describes an **observed trend** in the snapshot. It is not a verified future-period forecast and is not a label for successful recovery after a content refresh.

### Features

The Logistic Regression model uses `content_age_days`, `days_since_last_update`, `word_count`, `char_count`, `search_volume`, `competition`, `cpc`, `content_type`, `main_intent`, and `competition_level`. Numeric missing values are median-imputed with missingness indicators and scaled; categorical missing values are labeled “unknown” and one-hot encoded. Preprocessing is fit on training data only.

The model excludes `trend_direction`, `trend_pct`, the target, identifiers, current traffic totals, 30-day trend inputs, and product flags such as `health_score`, `needs_ctr_fix`, and `is_quick_win`. These exclusions avoid direct target leakage, identity memorization, and circular use of existing product decisions.

### Baseline

The Week-4 rule selects pages that have not been updated for at least **180 days** and have at least **500 impressions in the trailing 90 days**, ranking selected pages by impressions. The rule is transparent and useful for comparison, but it uses different information from the model; the comparison is an operational benchmark rather than an equal-information experiment.

### Validation design

A fixed `GroupShuffleSplit` uses `test_size=0.25` and random seed **42**, grouping by `client_id`. The saved run used 22,885 training rows across 24 clients and 7,115 test rows across 8 clients, with zero client overlap. The observed decline-label rate was 0.550 in training and 0.517 in testing. The baseline and model are evaluated on the same test rows.

### Leakage audit

The saved Week-6 audit reports PASS for target exclusion, trend-field exclusion, identifier exclusion, direct 30-day input exclusion, current traffic/product-flag exclusion, and zero client overlap. It also notes that the snapshot cannot prove every remaining field was captured strictly before a future outcome window. Separately, the first full-warehouse prototype used position volatility from the last 30 days and fixed-window query signals that can overlap its outcome period. Those prototype metrics are **not** used as headline evidence here.

## 5. Results (model vs baseline)

| Method | Test base rate | Average precision | Precision@20 | Precision@50 |
|---|---:|---:|---:|---:|
| Week-4 rule baseline | 0.517 | 0.504 | 0.450 | 0.440 |
| Logistic Regression | 0.517 | 0.529 | 0.600 | 0.640 |

The model's average precision was **0.025 higher** on this split. Precision@20 was 0.150 higher and Precision@50 was 0.200 higher. These are measurements from one client-held-out split, not a guarantee of performance for future dates or unseen client populations. The test-label base rate (0.517) is included so the ranking scores are read in context.

The earlier random-row validation run reported average precision 0.6447 and Precision@50 0.70; the client-grouped rerun reported 0.5315 and 0.64. The grouped result is the more relevant result for generalization across clients, because rows from a client do not appear in both train and test.

### What the signal audit showed

In the starter snapshot, the observed decline-label rate was 51.14% for pages updated 0–30 days ago (n=20,480), 61.11% for the 91–180-day group (n=9,171), and 47.13% for the 181+ day group (n=174). The oldest bucket is small, and the pattern is not monotonic. By impression bucket, the rates were 38.90% (0–99; n=7,994), 60.44% (100–499; n=5,280), 62.06% (500–2,999; n=8,443), and 57.00% (3,000+; n=8,283). This does not support a simple claim that older or more visible pages reliably decline.

## 6. Limitations and honest framing

- The main benchmark uses the 30,000-row starter snapshot; the report does **not** claim that this Logistic Regression model was trained on all 78.8 million warehouse rows.
- The label is an observed trend proxy, not a future forecast and not a refresh-success outcome.
- Client grouping reduces client leakage, but this is not a time-forward validation. Seasonal effects, measurement changes, and future distribution shift remain possible.
- The rule and model use different information, so this is an operational comparison rather than a controlled equal-feature experiment.
- The oldest staleness bucket in the signal audit contains only 174 rows. Its rate should not be treated as a stable population estimate.
- A static snapshot cannot establish that updating a page causes performance to improve. The people who choose pages to refresh may select them for reasons not present in the data.
- Warehouse coverage differs by client. The release notes say only four clients have 12+ months of GSC history, and GA4 availability varies. Missing or zero-filled values need to be interpreted using the data-availability flags.
- The initial warehouse model prototype had outcome-window overlap in some features. Its classification report is not presented as validated evidence.

**Allowed conclusion:** On this held-out starter-slice test, Logistic Regression ranked the observed decline label better than the simple Week-4 rule at the top 20 and top 50. This is directional decision-support evidence that merits further time-aware testing; it does not establish future decline or causal impact.

## 7. Ranked recommendations: the action playbook

The queue is a human-review list, not an automatic content-refresh order.

1. **Stale and visible — review first.** At least 180 days since update and at least 500 impressions in the trailing 90 days. Check intent match, accuracy, freshness, and competing pages before deciding whether to edit.
2. **Visible but recently updated — inspect the search result.** At least 500 impressions but less than 180 days since update. Review title/snippet alignment, search intent, and possible cannibalization before making changes.
3. **Stale with low visibility — diagnose before investing.** At least 180 days since update and fewer than 500 impressions. Check whether the page still has a useful role and whether low visibility has a known explanation.
4. **Missing or invalid signal — verify the data.** Do not assign an action until freshness and impression inputs are checked.
5. **Routine monitoring — no refresh based on the queue alone.** Keep observing and require human judgment.

Reason codes in the playbook are `stale_visible_review`, `visible_recent_review`, `stale_low_visibility_check`, `missing_signal_manual_check`, and `routine_monitor`. The queue uses only freshness and impression signals; it does not use the decline label, trend fields, or model predictions to rank pages.

## 8. Reproducibility

- **Repository:** [Chaman4211/flyrank-ml-internship](https://github.com/Chaman4211/flyrank-ml-internship)
- [Full-release DuckDB exploration](../notebooks/03_working_with_the_full_release.ipynb)
- [Week 4 baseline and signal audit](../work/notebooks/w04_baseline_score.ipynb)
- [Week 5 model comparison](../work/notebooks/w05_model.ipynb)
- [Week 6 validation and leakage audit](../work/notebooks/w06_validation_audit.ipynb)
- [Week 7 action playbook](../work/notebooks/w07_action_playbook.ipynb)
- [Capstone notebook](../work/notebooks/capstone.ipynb)

To reproduce the starter-slice benchmark, open the capstone notebook in Colab and run all cells. It uses seed 42 and a client-grouped 75/25 split. To repeat the full-release warehouse profile, follow the setup instructions in [SETUP.md](../SETUP.md), request access to the gated Hugging Face release, use a read-only token through Colab Secrets as `HF_TOKEN`, and run the full-release notebook. Never commit the token or warehouse extracts. The full 79M-row scan is expensive; use the saved exploration and avoid unnecessary repeated scans.

## 9. Acknowledgments and data credit

This work was completed as part of the FlyRank ML Internship and uses the anonymized FlyRank internship data resources. Data provenance and coverage are described in the repository's data dictionary and the full-release notebook.

**Built on the [FlyRank ML Internship dataset](https://flyrank.ai).**

## Research-page design notes

This page uses a paper-like reading order, a short abstract that includes the question, method, result, and use, tables for exact comparisons, and a small number of charts rather than decorative graphics. It follows general manuscript guidance that an abstract should explain the objective, method, key findings, and implications, and the official GitHub Pages pattern of serving a self-contained entry page from `docs/`.
