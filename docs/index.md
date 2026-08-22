---
Title: Refresh Priority Scoring — A FlyRank ML Internship Capstone
---

# Refresh Priority Scoring: Finding Which Content to Refresh First

**Author:** Ramanjeet Kaur · **Lane:** Refresh / Content Opportunity Scoring
**Repo:** https://github.com/ramanchauhan2271-dev/flyrank-ml-internship
**Date:** August 2026

> Built on the FlyRank ML Internship dataset — [https://flyrank.ai](https://flyrank.ai)

---

## Abstract

Content teams can't manually review every page, so the question this project asks is: **which
pages should be refreshed first, and can a model beat a simple hand-written rule at picking the
ones that are genuinely declining?** The approach builds 90-day traffic and engagement features
from an anonymized slice of FlyRank's content dataset (30,000 pages, 32 clients), trains a
logistic regression and a random forest against a transparent baseline rule, and validates with a
client-grouped train/test split so no client's pages leak across the split. Under this honest
split, the best model reaches **Precision@50 of 0.72–0.76**, versus the baseline rule's **0.24** —
roughly a **3x lift** — after a leakage check showed a naive random split had inflated the same
model's score to 0.96 purely by memorizing per-client traffic patterns. The features that actually
drive the ranking are recency signals like impression and session activity, not the label-adjacent
trend fields, which were deliberately excluded. The output is a ranked action queue with reason
codes that flags roughly 30% of content — every refresh and consolidation call, plus borderline
predictions — for mandatory human review, built as a prioritization aid, not an autonomous
decision-maker.

## 1. Introduction / Problem Statement

Content teams sit on thousands of pages and can't manually audit all of them every week. The
decision this project supports is: **given a page's traffic, engagement, and metadata signals,
which pages are declining and should be reviewed for refresh first** — and is a learned score
actually better than a simple, explainable rule at making that call? The unit of analysis is a
single content page. The output is a ranked queue with a suggested action and a reason code. The
action a human takes from it is picking what to review first in a weekly planning cycle. Getting
this wrong is costly in both directions: refreshing healthy content wastes editorial time, while
missing a genuinely declining page means it keeps losing traffic un-noticed. Data/ML helps here
because "declining" isn't visible from any single metric — it's a pattern across traffic,
engagement, and content-age signals that a hand rule can only approximate.

## 2. Data

- **Release:** the anonymized starter slice, `data/raw/content_refresh_anonymized.csv`, processed
  into a 30,000-row × 52-column feature vector by the repo's own `01_prepare_features.py`.
- **Scope:** 32 anonymized clients, pseudonymous `client_id` / `content_id` — used only for
  grouping in the split, never as model features.
- **Date windows:** rolling 90-day traffic/engagement windows (`log_impressions_90d`,
  `log_clicks_90d`, `log_sessions_90d`, `log_ai_sessions_90d`), plus `content_age_days` and
  `days_since_last_update`.
- **Label:** `is_declining_label = (trend_direction == "down")`.
- **Deliberately excluded:** `trend_direction` and `trend_pct` — both label-derived and a direct
  leakage risk (confirmed in the leakage audit, Section 3 below).
- **Public-safe:** no client names, domains, URLs, titles, or keywords appear anywhere in this
  dataset or in this paper, per the repo's `DATA_USE.md`.

## 3. Methodology

**Baseline:** a transparent, hand-written "fix this first" rule (built in Week 4) that scores
pages using visible signals only — no learned weights. It scores **Precision@50 = 0.24**.

**Models:** Logistic Regression and Random Forest, trained on the same numeric + categorical
feature set (search volume, competition, CPC, word/char count, the 90-day log-traffic features,
content age, update recency, CTR, average position, engagement-recency features).

**Split design:** a **client-grouped** `GroupShuffleSplit` holding out ~20% of *clients* — not
rows — so no client's pages appear in both train and test. This mirrors the repo's own
`03_train_model.py` for a fair, like-for-like comparison against the baseline.

**Leakage checks:**
- Confirmed `trend_direction` / `trend_pct` are absent from the feature set.
- Ran a confession test: injecting `trend_pct` as a feature on the same split pushed Precision@50
  from ~0.72 toward **1.00** — proof the validation harness would actually catch a real leak.
- Compared a naive **random** row split against the **grouped** client split on the identical
  model: random split scored **0.96**, grouped split scored **0.72** — a **0.24 gap** that is
  almost entirely the model memorizing individual clients' overall traffic level rather than
  learning what "declining" looks like in general. The grouped number is the one reported as the
  result.
- Base rate: 54.2% of pages are labeled declining, so precision numbers are read against that base
  rate, not in isolation.

## 4. Results (vs Baseline)

| Model | Precision@50 | Lift vs. baseline |
|---|---|---|
| Week-4 Baseline (hand rule) | 0.24 | 1.0x |
| Random Forest (client-grouped split) | 0.72 | 3.0x |
| Logistic Regression (client-grouped split) | **0.76** | **3.17x** |
| *(for reference — random row split, not the reported result)* | *0.96* | *inflated by leakage* |

**Best model:** Logistic Regression. Overall classification numbers on the held-out clients:
accuracy 0.57, precision 0.56 (not-declining) / 0.59 (declining), recall 0.62 / 0.53.

**Top 5 features (honest permutation importance, grouped split):**

| Feature | Importance |
|---|---|
| `days_with_impressions` | 0.0368 |
| `log_clicks_90d` | 0.0104 |
| `avg_position` | 0.0098 |
| `log_impressions_90d` | 0.0064 |
| `log_sessions_90d` | 0.0052 |

**Error analysis:** of the top-50 pages the model ranked highest, 12 were false positives — and 9
of those 12 belonged to a single client, suggesting the model still over-indexes somewhat on a
client's overall traffic profile rather than purely page-level decline signals, even after
grouping the split.

## 5. Limitations

- **Correlational, not causal** — the model finds association between recency/traffic signals and
  the decline label; it does not explain *why* a page is declining.
- **Client concentration in errors** — false positives clustered heavily around one client,
  meaning the score should be read as directional, not a guarantee, for any single client.
- **Small client pool** — 32 clients is not enough to claim the model generalizes to client types
  outside this sample.
- **One bundled sample** — numbers reflect a single anonymized pull; a fresh pull would move the
  exact decimals (the repo's own committed report notes 0.68–0.74 is the expected range).
- **Decision support only** — this output ranks and suggests; it does not decide. See Section 6 for
  the explicit human-review gate.

## 6. Ranked Recommendations

The validated model's held-out scores (all 6,163 test-set pages, client-grouped split) feed a
decay-and-action framework built from real signals — content age, days since last update, and the
30-day traffic change (last 30d vs. prior 30d impressions):

| Action | Count | Share |
|---|---|---|
| Monitor (light-touch) | 4,250 | 69.0% |
| Boost | 1,029 | 16.7% |
| Monitor | 481 | 7.8% |
| Refresh | 273 | 4.4% |
| Consolidate/Retire | 130 | 2.1% |

Each action carries a **reason code** so a reviewer can see *why* — e.g. RC01 = high decay score
(≥0.6) + traffic down >10% → time-sensitive refresh; RC02 = traffic up >20% → boost the momentum;
RC04 = old, low-traffic, high-decay → consolidation candidate. **29.8% of the queue is flagged for
mandatory human review** (every `Refresh` and `Consolidate/Retire` call, plus any borderline
prediction near the 0.5 decision boundary), and `Consolidate/Retire` always requires sign-off —
this is a prioritization aid, not an autonomous decision-maker; it never auto-publishes,
auto-edits, or auto-deletes anything.

Median decay score across the test set is 0.32 (0 = healthy, 1 = badly decayed), confirming most
content sits in "watch, don't touch yet" territory rather than needing urgent action — which
matches the base rate of ~54% declining but only ~7% flagged as urgent refresh/retire candidates.

**Monitoring:** in production this queue would be recomputed weekly on fresh traffic pulls, with a
retrain triggered on prediction drift beyond a set threshold in score-distribution standard
deviations.

## 7. Acknowledgments & Data Credit

Built on the FlyRank ML Internship dataset — [https://flyrank.ai](https://flyrank.ai). Thanks to
Mirza Ašćerić (Director of AI Development) and the FlyRank ML track for the structured
assignments and live sessions this capstone builds on.
