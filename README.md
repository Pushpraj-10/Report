# Depression Detection ML System Review

## 1) Project Overview

This review compares two versions of the same depression-detection effort:

- Old system: [MINOR/ml-service](MINOR/ml-service)
- New system: [Minor-Project/model](Minor-Project/model)

The old system built multiple architecture variants and a later HLG-Net path, but remained partially fragmented across scripts, mixed frameworks, and dataset-specific logic. The new system moves toward a unified, configurable, multi-dataset PyTorch pipeline centered on frame-sequence modeling and standardized reports.

The goal of this document is to provide a presentation-ready engineering assessment covering:

- What changed technically and why
- What is currently working
- What still fails or is brittle
- What should be done next to make the system robust in real-world deployment

---

## 2) Old System (MINOR) Analysis

### 2.1 Architecture and Pipeline Design

The old codebase combined:

- TensorFlow/Keras architectures and TFLite export workflows
- PyTorch HLG-Net training and evaluation
- Several dataset-specific training/evaluation scripts

This created flexibility for experimentation but weak consistency in training contracts, data semantics, and evaluation comparability.

### 2.2 Main Strengths

1. Broad architecture exploration (CNN, BiLSTM, CNN+LSTM, attention, multi-feature, HLG-Net).
2. Working end-to-end inference and export path for mobile targets in earlier TensorFlow pipelines.
3. Explicit cross-dataset evaluation attempts (DAIC-WOZ, EATD, depression dataset, RAVDESS).

### 2.3 Main Weaknesses

1. Fragmented code paths and inconsistent preprocessing contracts.
2. Mixed framework maintenance burden (TensorFlow + PyTorch).
3. Threshold-dependent collapse for some datasets.
4. Very strong in-domain results but weak generalization on harder external domains.

### 2.4 Evidence from Old Artifacts

From [MINOR/ml-service/artifacts/evaluation/hlgnet/results_all.json](MINOR/ml-service/artifacts/evaluation/hlgnet/results_all.json):

- depression_test: accuracy 0.99875, F1 0.9987, recall 0.9975
- eatd_test: accuracy 0.8523, F1 0.0, recall 0.0
- ravdess_test: accuracy 0.4028, F1 0.2182, recall 0.125
- daicwoz_test: accuracy 0.75, F1 0.0, recall 0.0

Interpretation:

- The old model could rank some cases reasonably (non-trivial AUC in places), but classification at fixed threshold often under-called positives on DAIC-WOZ/EATD.
- This points to calibration/domain-shift issues rather than pure feature absence.

### 2.5 Intermediate Multi-Feature Baselines

From [MINOR/ml-service/artifacts/models/multi_feature_combined/training_summary.json](MINOR/ml-service/artifacts/models/multi_feature_combined/training_summary.json) and [MINOR/ml-service/artifacts/models/multi_feature_v2/training_summary.json](MINOR/ml-service/artifacts/models/multi_feature_v2/training_summary.json):

- Validation AUC reached about 0.84 to 0.91 in some runs.
- Cross-dataset test F1 remained low despite better internal metrics.

This confirms the persistent pattern: internal improvements did not reliably transfer to external domains.

---

## 3) New System (Minor-Project) Analysis

### 3.1 Architectural Shift

The new system introduces:

- Config-driven orchestration using YAML files in [Minor-Project/model/configs](Minor-Project/model/configs)
- Unified multi-dataset preprocessing with frame-sequence contract
- A clearer candidate model family with shared trainer/evaluator logic
- Generalized reporting with per-dataset and macro metrics

### 3.2 Data and Processing Scale

From [Minor-Project/model/data/processed/multi_dataset/multi_preprocess_report.json](Minor-Project/model/data/processed/multi_dataset/multi_preprocess_report.json):

- Datasets processed: DAIC-WOZ, EATD-Corpus, dataset_depression, RAVDESS
- Total subjects: 1251
- Total segments: 6030
- Splits:
  - Train: 875 subjects, 4357 segments
  - Val: 188 subjects, 769 segments
  - Test: 188 subjects, 904 segments
- Feature setup: MFCC, feature_dim 120, frame-sequence standardized scaler

This is materially more scalable and reproducible than the old script-by-script flow.

### 3.3 Current Candidate Modeling Direction

The active reporting focus is currently centered on conv1d_lightweight_attention, with generalized and all-samples report snapshots.

Key artifacts:

- [Minor-Project/model/artifacts/reports/generalized_conv1d_lightweight_attention.json](Minor-Project/model/artifacts/reports/generalized_conv1d_lightweight_attention.json)
- [Minor-Project/model/artifacts/reports/all_samples__conv1d_lightweight_attention.json](Minor-Project/model/artifacts/reports/all_samples__conv1d_lightweight_attention.json)

---

## 4) Old vs New: What Changed and Why It Matters

### 4.1 System Engineering Changes

1. From script-centric to config-centric operation
	- Old: many specialized scripts, implicit assumptions
	- New: explicit YAML-driven behavior and consolidated pipelines

2. From mixed-framework burden to PyTorch-first coherence
	- Simplifies maintenance and model iteration

3. From mostly dataset-specific logic to multi-dataset generalized pipeline
	- Better for cross-domain experimentation and reproducibility

4. From partial metrics snapshots to structured, multi-granular reports
	- Per split, per dataset, overall, and macro summaries in JSON

### 4.2 ML Methodology Changes

1. Stronger emphasis on subject-level and segment-level contracts.
2. Explicit threshold calibration workflows in reporting.
3. Better separation of preprocessing, training, and evaluation responsibilities.

### 4.3 Net Effect

- Engineering maturity improved significantly.
- Performance portability across datasets is still the main unresolved scientific challenge.

---

## 5) Current System Deep Dive (As-Is)

### 5.1 Data Contract

- Audio is converted to frame-sequence MFCC features (120 dim).
- Segments are traced via manifest and split metadata.
- Scaler statistics are persisted for deterministic inference.

Artifacts:

- [Minor-Project/model/data/processed/multi_dataset/frame_manifest.csv](Minor-Project/model/data/processed/multi_dataset/frame_manifest.csv)
- [Minor-Project/model/data/processed/multi_dataset/frame_feature_scaler.json](Minor-Project/model/data/processed/multi_dataset/frame_feature_scaler.json)

### 5.2 Training/Evaluation Contract

- Candidate model selection and generalized trainer paths are implemented under [Minor-Project/model/src/training](Minor-Project/model/src/training).
- Cross-dataset report generation is under [Minor-Project/model/src/evaluation](Minor-Project/model/src/evaluation).
- Current threshold in the main reports is approximately 0.5606.

### 5.3 Practical Behavior Observed

There are two distinct states visible in artifacts:

1. Strong run family
	- High overall metrics with excellent discrimination on the largest dataset.

2. Weak/collapsed run family
	- Multiple evaluation files with F1 near 0 and conservative positive prediction behavior.

This indicates the pipeline can succeed, but run stability and protocol consistency are not yet guaranteed.

---

## 6) Experiments and Performance Evidence

### 6.1 Strong Generalized Report (Test Split Snapshot)

From [Minor-Project/model/artifacts/reports/generalized_conv1d_lightweight_attention.json](Minor-Project/model/artifacts/reports/generalized_conv1d_lightweight_attention.json):

- Overall classification (test):
  - Accuracy: 0.8723
  - Precision: 0.9833
  - Recall: 0.7195
  - F1: 0.8310
  - ROC-AUC: 0.9467

- Per-dataset behavior (test):
  - dataset_depression: F1 0.9818, recall 0.9643
  - ravdess: F1 0.4762, recall 0.3333
  - daic-woz: F1 0.0, recall 0.0
  - eatd_corpus: F1 0.0, recall 0.0

Interpretation:

- Even in a strong run, cross-domain positive recall remains fragile on DAIC-WOZ and EATD.

### 6.2 All-Samples Report (Broader Test Set)

From [Minor-Project/model/artifacts/reports/all_samples__conv1d_lightweight_attention.json](Minor-Project/model/artifacts/reports/all_samples__conv1d_lightweight_attention.json):

- Overall classification:
  - Accuracy: 0.8906
  - Precision: 0.9854
  - Recall: 0.7642
  - F1: 0.8608
  - ROC-AUC: 0.9723

- Cross-dataset recall still uneven:
  - dataset_depression recall: 0.9555
  - daic-woz recall: 0.0769
  - eatd_corpus recall: 0.3000
  - ravdess recall: 0.3152

Interpretation:

- The model is highly precise but conservative, missing many positives outside the dominant/easier domain.

### 6.3 Weak-Run Evidence (Important for Risk Assessment)

From [Minor-Project/model/experiments/results/evaluation/latest_eval_summary.json](Minor-Project/model/experiments/results/evaluation/latest_eval_summary.json):

- Several evaluated outputs show F1 = 0.0 across splits.
- Some datasets sit near chance-level thresholded classification despite moderate Pearson values.

Interpretation:

- Ranking signal may exist, but thresholded classification can collapse depending on run/protocol.
- This is a deployment risk unless calibration is dataset-aware and validated prospectively.

### 6.4 Tabular Experiment Instability Snapshot

From [Minor-Project/model/experiments/results/training_report.json](Minor-Project/model/experiments/results/training_report.json):

- Very strong training fit with weak validation/test consistency on tiny subject splits.

Interpretation:

- Overfitting risk remains real in low-subject regimes.

---

## 7) Core Technical Findings

1. Engineering quality improved substantially in the new system.
2. Reproducibility and traceability are better due to explicit config and artifact reporting.
3. The central unresolved issue is cross-dataset generalization, especially positive recall on DAIC-WOZ and EATD.
4. Reported performance can look strong overall while masking domain-specific failure.
5. Calibration and decision-threshold strategy are currently the biggest levers for practical gains.

---

## 8) Risks and Limitations

### 8.1 Dataset Shift and Label Semantics

- Different corpora have distinct recording conditions, language/emotion distributions, and label constructions.
- A single global threshold is insufficient for stable recall across all datasets.

### 8.2 Metric Interpretation Risk

- High accuracy and precision can coexist with poor recall under imbalance.
- Macro-level metrics should be treated as primary model-selection criteria for multi-domain deployment.

### 8.3 Run-to-Run Stability

- Presence of both strong and collapsed artifacts indicates sensitivity to protocol, calibration, or data split composition.

### 8.4 Clinical Product Risk

- False negatives are costly in depression detection.
- Current cross-domain recall behavior is not yet suitable for safety-critical usage without stronger safeguards.

---

## 9) Actionable Recommendations

### 9.1 Evaluation and Selection Policy

1. Make macro recall and per-dataset recall hard gates for model acceptance.
2. Keep ROC-AUC and PR-AUC as secondary ranking signals, not sole decision criteria.
3. Track confidence intervals via repeated stratified subject-level splits.

### 9.2 Calibration Strategy

1. Replace one global threshold with dataset-conditioned calibration where appropriate.
2. Add explicit operating-point optimization for target recall floors per dataset.
3. Persist calibration artifacts alongside model weights and verify in CI.

### 9.3 Domain Generalization Work

1. Increase effective diversity for DAIC-WOZ and EATD via balanced sampling and augmentation tuned per domain.
2. Expand domain-adaptation experiments with clear ablations and fixed random seeds.
3. Add leave-one-dataset-out validation to expose hidden overfitting.

### 9.4 Reproducibility and Governance

1. Version-lock preprocessing config, scaler, and threshold in one immutable run manifest.
2. Standardize report schema for every experiment branch.
3. Introduce a quality dashboard emphasizing per-dataset confusion matrices and FN rates.

---

## 10) PPT-Ready Summary

### Slide 1: Problem and Objective

- Build a robust voice-based depression detector that generalizes across heterogeneous datasets.

### Slide 2: Old vs New System

- Old: fragmented, mixed-framework, script-heavy experimentation.
- New: unified PyTorch pipeline, YAML-configured, multi-dataset frame-sequence processing, richer reporting.

### Slide 3: What Improved

- Better engineering discipline, reproducibility, and scalable preprocessing.
- Strong overall metrics in top runs.

### Slide 4: What Still Fails

- Cross-dataset recall remains low on DAIC-WOZ/EATD in key runs.
- Some runs collapse to near-zero F1 after thresholding.

### Slide 5: Why This Happens

- Domain shift, label mismatch, imbalance, and global-threshold limitations.

### Slide 6: Immediate Next Steps

- Use macro/per-dataset recall gates.
- Apply dataset-aware calibration.
- Run stronger domain-generalization ablations with fixed protocols.

### Slide 7: Final Verdict

- The new system is a major engineering upgrade over the old one.
- It is promising but not yet consistently reliable across domains.
- Priority should shift from architecture expansion to calibration, domain robustness, and evaluation rigor.

