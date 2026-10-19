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

The notebook evaluates four methods over the five fixed compound-held-out folds in `prep/folds_ct_compound.npz` (prepared with seed 42 and validation share 0.15). Each fold keeps a compound's known interactions within one partition. The notebook uses the saved positive and sampled-negative pairs without regenerating them.

The prepared folds use 10 sampled unlabeled links per positive for training and validation and 50 per positive for test. PR-AUC uses the sampled negatives; filtered ranking compares against the full target universe and excludes all other known positive targets for each compound. Evaluation reports PR-AUC, filtered Hits@10, Hits@50, MRR, paired BPR loss, and positive/negative link counts, matching `metrics.py`.

### Models

1. **Two-step PU:** an Isolation Forest selects the lowest-support 20% from the prepared 10:1 sampled training negatives as reliable-negative candidates. A gradient-boosted classifier is then retrained for up to two expansion rounds, adding candidates scored below 0.01.
2. **Elkan-Noto:** a gradient-boosted classifier trained on positive and sampled unlabeled training pairs. The mean score on held-out validation positives estimates the selection probability used to calibrate test scores.
3. **Cost-sensitive SVM:** a `LinearSVC` trained on positive and sampled unlabeled pairs, with the positive-class weight set to the full unlabeled-to-positive universe ratio (3,095,613 / 14,166).
4. **RWR-H:** random walk with restart over a heterogeneous herb-compound-protein graph. Herb-compound edges are retained; compound-protein edges are training positives only. The restart probability is 0.15, with convergence tolerance `1e-10` and a 200-iteration limit.

ECFP4 and AAC vectors are concatenated for pairwise classifier features. XGBoost is used for the tree models when installed; otherwise the notebook falls back to scikit-learn `HistGradientBoostingClassifier`. In the environment used for this report, XGBoost was unavailable and the fallback was used.

## Prepared-fold performance report

Validation and test metrics are produced by the notebook and summarized as mean +/- sample standard deviation across the five prepared folds. The result table below is intentionally generated when the notebook is run; it is not a claim that the prepared-fold evaluation has already been executed.

<!-- FIVE_SEED_RESULTS_START -->
Run [work.ipynb](./work.ipynb) to generate per-fold train, validation, and test metrics and their mean +/- sample standard deviation. The prior random-seed results used a different split and metric protocol and are not included here.
<!-- FIVE_SEED_RESULTS_END -->

## Run

From the repository root, open and run [work.ipynb](./work.ipynb) in a Python environment with `numpy`, `pandas`, `matplotlib`, `scikit-learn`, and `scipy` installed. XGBoost is optional. The input data should be available at `datasets/imppat_hti_aac_ecfp4_v1/`, with the prepared folds and metadata in `prep/`.

The notebook also provides `rank_unlabeled_targets(compound_id, model_name=..., top_k=...)` for target prioritization using the average scores from the five fitted seed models. Each ranked compound-target row includes the standardized phytochemical name and herb associations traced through the herb-compound edges, enriched with `herbs.tsv` and `compounds.tsv` (including herb ID and reported plant part). These are derived herb-compound-target paths, not direct evidence that a herb engages a target. Ranked candidates are hypotheses for follow-up, not confirmed interactions.
