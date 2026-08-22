## Social Post

Spent this capstone proving my own model wrong before trusting it. Trained a Precision@50 model to flag which content needs refreshing — first pass scored 0.96, which should've felt like a win. It wasn't. A random train/test split let the model memorize per-client traffic patterns instead of learning real decay signals. Switched to a client-grouped split, score dropped to 0.72–0.76 — still a 3x lift over a hand-written baseline rule, but now it's a number I actually believe.

## Employer Summary

Built a content-refresh prioritization model for FlyRank's ML internship, trained on 30,000 pages across 32 anonymized clients. Using a leakage-safe, client-grouped validation split, the model reaches Precision@50 of 0.76 versus a 0.24 baseline — a 3x lift — while flagging ~30% of predictions for mandatory human review. Delivered as a human-in-the-loop prioritization aid, not an autonomous decision system.
