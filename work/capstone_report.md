# Capstone Report — <your lane>

- **Author:** Ritashree Mukherjee
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** Project-1-Applied-Search-Intelligence-Google-Search-Ranking-Discoverability
- **Date:** 25th October, 2026
  

## 1. Problem framing

**Decision supported.** Which pages a content/SEO reviewer opens first. FlyRank clients have far more indexed pages than review hours, so the real question is "which handful is worth the next hour?", not "is this page bad?".

**Unit of analysis.** One row = one page (client_hash_id, content_hash_id) at one as-of date. Features look back from the as-of date. The label looks forward 30 days.

**Output.** A ranked review queue: priority score, suggested action (refresh / expand / protect / prune / monitor), and reason codes a human can sanity-check. A person stays in the loop for every page.

**Cost of a wrong call.**

False positive: a reviewer spends 15–30 minutes on a page that was fine. This is annoying and burns scarce reviewer time, but it is not dangerous.
False negative: a genuinely declining, high-demand page never surfaces and keeps losing clicks unnoticed.
The queue is capacity-limited (a reviewer works the top 20–100), so precision at the top of the list matters more than overall accuracy. AUC is reported as the honest discrimination number.

## 2. Data safety

**Data used.** FlyRank warehouse release: dim_content (519,606 rows, one per client × content item), dim_clients, and fact_content_daily_performance_sample (11.7M rows, 2026-06-01 to 2026-06-30). The full daily fact table (78.8M rows, 70 clients, 2025-01-27 to 2026-06-30) was profiled for the data contract but not used for the modelling features.

**Feature table.** features.parquet: 156,778 rows × 50 columns, three as-of dates (2026-06-01, 06-15, 06-29).

**Deliberately excluded (and why).**

Field	Why excluded

trend_direction, trend_pct (starter CSV)	Label-derived, same-window. A model recovers the rule that made the label.
clicks_future30, y_rate, is_declining	Label or future window. Used only to build the target.
client_hash_id, content_hash_id, as_of_date	IDs and dates are for grouping and filtering only, never features.
provider_used, model_used	Record decisions the content-generation system already made (post-decision information).
last_optimized_date, content_updated_date, optimization_eligible_date, is_published, is_deleted	Current-state snapshots. Historical availability is not reconstructed.

Leakage tests run (W03, W06).

Future-window test: adding clicks_future30 on purpose pushed AUC to 0.999 (sampled run) and to 1.000 in the W06 harness. This confirms the harness detects leakage.
Split honesty: random vs client-grouped CV (Section 5).
Date-proxy check: see below. This was the most important finding.

Date-proxy artifact (negative result). The first clean-looking model scored AUC about 0.92. Permutation importance showed one feature, active_days_90d, carrying nearly all of it (about 0.36 AUC drop, versus less than 0.01 for the next features). That column mostly encodes which as-of date a row comes from, and the label base rate depends on the date:

| as_of_date | rows | label base rate |
| --- | ---: | ---: |
| 2026-06-01 | 17,992 | 0.065 |
| 2026-06-15 | 62,550 | 0.585 |
| 2026-06-29 | 76,236 | 0.990 |

The sample starts on 06-01, so the 06-01 row has almost no usable prior window and the 06-29 row has only one future day. active_days_90d has pooled AUC 0.881 but only 0.50–0.54 within a single date. Fix: model one as-of date (2026-06-15) with a rate-based label.

## 3. Baseline

**Rule (W04).** Score = number of flags fired:

| Flag | Condition |
| --- | --- |
| low visibility | impressions_last30 < 500 |
| impression decline | impressions_last30 / impressions_prev30 < 0.80 |
| stale | content_age_days_at_as_of ≥ 180 |

Ties are broken by lowest impressions. Reason codes: reason_low_visibility, reason_impression_decline, reason_stale.

Honest caveats about this baseline.

impressions_prev30 is 100% missing at 2026-06-15 because the sample starts 06-01. Flag 2 therefore never fires, and the rule is effectively two flags.
The W04 top-50 are all score 2 with 1 impression and 1 click, so the queue is driven by ties. Several pages repeat across dates. The "confidence: High" note in the W04 top-20 review is boilerplate and should not be cited.
A fair second baseline is the single feature -impressions_last30. It is reported next to the rule in Section 5.

## 4. Model / analysis

**Task type.** Ranking/scoring. A binary classifier is trained and its predicted probability is the priority score. It is judged by AUC and precision@K, not accuracy.

**Models.** Random Forest (300 trees, min_samples_leaf=20, constant fill −1) and Logistic Regression (median impute + indicators, standard scaling). Seed = 0.

Target (one sentence). y_rate = 1 if average daily clicks over the 15 days after 2026-06-15 are below 0.70 × average daily clicks over the 15 days before it, restricted to pages with clicks_last30 ≥ 10 (n = 10,436).

Why the filter. Rate-based labels are mostly noise on pages with very few clicks.

Label sensitivity. Base rate is 0.386 at ratio 0.6, 0.490 at 0.7, and 0.578 at 0.8, so the label depends on the threshold.

Features (about 43–45).

Search: impressions_90d, impressions_last30, clicks_last30, ctr_last30, avg_position_last30, ga4_sessions_last30, active_days_90d
Content metadata: search_volume, competition, cpc, backlinks, category_count, keyword_char_count, keyword_token_count, url_char_count, content_age_days_at_as_of
One-hot: content_type, competition_level, main_intent (with explicit UNKNOWN categories)
has_* missingness flags

Left out on purpose. Everything in the Section 2 table, plus all-NaN columns (impressions_prev30, clicks_prev30).

Known redundancy. At 2026-06-15, impressions_90d == impressions_last30 for more than 99% of rows (the sample only reaches back to 06-01). Several has_* flags duplicate each other. The effective feature set is a roughly 15-day window.

## 5. Evaluation

Split. One as-of date (2026-06-15) with 5-fold GroupKFold by client_hash_id. Pages from the same client share traffic patterns, so a random split lets the model learn the client.

Why not time-aware. Only three as-of dates exist: 06-01 has no prior window and 06-29 has no future window. A time-aware split with a 30-day embargo needs the full daily table (Section 8).

Random vs grouped (W06, Random Forest).

Split	Model AUC (mean ± sd)	Impressions-rule AUC	Model top-5% precision	Base rate
Random 5-fold	0.700 ± 0.014	0.583	0.890	0.529
Grouped by client	0.588 ± 0.036	0.584	0.681	0.529

The grouped split removes most of the apparent lift. Under it the forest is about equal to a one-feature rule.

Models vs baselines, same rows and folds (W05, pooled out-of-fold, base rate 0.49).

| Score | AUC | P@20 | P@50 | P@100 |
| --- | ---: | ---: | ---: | ---: |
| Random (base rate) | 0.500 | 0.49 | 0.49 | 0.49 |
| Baseline rule (tie-break) | 0.570 | 0.65 | 0.70 | 0.64 |
| Baseline rule (no tie-break) | 0.521 | 0.70 | 0.70 | 0.56 |
| Single feature -impressions_last30 | 0.595 | 0.70 | 0.64 | 0.63 |
| Logistic Regression | 0.518 | 0.80 | 0.70 | 0.66 |
| Random Forest | 0.520 | 0.60 | 0.60 | 0.58 |

Standard error of P@100 at p ≈ 0.6 is about 0.05, so the P@K differences between rows are within noise.

Per-fold vs pooled AUC. Random Forest per-fold AUC is 0.55–0.66 (mean about 0.60). Pooled out-of-fold AUC is 0.52 because score levels differ between folds. Both are reported. The W05 and W06 numbers differ partly for this reason, and partly because the feature table was rebuilt between them (45 vs 43 features, base rate 0.49 vs 0.529).

Error analysis (Random Forest, 2026-06-15).

- High-click pages are where the model is worst. AUC by clicks_last30 bucket is 0.51 (10–20), 0.49 (20–50), 0.39 (50–200), and 0.31 (>200). This is consistent with regression to the mean: pages with unusually many recent clicks tend to fall back.
- Poor-position pages are under-predicted. For average position 12–20, the actual decline rate is 0.70 and the mean prediction is 0.55. For position > 20, it is 0.72 vs 0.49.
- Confident misses are young pages (27–38 days old) with strong positions (about 2.5–5.8) that nonetheless declined.
- Confident false alarms are pages around position 11–12 that held up.
## 6. Interpretation



## 7. Recommendation

The ranked actions or decisions your output supports, and how a FlyRank editor would use them
tomorrow. State your confidence and the limits explicitly.

## 8. Reproducibility

The exact commands to re-run everything from a fresh clone, your random seeds, and your
environment (`pip freeze` highlights or `requirements.txt` deltas).

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
