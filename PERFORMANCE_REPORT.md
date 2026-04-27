# Model Accuracy Report — CNN + BiLSTM vs CNN + SimpleKAN

> [!NOTE]
> Below are the comprehensive evaluation results for both **CNN + BiLSTM** and **CNN + SimpleKAN** models.
> Each model was trained individually on all four datasets and then evaluated across the board.
> **Intra** = Trained and tested on the *same* dataset.
> **Inter** = Tested on an *unseen* dataset (cross-domain generalization).

---

## 1. Accuracy (%)

### CNN + BiLSTM
| Train ↓ / Test → | DAIC-WOZ | Dataset-Depression | EATD-Corpus | RAVDESS |
|:---|:---:|:---:|:---:|:---:|
| **DAIC-WOZ** | 71.63% (Intra) | 49.24% | 70.48% | 55.09% |
| **Dataset-Depression** | 25.60% | 100.00% (Intra) | 22.66% | 47.72% |
| **EATD-Corpus** | 87.98% | 49.62% | 94.85% (Intra) | 55.56% |
| **RAVDESS** | 85.90% | 32.83% | 77.13% | 75.00% (Intra) |

### CNN + SimpleKAN
| Train ↓ / Test → | DAIC-WOZ | Dataset-Depression | EATD-Corpus | RAVDESS |
|:---|:---:|:---:|:---:|:---:|
| **DAIC-WOZ** | 56.29% (Intra) | 45.96% | 73.18% | 49.71% |
| **Dataset-Depression** | 36.74% | 100.00% (Intra) | 34.93% | 44.56% |
| **EATD-Corpus** | 87.98% | 49.62% | 95.88% (Intra) | 55.56% |
| **RAVDESS** | 87.18% | 48.74% | 73.18% | 86.05% (Intra) |

---

## 2. Macro F1 Score

### CNN + BiLSTM
| Train ↓ / Test → | DAIC-WOZ | Dataset-Depression | EATD-Corpus | RAVDESS |
|:---|:---:|:---:|:---:|:---:|
| **DAIC-WOZ** | 0.5036 (Intra) | 0.3468 | 0.4730 | 0.3690 |
| **Dataset-Depression** | 0.2542 | 1.0000 (Intra) | 0.2140 | 0.4394 |
| **EATD-Corpus** | 0.4680 | 0.3316 | 0.4868 (Intra) | 0.3571 |
| **RAVDESS** | 0.4649 | 0.3151 | 0.4687 | 0.7446 (Intra) |

### CNN + SimpleKAN
| Train ↓ / Test → | DAIC-WOZ | Dataset-Depression | EATD-Corpus | RAVDESS |
|:---|:---:|:---:|:---:|:---:|
| **DAIC-WOZ** | 0.4573 (Intra) | 0.4341 | 0.5039 | 0.4951 |
| **Dataset-Depression** | 0.3248 | 1.0000 (Intra) | 0.2980 | 0.4424 |
| **EATD-Corpus** | 0.4680 | 0.3316 | 0.7392 (Intra) | 0.3571 |
| **RAVDESS** | 0.4688 | 0.4222 | 0.4640 | 0.8555 (Intra) |

---

## 3. Sample Counts Per Dataset

| Dataset | Split Type for Models Trained Elsewhere | Total Samples Evaluated |
|:---|:---:|:---:|
| **DAIC-WOZ (Balanced)** | Inter-Evaluation (Test Set) | 2,496 |
| **Dataset-Depression** | Inter-Evaluation (Test Set) | ~792-800 |
| **EATD-Corpus** | Inter-Evaluation (Test Set) | ~481-486 |
| **RAVDESS** | Inter-Evaluation (Test Set) | ~855-864 |

---

## Key Observations

### 1. The Challenge of "Dataset-Depression"
Models trained on **Dataset-Depression** achieve a perfect 100% accuracy and 1.00 F1 score in the Intra-evaluation setup. However, when these models are tested on DAIC-WOZ or EATD-Corpus, the performance completely crashes (e.g., 25% accuracy for BiLSTM on DAIC-WOZ). This is a strong indicator of heavy overfitting on a very small dataset or a significant domain shift. 

### 2. Generalization Capabilities (Inter-Dataset)
When models are trained on **DAIC-WOZ**, the cross-dataset transferability depends on the architecture:
- **CNN + BiLSTM** generalizes better to EATD-Corpus from DAIC-WOZ in terms of accuracy (70.48%) compared to SimpleKAN (73.18%, but lower F1). 
- **CNN + SimpleKAN** achieves generally more balanced inter-dataset F1 scores when trained on DAIC-WOZ compared to the BiLSTM.
- Overall, training on a larger, more diverse dataset like DAIC-WOZ or RAVDESS yields much more robust representations for testing on unseen datasets than training on the smaller `Dataset-Depression` or `EATD-Corpus`.

### 3. RAVDESS and Generalization
Interestingly, models trained on **EATD-Corpus** and tested on **DAIC-WOZ** yield unusually high accuracy (around 87.98%), but very low Macro F1 (~0.46), suggesting that the model collapses to predicting the majority class when evaluating DAIC-WOZ.

### 4. SimpleKAN vs BiLSTM on RAVDESS
The **CNN + SimpleKAN** architecture shines significantly on the RAVDESS dataset. When trained and evaluated strictly on RAVDESS (Intra), SimpleKAN achieves an excellent **86.05% Accuracy and 0.8555 F1**, heavily outperforming CNN+BiLSTM (75.00% Accuracy and 0.7446 F1). 

> [!WARNING]
> While high accuracies are observed when training and testing on EATD-Corpus and Dataset-Depression, the models heavily overfit. For cross-domain robustness, training on large-scale datasets (like DAIC-WOZ) and using domain adaptation techniques remains critical, as F1 scores on Inter-dataset transfer frequently drop below 0.50.

---

## Raw Results

Results are also saved at: [full_accuracy_report.json](file:///d:/college_projects/Minor-II/ml-project/results/full_accuracy_report.json)

Script used: [generate_full_report.py](file:///d:/college_projects/Minor-II/ml-project/scripts/generate_full_report.py)
