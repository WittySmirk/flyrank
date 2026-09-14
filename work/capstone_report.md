# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Wyatt Degenhart
- **Lane:** Lane 2: Refresh / Content Opportunity Scoring
- **Repo:** flyrank
- **Date:** 2026-05-31 decision point (May → June window)

## Abstract

I score labelable pages by how much attention they need so site maintainers know which ones to target first. I use May search + engagement aggregates from the FlyRank warehouse with the label `declined_30d_future` (June impressions under 80% of May). A depth-4 decision tree, grouped by client, ranks declining pages at P@100 0.98 vs the frozen Rule-2 baseline 0.89 and base rate 0.641. The win holds over 5 grouped seeds and survives a random-split audit; about 1 in 5 confident picks self-heal. The paper ends in a review-first queue with reason codes — momentum ranking, not proof a refresh will work.

Decision-support queue for Refresh / Content Opportunity Scoring, from May features to a June label at D = 2026-05-31.

## 1. Problem framing

### Lane 2: Refresh / Content Opportunity Scoring

Which pages should be reviewed first for refresh, expansion, protection, pruning, or monitoring?

I chose refresh/content opportunity scoring because it seems integral to FlyRank's core product. From what I've gathered, picking and ranking pages that a client should refresh in order to optimize traffic would need to use most of the observable data, and be critical to the business problem.

My lane *Refresh/Content Opportunity Scoring* is a combination of scoring and ranking. Fundamentally the task is to score the pages based on how much attention is needed, but also rank them based off these scores so site maintainers know which ones to target first.

The work empowers business' to choose which pages to first further optimize to improve overall SEO. Project managers/owners would mainly act on this recommendation; allocating resources appropriately to update the suggested pages. A wrong recommendation would mainly cost the time of the business. There is also the possibility that a refresh could not improve the SEO of a page, but have the inverse effect. Additionally a missed declining page would lead to continued traffic loss that could've been improved. There are many signals that impact the ranking of which pages need attention and these signals shift over time. This lends this problem to be hard to predict with a simple logic rule, and an effective problem for ML to tackle.

The pattern is too messy for an if-statement because there are many factors that go into which pages need to be refreshed. For example just observing the trend and ranking based on that might not tell the full story; an old outdated blogpost could have a worst trend than a real product page, but the product page may be more important to refresh first.

Success metric: Precision at 10 — how many of the top 10 pages the model ranks highest actually needed a refresh. See how many of the top 10 actually have a negative trend direction in the data.

Careful words from the start: The work I will complete in the program will be creating a model able to observe trends based on the provided dataset and predict which pages are most in need of a refresh. This list prediction will help business owners and website builders allocate their SEO optimization efforts more efficiently; in a linear fashion. The work will not be able to predict Google/ChatGPT/Claude's recommendation engine or prove that if you refresh X page the SEO will improve by Y. It will only create predictions based on observable trends in FlyRank's dataset and be a good starting point for allocating SEO optimization efforts.

- Backing (from `capstone.ipynb` Cell 2): lane Refresh / Content Opportunity Scoring (score + rank); decision: which labelable pages to review first for refresh; actor: project managers / owners; wrong-call cost: wasted refresh time + missed decliner keeps losing traffic; metric: precision@K (P@10 primary in W02; P@50/P@100 reported as the stable claim).

## 2. Data safety

One row is one piece of content for one client's combined GSC and GA4 data on a given date. The dates are from 1/27/2025 until 6/30/2026.

Tables: `dim_clients` (104 clients) + `dim_content` + `fact_content_daily_performance` (78,835,655 rows) + `fact_content_query_90d`. I iterate on the two month partitions that cover the window (`month=2026-05` = prior, 23,381,448 May+Jun rows with June; `_sample` is June-only so it cannot feed the prior window on its own).

Decision point D = 2026-05-31. Features come from (D-30, D]; the label is June (D, D+30] so it was not knowable at decision time. Facts run through 2026-06-30, so the label window is fully observed.

What I excluded and why: sessions_paid — paid-for traffic should be irrelevant to refresh. ai_* vendor splits — AI traffic is measured but which vendor is irrelevant. IDs (`client_hash_id`, `content_hash_id`) group and split only, never features.

Data limits, in my words: All clients can't be compared equally because some clients have much more data than others; they start tracking at different times. 37.6% of rows have no available GA4 data and only GSC data. The zeroes in the GA4 columns thus aren't real tracked zeroes, they mean no data tracked. There's 6390 rows of duplicate data, so deduplication must be handled before any modelling.

Labelable filters (survivorship note, stated with every finding): prior_imp>=100, prior_obs>=7, future_obs>=7 — 100,785 of 333,275 window rows (30.3%, 41 of 52 clients). 232,490 low-volume/thin-history rows were filtered out, not scored as "not declined". Label: `declined_30d_future` = 1 when June impressions are under 80% of May; base rate 0.655 (0.641 on the grouped test side).

- Backing (from `capstone.ipynb` Cell 4): `work/outputs/baseline_features.csv` — 333,275 window rows, 100,785 labelable (30.2%), base rate 0.655; 52 clients in window, 41 labelable; one row per content; label window fully observed (D+30 == 2026-06-30).

## 3. Baseline

Baseline (frozen Rule 2, in plain words): a page is worth refreshing when its impressions are falling across the prior month **and** its position is slipping too — the classic fading star. Falling = second half under 80% of the first half's impressions; slipping = weighted position got worse. Reason codes: `IMPRESSIONS_FALLING`, `POSITION_SLIPPING`. Rule 2 stays a tie-break only, never a feature.

## 4. Model / analysis

All features can be known on 2026-05-31. We generate prior-30d by aggregating this with static metadata. imp_ratio and pos_delta split the prior month in half (like the baseline). declined_30d_future consists of June and is only used for testing. We use one matrix for both models. We replace NaN with the median from the training set.

The two main models that fit the refresh/opportunity scoring track are decision tree and logistical regression, of which we will test both against the rule created in Week 4. Both models evaluate ranking at precision@k. A decision tree with depth 4 and min_samples_leaf 100 was the primary model. A standardize L2 logistical regression model was the comparison model. Both are fit on the same split and reported in one table against the baseline. The tree keeps its job only if it also wins the numbers; otherwise the table decides.

Validation: grouped by client (70/30, seed 42) so every row of a client stays on one side. The question becomes "does it work on a client it never saw?" — the deployable question. Week 6 holds everything fixed except the split: random rows (39 of 41 clients on BOTH sides, memorizes client character) vs grouped (0 shared). A pure time split is noted as future work.

Leakage hunt (three families, each ends in a number): label-derived columns (inject future/prior, watch the confession to 1.000, then remove it); future/overlapping windows (every feature tagged to end at D; query-table `impressions_90d`/`*_last30` CONTAIN June so only `*_prev30` would ever be safe — this work uses the daily fact only); product flags (none exist; Rule 2 breaks ties only). Solo-feature scan tops at imp_ratio 0.860 with ~0% ties — momentum, not a miracle.

## 5. Evaluation

Verdict: decision tree d=4 wins, beats baseline at every K, and beats logistic regression at P@50/P@100.

| model | P@10 | P@20 | P@50 | P@100 |
|---|---|---|---|---|
| baseline (Rule 2) | 0.80 | 0.85 | 0.86 | 0.89 |
| logistic regression | 0.80 | 0.85 | 0.84 | 0.79 |
| decision tree d=3 | 0.80 | 0.85 | 0.94 | 0.94 |
| decision tree d=4 | 1.00 | 1.00 | 0.98 | 0.98 |
| decision tree d=5 | 1.00 | 1.00 | 1.00 | 0.99 |
| base rate (grouped test, 12 clients) | 0.641 | | | |

Some observations from the data:

1. Logistic regression loses to the frozen rule at P@100 (0.79 vs 0.89).
2. The tree's perfect head is partly the tie-break. A depth-4 tree has only 16 distinct scores; the head of the ranking is the crash-leaf (imp_ratio ≤ 0.65) sorted by Rule 2's May impression loss. That leaf is 82.5% positive on test, not 100% — the perfect P@10/20 is the ranking on top of a strong-but-imperfect leaf.
3. The P@10 number flips with depth (0.80 at d=3, 1.00 at d=4/5). The stable claim is P@50/P@100 ≈ 0.94–0.99; the exact head is sensitive to the chosen tree size.

What the tree leans on: imp_ratio (the May mid-month collapse) is the tree's #1 split (importance 0.474) and logistic regression's #1 coefficient. Sanity check: a page that already lost most of its impressions in the second half of May usually keeps losing them in June — that is momentum, not magic, and it is not leakage (the ratio ends at D). content_age_days is the tree's second split (0.246): older pages are the classic refresh targets.

The audit keeps the verdict: the naive random split flatters logistic regression (+12 to +17 pts at P@50/P@100) and Rule 2 (+5 to +20 pts) because 39 of 41 clients sit on BOTH sides — client memorization. The tree is split-proof (gap ≈ 0 at P@50/P@100). Over 5 grouped seeds the tree holds P@50 0.972±0.016 and P@100 0.980±0.006 while Rule 2 spreads wider (P@100 0.924±0.025); the tree stayed above the baseline in every draw. Positive controls hit 1.000, so the harness does catch a leak when one exists.

## 6. Interpretation

What the model actually found, in plain words (your Cells 7–9): pages that already collapsed in the second half of May usually keep declining in June, and older pages are the classic refresh targets. A well-understood negative result: logistic regression loses to the frozen rule at P@100, and ~1-in-5 confident picks self-heal — so the list is "already bleeding," not "will bleed."

## 7. Recommendation

The queue suggests which pages to review first and does not guarantee refresh wins. What to do first? Tackle the pages at the top of the list going down.

BEFORE acting on any REVIEW_FIRST row, a human confirms: 1. Open the page: is it live, indexable, canonical — not redirected or mid-migration? 2. Check the dip: one bad week or a full-half collapse? Prefer WATCH when it looks like a single spike. 3. Check ownership and staleness: who owns it, was it edited elsewhere, is a campaign ending? 4. Check revenue/brand risk: top-traffic or sensitive pages need owner sign-off even at rank 1. 5. Log the decision so the next audit has a treatment record.

NO-GO list: never auto-publish; never act on non-labelable rows; never use query `*_last30` columns; never claim "refresh will recover X%"; never refresh solely for missing keyword/word-count; never compare raw ranks across clients as quality scores.

Review queue (from `capstone.ipynb` Cell 12 — top 500 of 100,785 labelable, all REVIEW_FIRST, 100% carry SHARP_DROP/FALLING; reason mix: SHARP_DROP 500, POSITION_SLIPPING 449, THIN_ENGAGEMENT 373, OLD_PAGE_365D 72, MISSING_WORDCOUNT 42, STALE_90D 18, MISSING_KEYWORD 5). Top-10 head (prob_decline 0.9497 throughout, ranked by Rule-2 tie-break): ranks 1–10 show imp_ratio 0.039–0.518 with position slippage on most rows — see `work/outputs/action_playbook_queue.csv`.

## 8. Reproducibility

- Source notebook: `work/notebooks/capstone.ipynb` (16 cells, runs top-to-bottom; backing cells 2/4/6/8/10/12/14 regenerate every number cited above).
- Frozen artifacts in `work/outputs/`: `baseline_features.csv`, `baseline_action_score.csv`, `action_playbook_queue.csv` (top 500), `action_playbook_summary.json`, `action_playbook_monitor.json`, `capstone_honest_table.csv`, `capstone_p100.svg`.
- Seeds/split: grouped by client 70/30, seed 42; 5-seed audit holds P@50/P@100; tree `max_depth=4, min_samples_leaf=100, random_state=42`; NaN → train-median.
- Re-run (fresh clone): `pip install -r requirements.txt` then Run All on `work/notebooks/capstone.ipynb`. Label definition: `declined_30d_future = (future_imp < 0.8 * prior_imp)` on labelable (`prior_imp>=100 & prior_obs>=7 & future_obs>=7`).
- Honest claim (from `action_playbook_summary.json`): observed May→June on one snapshot: the model ranks/flags declining pages at P@100 ~0.98 on held-out clients; ~1-in-5 confident picks recover without action; head P@10-20 is seed-sensitive so the reliable claim is P@50/P@100; review-first candidates, not causal refresh effects.

## Acknowledgments & data credit

FlyRank warehouse: `dim_clients` + `dim_content` + `fact_content_daily_performance` + `fact_content_query_90d`, via the two month partitions covering D = 2026-05-31. Analysis notebooks: `work/notebooks/capstone.ipynb` (this paper) building on W04–W07. No client-identifying details published; hashes group/split only.

---