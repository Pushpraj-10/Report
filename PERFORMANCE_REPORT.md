# Model Accuracy Report — CNN + BiLSTM vs CNN + SimpleKAN

> [!NOTE]
> Both models were **trained on DAIC-WOZ (Balanced)** and evaluated on all datasets.
> **Intra** = trained & tested on same dataset | **Inter** = tested on unseen dataset (cross-domain generalization).

---

## Accuracy (%)

| Architecture ↓ / Dataset → | DAIC-WOZ (Intra) | Dataset-Depression (Inter) | EATD-Corpus (Inter) |
|:----------------------------|:-----------------:|:--------------------------:|:-------------------:|
| **CNN + BiLSTM**            | 64.02%            | 46.21%                     | 62.99%              |
| **CNN + SimpleKAN**         | **82.81%**        | **50.88%**                 | 56.96%              |

---

## Macro F1 Score

| Architecture ↓ / Dataset → | DAIC-WOZ (Intra) | Dataset-Depression (Inter) | EATD-Corpus (Inter) |
|:----------------------------|:-----------------:|:--------------------------:|:-------------------:|
| **CNN + BiLSTM**            | 0.4999            | **0.4463**                 | **0.4502**          |
| **CNN + SimpleKAN**         | **0.5274**        | 0.3589                     | 0.4104              |

---

## Sample Counts Per Dataset

| Dataset              | Type  | Samples Evaluated |
|:---------------------|:-----:|:-----------------:|
| DAIC-WOZ (Balanced)  | Intra | 2,496             |
| Dataset-Depression   | Inter | 792               |
| EATD-Corpus          | Inter | 481               |

---

## Key Observations

### Intra-Dataset (DAIC-WOZ)
- **CNN + SimpleKAN** significantly outperforms CNN + BiLSTM on the training domain (82.81% vs 64.02% accuracy).
- KAN's higher intra-dataset accuracy suggests it learned the DAIC-WOZ feature distribution more aggressively.

### Inter-Dataset (Cross-Domain Generalization)
- **CNN + BiLSTM** generalizes better to unseen datasets, particularly on EATD-Corpus (62.99% vs 56.96%).
- On Dataset-Depression, both models struggle (below 51%), indicating significant domain shift from DAIC-WOZ.
- CNN + BiLSTM's temporal modeling (BiLSTM) appears to capture more generalizable speech patterns.

### F1 Scores
- Both models show **low Macro F1 scores** (≤ 0.53), suggesting class imbalance sensitivity — models tend to favor the majority class.
- CNN + SimpleKAN has the highest F1 on the intra-dataset (0.5274) but degrades more on inter-datasets.

> [!WARNING]
> The low F1 scores across all evaluations indicate that while accuracy may look reasonable, the models are not effectively distinguishing between Normal and Depressed classes in a balanced manner. Further training improvements (data augmentation, longer training, domain adaptation) are recommended.

---

## Raw Results

Results are also saved at: [accuracy_report.json](file:///d:/college_projects/Minor-II/ml-project/results/accuracy_report.json)

Script used: [generate_accuracy_report.py](file:///d:/college_projects/Minor-II/ml-project/scripts/generate_accuracy_report.py)
