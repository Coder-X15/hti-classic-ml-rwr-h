# Phytochemical-Protein Interaction Prediction

This project ranks candidate human protein targets for phytochemical compounds using curated IMPPAT compound-target associations. The modeling notebook is [work.ipynb](./work.ipynb).

## Data and features

The notebook reads files from `datasets/imppat_hti_aac_ecfp4_v1/`:

- 2,319 compounds represented by 2,048-bit ECFP4 fingerprints.
- 1,341 reviewed human targets represented by 20-dimensional amino-acid-composition (AAC) features.
- 14,166 curated positive compound-target associations.
- An unlabeled universe containing the compound-target pairs without a curated positive association.
- Herb-compound edges used by the heterogeneous graph model.

Pairs without a curated association are **unlabeled**, not experimentally verified negatives.

## Evaluation method

The notebook evaluates four methods over five independent random seeds (`42`-`46`). Each seed uses a 70%/15%/15% **cold-compound split**, keeping every compound's known interactions within one partition. The split is repeated with each seed; results are reported as mean +/- sample standard deviation across seeds.

Validation and test rankings score every target protein for each held-out compound, not a sampled 1:50 candidate subset. The candidate universe includes all 1,341 targets except any known training-positive pairs; held-out curated interactions are relevant, and unrecorded pairs remain unlabeled. Metrics include average precision (PR-AUC), mean reciprocal rank (MRR), and per-compound Hits@1, Hits@10, and Hits@50.

### Models

1. **Two-step PU:** an Isolation Forest selects the lowest-support 20% from a 5:1 sampled training background as reliable-negative candidates. A gradient-boosted classifier is then retrained for up to two expansion rounds, adding candidates scored below 0.01.
2. **Elkan-Noto:** a gradient-boosted classifier trained on positive and sampled unlabeled training pairs. The mean score on held-out validation positives estimates the selection probability used to calibrate test scores.
3. **Cost-sensitive SVM:** a `LinearSVC` trained on positive and sampled unlabeled pairs, with the positive-class weight set to the full unlabeled-to-positive universe ratio (3,095,613 / 14,166).
4. **RWR-H:** random walk with restart over a heterogeneous herb-compound-protein graph. Herb-compound edges are retained; compound-protein edges are training positives only. The restart probability is 0.15, with convergence tolerance `1e-10` and a 200-iteration limit.

ECFP4 and AAC vectors are concatenated for pairwise classifier features. XGBoost is used for the tree models when installed; otherwise the notebook falls back to scikit-learn `HistGradientBoostingClassifier`. In the environment used for this report, XGBoost was unavailable and the fallback was used.

## Five-seed performance report

Both validation and test metrics are produced by the notebook and summarized here as mean +/- sample standard deviation. Test metrics are reported after the same full-target ranking protocol as validation; these are retrospective results, not calibrated probabilities or proof of biological interaction.

<!-- FIVE_SEED_RESULTS_START -->
Metrics below are shown as mean +/- sample standard deviation (`n=5`).

### Validation

| Model | PR-AUC | MRR | Hits@1 | Hits@10 | Hits@50 |
|---|---:|---:|---:|---:|---:|
| Two-step PU | 0.0221 +/- 0.0049 | 0.2738 +/- 0.0741 | 0.1615 +/- 0.0712 | 0.5083 +/- 0.1010 | 0.7358 +/- 0.0715 |
| Elkan-Noto | 0.0525 +/- 0.0123 | 0.3291 +/- 0.0539 | 0.1945 +/- 0.0517 | 0.5945 +/- 0.0694 | 0.6055 +/- 0.0587 |
| Cost-sensitive SVM | 0.0149 +/- 0.0035 | 0.0267 +/- 0.0057 | 0.0000 +/- 0.0000 | 0.0440 +/- 0.0286 | 0.4000 +/- 0.0701 |
| RWR-H | 0.0956 +/- 0.0152 | 0.3962 +/- 0.0339 | 0.2587 +/- 0.0364 | 0.6624 +/- 0.0508 | 0.8569 +/- 0.0322 |

### Test

| Model | PR-AUC | MRR | Hits@1 | Hits@10 | Hits@50 |
|---|---:|---:|---:|---:|---:|
| Two-step PU | 0.0257 +/- 0.0072 | 0.2635 +/- 0.0266 | 0.1468 +/- 0.0251 | 0.4972 +/- 0.0732 | 0.7468 +/- 0.0545 |
| Elkan-Noto | 0.0565 +/- 0.0082 | 0.3408 +/- 0.0140 | 0.2092 +/- 0.0120 | 0.5945 +/- 0.0592 | 0.6037 +/- 0.0533 |
| Cost-sensitive SVM | 0.0171 +/- 0.0028 | 0.0326 +/- 0.0097 | 0.0037 +/- 0.0050 | 0.0661 +/- 0.0442 | 0.3945 +/- 0.0481 |
| RWR-H | 0.0937 +/- 0.0089 | 0.3958 +/- 0.0508 | 0.2606 +/- 0.0513 | 0.6716 +/- 0.0369 | 0.8679 +/- 0.0221 |

The unrounded per-seed measurements and aggregation are generated in [work.ipynb](./work.ipynb). The standard deviation describes variability across random compound splits and training-sample/model seeds.
<!-- FIVE_SEED_RESULTS_END -->

## Run

From the repository root, open and run [work.ipynb](./work.ipynb) in a Python environment with `numpy`, `pandas`, `matplotlib`, `scikit-learn`, and `scipy` installed. XGBoost is optional. The input data should be available at `datasets/imppat_hti_aac_ecfp4_v1/`.

The notebook also provides `rank_unlabeled_targets(compound_id, model_name=..., top_k=...)` for target prioritization using the average scores from the five fitted seed models. Each ranked compound-target row includes the standardized phytochemical name and herb associations traced through the herb-compound edges, enriched with `herbs.tsv` and `compounds.tsv` (including herb ID and reported plant part). These are derived herb-compound-target paths, not direct evidence that a herb engages a target. Ranked candidates are hypotheses for follow-up, not confirmed interactions.
