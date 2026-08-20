# Rule vs. Model: Prioritizing Under-Performing Pages for Content Review

## Abstract

Content teams with limited review capacity need a reliable way to prioritize which pages to check first for underperforming click-through rates. This study compares a simple rule-based baseline against a trained Logistic Regression model, both scoring pages by how far their CTR falls below their position tier's expected value, using one month of FlyRank's production search data (~1.04M page-days, client-grouped train/test split). Contrary to the original hypothesis, the rule-based baseline outperformed the model at precision@20 and precision@50 (0.40/0.58 vs. 0.35/0.34), a result confirmed stable across two independent client-grouped test folds. Both methods substantially beat a random baseline, and validation checks (leakage test, split-type comparison) confirmed the pipeline itself is sound. The practical takeaway is a ranked, human-reviewed action queue built on the baseline score rather than the model — with the caveat that this reversal, while reproducible within this month's data, should be re-tested on a second month before being treated as a stable conclusion.

## Introduction

Search-visible content that ranks well but fails to earn its expected share of clicks represents a hidden opportunity: the page is already found, but something — a weak snippet, a mismatched title, an unhelpful meta description, or simply query intent the content doesn't fully answer — is costing it clicks it should be getting. For a content team, the challenge isn't identifying that this happens; it's deciding, out of thousands of pages, which ones to review first when review time is limited.

This work asks whether a trained model can make that prioritization decision more reliably than a simple, interpretable rule of thumb. The answer matters practically: a content-ops team choosing between "build a model" and "keep a simple heuristic" needs to know whether the added complexity of a model actually buys better prioritization, or whether a transparent, easy-to-explain rule does the job just as well — or, as this study finds, better.

## Data

This study uses the FlyRank ML Internship dataset (`FlyRank/internship-warehouse` on Hugging Face), specifically the `fact_content_daily_performance` table for the March 2026 partition — one month out of a release spanning roughly 79 million rows in total. Analysis is restricted to rows with confirmed Google Search Console coverage (`gsc_data_available IS TRUE`) and at least 50 impressions, the same filter used consistently throughout this internship's earlier weekly assignments. This threshold excludes pages with too little search exposure for a click-through-rate signal to be meaningful — roughly 1.04 million rows (about 37% of the month) remain after filtering.

No client names, domains, URLs, or raw exports appear anywhere in this study or its outputs; all client and content identifiers are hashed (`client_hash_id`, `content_hash_id`).

## Methodology

**Label:** each page-month is assigned an expected click-through rate — the median CTR among all pages in the same average-position tier (`top_3`, `page_1`, `page_2`, `page_3_5`, `deep`, bucketed from Google's average search position). The gap between a page's expected and actual CTR (`ctr_gap`) is the underlying signal both methods below try to rank by. The evaluation label, `needs_review`, marks the bottom quartile of actual CTR within each tier — a self-defined proxy, not one of FlyRank's own pre-computed flags, which an earlier signal audit found unreliable for this purpose.

**Baseline:** a simple rule scores each page by `ctr_gap`, using tier-mean CTR (computed on the training split only, to avoid leakage) as the "expected" reference. This baseline is disclosed as scale-inconsistent across tiers — a large gap in a high-CTR tier isn't the same magnitude as a large gap in a near-zero-CTR tier — a known weakness carried forward from earlier development work.

**Model:** a Logistic Regression classifier trained on three features — impression volume, average position, and position tier — predicts the probability that a page belongs in the `needs_review` set.

**Validation design:** both methods are evaluated on the same held-out test split, grouped by client (`client_hash_id`) so that no client's pages appear in both train and test — preventing the model from learning client-specific patterns rather than generalizable signal. Because a naive `GroupKFold` split can produce highly imbalanced folds (some folds place only a single client in the test set), folds were scanned and the first fold with at least three distinct test clients was used, yielding 33 training clients and 8 test clients.

**Leakage checks:** the evaluation harness was validated by deliberately adding the true CTR as a model feature and confirming precision rose as expected (a positive control), and by confirming no FlyRank product-derived flag columns were present in the honest feature set. A comparison against a naive random (non-grouped) split confirmed the grouped split does not artificially inflate performance.

## Results

| Method | Precision@20 | Precision@50 |
|---|---|---|
| Baseline (rule-based) | 0.40 | 0.58 |
| Model (Logistic Regression) | 0.35 | 0.34 |
| Base rate (random) | 0.25 | 0.25 |

Contrary to the original hypothesis, the rule-based baseline outperformed the trained model at both precision@20 and precision@50 in this study's reproducible run. Both methods clearly beat the random base rate, confirming that both are picking up real signal — the baseline's rule and the model's three-feature approach simply differ in how well that signal translates into a ranked queue.

This result was checked across two independently sampled client-grouped test folds and held in the same direction both times, so it is not attributable to one unlucky split. A plausible explanation: the baseline's tier-relative CTR gap, despite its known scale-inconsistency across tiers, may capture more genuine signal in practice than the model's limited three-feature representation — impression count, position, and position tier alone may be too coarse to outperform a targeted heuristic on this particular task.

Two validation checks confirm the pipeline itself is sound rather than broken: adding the true CTR as a feature (a deliberate leakage test) raised the model's precision@50 from 0.34 to 0.38, confirming the evaluation harness correctly rewards features that shouldn't be available; and a naive random (non-grouped) split scored precision@50 of 0.00 versus 0.34 for the proper grouped split, confirming the client-grouped validation design is doing meaningful work.

## Limitations

**Single closed month:** this study uses one historical month (March 2026) with no forward-looking validation. Results describe what was observed in that period, not a forecast of future performance.

**Result reversal is itself a limitation:** in earlier development work on this same task, the model consistently outperformed the baseline; this study's own reproducible run found the opposite. That reversal held across two different test folds, so it is not a one-off fluke — but it does mean the direction of "which method wins" has proven unstable across runs and splits, and should not be treated as settled without testing on at least one additional month of data.

**Small, uneven client population:** the dataset's grouped-by-client validation split had to be manually selected because the default splitting produced highly imbalanced folds (some placing only a single client in the entire test set). The fold used here (33 training clients, 8 test clients) is a reasonable choice, but a different fold could plausibly have produced different headline numbers — this is a real source of sampling variance, not a settled ground truth.

**Baseline's known scale problem, still winning anyway:** the rule-based baseline's score is not directly comparable across position tiers, a disclosed methodological weakness — yet it still outperformed the model in this run, suggesting either that this weakness matters less in practice than expected, or that the model's feature set is simply too limited.

**Severe class imbalance in any ranked output:** the highest-confidence tier of any resulting review queue is dominated by low-impression, zero-click pages — a pattern observed consistently regardless of which method (rule or model) produces the ranking, and one that requires human filtering before any action is taken.

**Claim language:** all findings in this study are stated as observed, directional, or decision-support — never causal. No controlled experiment (such as an A/B test) was run, so no claim is made that acting on these findings will change search performance.

## Ranked recommendations

Given this study's own reproducible result — the rule-based baseline outperforming the trained model — the practical recommendation is to build the content review queue on the baseline's `ctr_gap` score, not the model's prediction, at least until a second month of data confirms or overturns this finding.

A recommended workflow for a content team adopting this queue:

1. **Rank pages by baseline score, filtered by impression volume.** A meaningful share of the highest-priority tier still consists of low-impression, zero-click pages; these should be set aside for a lower-priority pass rather than reviewed first, since low volume itself can drive an extreme score without necessarily indicating a fixable problem.
2. **Route the impression-qualified subset to human review**, checking for two known confounds before acting: whether a flagged page was recently updated and may simply be waiting on search-engine re-indexing (a temporary false positive), and whether the page's query already has a non-clickable SERP feature (like a featured snippet) that structurally caps its CTR regardless of content quality.
3. **Never automate action from this queue.** It is a starting triage order, not a certainty ranking — no bulk deletion, deprioritization, or rewriting should happen without a human review step, and no action should span multiple clients without separate per-client validation.
4. **Re-test on a second month before treating this as settled.** A result this clean, based on one month of data across only two validation folds, still warrants independent confirmation in either direction before it drives a permanent process change.

## Reproducibility

All code behind this study is available in the project repository, organized under `work/notebooks/`. The full pipeline — from data loading through the final ranked recommendations — is reproducible end-to-end from `work/notebooks/capstone.ipynb`, which runs top to bottom without errors. Supporting weekly notebooks (data contracting, leakage checks, signal auditing, baseline construction, model training, validation auditing, and the content action playbook) are also included in the same directory for anyone who wants to see the method built up step by step. Exported artifacts referenced in this paper — the precision comparison chart, the validation checks summary, and the ranked action queue metrics — are generated by the capstone notebook and committed under `work/outputs/` and `work/figures/`.

Repository: [github.com/gulgumusdere/flyrank-internship-ml](https://github.com/gulgumusdere/flyrank-internship-ml)
This work was completed with the assistance of an AI collaborator (Claude) for code drafting, debugging, and writing support, per the program's standard workflow — all analytical decisions, interpretation, and the final honest reporting of results were reviewed and owned by the author.

## Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. Data provided by [FlyRank](https://flyrank.ai).
