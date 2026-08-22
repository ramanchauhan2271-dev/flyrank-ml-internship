## 5-Minute Demo Outline

**Question:** Which pages should content teams refresh first — and does a learned model actually beat a simple hand-written rule at picking them?

**Method:** Built 90-day traffic/engagement features on 30,000 pages across 32 clients, trained Logistic Regression + Random Forest against a transparent baseline rule, validated with a client-grouped split so no client's pages leak across train/test.

**Chart:** results_precision50.png — Precision@50 across baseline, Random Forest, and Logistic Regression on the identical grouped split.

**Honest result:** Best model (Logistic Regression) hits Precision@50 of 0.76 vs. the baseline's 0.24 — a ~3x lift. A naive random split had inflated this to 0.96 purely by memorizing per-client traffic patterns; the grouped number is the one reported.

**Recommendation:** Ship the ranked action queue as a human-in-the-loop prioritization aid — ~30% of pages flagged for mandatory review, nothing auto-executes.
