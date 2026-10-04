# QML-Guard: Adversarial Robustness of a Variational Quantum Classifier

## Project Status

**Status:** Initial research study completed  
**Date:** October 2026  
**Framework:** PennyLane 0.45.1 + PyTorch  
**Execution:** Classical quantum simulation in Google Colab  
**Hardware:** No real quantum hardware used

---

## 1. Research Question

> **How vulnerable is a variational quantum classifier (VQC) to adversarial perturbations, and can adversarial training improve its robustness?**

The project investigates adversarial robustness in a small quantum machine learning classifier and compares its behaviour with a classical baseline.

The goal is **not** to claim quantum advantage. Instead, the study focuses on the security and robustness behaviour of a VQC under adversarial input perturbations.

---

## 2. Research Pipeline

The project follows:

**Build → Attack → Measure → Defend → Attack Again → Compare**

1. Prepare the dataset.
2. Train a classical baseline.
3. Build and train a 4-qubit VQC.
4. Measure clean performance.
5. Apply an FGSM adversarial attack.
6. Measure robustness.
7. Train a second VQC using adversarial examples.
8. Re-attack the defended model.
9. Compare the original and defended models.
10. Compare the VQC with the classical baseline.

---

## 3. Dataset

### Breast Cancer Wisconsin Dataset

The project uses the `sklearn` Breast Cancer Wisconsin dataset.

- Samples: **569**
- Original features: **30**
- Classes: **2**
  - Benign
  - Malignant

To keep the quantum circuit small, four features were selected:

1. Mean radius
2. Mean perimeter
3. Mean area
4. Mean smoothness

### Preprocessing

- Train/test split: **80/20**
- `random_state = 42`
- Stratified split
- `StandardScaler` fitted only on the training set
- The same scaler was used to transform the test set

The standardized feature space is also the space in which the FGSM perturbations were applied.

---

## 4. Classical Baseline

A Logistic Regression classifier was trained using the same four standardized features.

### Results

| Metric | Result |
|---|---:|
| Accuracy | **88.60%** |
| Malignant recall | **88%** |
| Benign recall | **89%** |
| Macro F1 | **88%** |

Confusion matrix:

```text
                 Predicted
              Malignant  Benign
Actual
Malignant         37        5
Benign             8       64
```

The classical baseline substantially outperformed the original VQC on clean test data.

---

## 5. Variational Quantum Classifier

### Architecture

- **4 qubits**
- **2 variational layers**
- **16 trainable parameters**
- `RY` feature encoding
- Trainable `RY` and `RZ` rotations
- CNOT entanglement chain
- Measurement: expectation value of Pauli-Z on qubit 0
- Output mapped to a probability using:

```text
probability = (expectation + 1) / 2
```

### Training

- Loss: Binary Cross Entropy
- Optimizer: Adam
- Learning rate: `0.01`
- Epochs: `50`

PennyLane provides automatic differentiation and PyTorch integration for hybrid quantum-classical models, which was used for training this VQC.

---

## 6. Original VQC: Clean Performance

The original VQC achieved:

| Metric | Result |
|---|---:|
| Accuracy | **71.05%** |
| Malignant recall | **29%** |
| Benign recall | **96%** |

Confusion matrix:

```text
                 Predicted
              Malignant  Benign
Actual
Malignant         12       30
Benign             3       69
```

### Observation

The model showed a strong bias toward predicting the benign class.

This is important because overall accuracy alone hides the weakness in malignant-class detection.

---

## 7. FGSM Adversarial Attack

The Fast Gradient Sign Method (FGSM) was used to perturb the input features.

The perturbation follows:

```text
x_adv = x + epsilon * sign(gradient)
```

The perturbations were applied in standardized feature space.

### Epsilon Sweep

| Epsilon | Original VQC Accuracy |
|---:|---:|
| 0.00 | 71.05% |
| 0.01 | 71.05% |
| 0.03 | 70.18% |
| 0.05 | 68.42% |
| 0.10 | 66.67% |
| 0.15 | 65.79% |

At `epsilon = 0.05`:

- Clean accuracy: **71.05%**
- Attacked accuracy: **68.42%**
- Accuracy drop: **2.63 percentage points**

The attack therefore caused measurable degradation, although the observed degradation was modest at this epsilon.

---

## 8. Adversarial Training

A fresh VQC with the same architecture was trained using FGSM adversarial examples generated with:

```text
epsilon_train = 0.05
```

Training was performed for 50 epochs.

### Training Loss

| Epoch | Loss |
|---:|---:|
| 5 | 0.8751 |
| 10 | 0.8144 |
| 15 | 0.7581 |
| 20 | 0.7066 |
| 25 | 0.6601 |
| 30 | 0.6185 |
| 35 | 0.5815 |
| 40 | 0.5491 |
| 45 | 0.5212 |
| 50 | 0.4978 |

---

## 9. Defended VQC: Clean Performance

After adversarial training:

| Metric | Original VQC | Defended VQC | Change |
|---|---:|---:|---:|
| Accuracy | 71.05% | **83.33%** | **+12.28 pp** |
| Malignant recall | 29% | **55%** | **+26 pp** |
| Benign recall | 96% | **100%** | **+4 pp** |

Confusion matrix:

```text
                 Predicted
              Malignant  Benign
Actual
Malignant         23       19
Benign             0       72
```

The defended model performed substantially better than the original VQC on the clean test set.

However, it still did not match the classical Logistic Regression baseline of 88.60% accuracy and 88% malignant recall.

---

## 10. Re-Attacking the Defended Model

The adversarially trained VQC was tested against the same FGSM attack.

### Epsilon Sweep

| Epsilon | Original VQC | Defended VQC |
|---:|---:|---:|
| 0.00 | 71.05% | **83.33%** |
| 0.01 | 71.05% | **82.46%** |
| 0.03 | 70.18% | **81.58%** |
| 0.05 | 68.42% | **80.70%** |
| 0.10 | 66.67% | **78.07%** |
| 0.15 | 65.79% | **72.81%** |

### At epsilon = 0.05

The defended model improved attacked accuracy from:

**68.42% → 80.70%**

Improvement:

**+12.28 percentage points**

### At epsilon = 0.15

The defended model achieved:

**72.81%**

compared with:

**65.79%**

for the original VQC.

Improvement:

**+7.02 percentage points**

---

## 11. Attack Success Rate

At `epsilon = 0.05`:

| Model | Attack Success Rate |
|---|---:|
| Original VQC | **3.70%** |
| Defended VQC | **3.16%** |

The attack success rate was low for both models.

This metric should **not** be treated as the main evidence of robustness because the original VQC already had limited clean performance and a strong benign-class bias.

The accuracy and class-specific metrics provide more useful evidence for this experiment.

---

## 12. Main Findings

### Finding 1: The classical baseline performed better

The Logistic Regression baseline achieved **88.60%** accuracy, while the original VQC achieved **71.05%**.

Therefore, this experiment provides **no evidence of quantum advantage**.

That was not the objective of the project.

---

### Finding 2: The original VQC was strongly biased toward benign predictions

The original VQC achieved only **29% malignant recall** despite achieving 71.05% overall accuracy.

This demonstrates why class-specific metrics are important for evaluating the model.

---

### Finding 3: FGSM degraded VQC performance

Increasing the perturbation strength generally reduced test accuracy.

For example:

```text
epsilon = 0.00 → 71.05%
epsilon = 0.05 → 68.42%
epsilon = 0.15 → 65.79%
```

---

### Finding 4: Adversarial training improved the VQC

The adversarially trained model achieved:

```text
Clean accuracy:
71.05% → 83.33%

Malignant recall:
29% → 55%
```

At `epsilon = 0.05`:

```text
Attacked accuracy:
68.42% → 80.70%
```

This provides evidence that adversarial training improved performance against the FGSM attack used in this experiment.

---

### Finding 5: Robustness was not absolute

The defended model still degraded as epsilon increased:

```text
83.33% → 72.81%
```

from clean evaluation to `epsilon = 0.15`.

Therefore, the correct conclusion is:

> **Adversarial training improved robustness against the evaluated FGSM attack, but it did not make the VQC robust to stronger perturbations.**

---

## 13. Current Research Conclusion

This study provides a small experimental demonstration of adversarial robustness in a variational quantum classifier.

The main result is that **FGSM adversarial perturbations can degrade VQC performance, while adversarial training can improve both clean performance and performance under the evaluated attack.**

However, the results are preliminary.

The study does **not** establish:

- quantum advantage
- general robustness against all adversarial attacks
- robustness on real quantum hardware
- medical deployment readiness
- superiority of VQCs over classical ML
- generalization across datasets

The work should therefore be presented as an **initial experimental study of adversarial robustness in a small simulated VQC**.

---

## 14. Limitations

1. Only one dataset was used.
2. Only four input features were used.
3. Only one VQC architecture was evaluated.
4. Only FGSM was used as the adversarial attack.
5. One main random seed was used.
6. The experiment used a quantum simulator rather than real quantum hardware.
7. The adversarial training used the same FGSM attack family that was used for evaluation.
8. The model has a noticeable class imbalance in its predictions.
9. The dataset is a standard machine-learning benchmark and should not be interpreted as a medical deployment study.

---

## 15. Recommended Future Work

The project is currently at a good stopping point for a first experimental version.

If the work is later developed into a paper, the next improvements should be:

### A. Multiple random seeds

Repeat the full experiment with several seeds and report:

- mean accuracy
- standard deviation
- malignant recall
- macro F1
- robust accuracy

### B. Stronger attacks

Add attacks such as:

- PGD
- multi-step gradient attacks
- potentially SPSA or other gradient-free attacks

This would test whether the adversarially trained model is robust only to FGSM or more broadly.

### C. Mixed clean + adversarial training

Compare pure adversarial training with a mixture of clean and adversarial samples.

### D. Better class-sensitive evaluation

Report:

- balanced accuracy
- macro F1
- malignant recall
- confusion matrices
- ROC-AUC where appropriate

### E. Architecture comparison

Compare several small VQC configurations, for example:

- 4 qubits / 1 layer
- 4 qubits / 2 layers
- 4 qubits / 3 layers

### F. Classical-vs-quantum robustness comparison

Apply the same attack methodology to the Logistic Regression baseline and compare robustness, rather than only comparing clean accuracy.

This would make the security comparison substantially stronger.

### G. Reproducibility

Record:

- package versions
- random seeds
- model initialization
- training configuration
- dataset preprocessing
- attack parameters

---

## 16. Research Positioning

A careful description of the project is:

> **QML-Guard investigates the adversarial robustness of a small variational quantum classifier using FGSM attacks and adversarial training. Using the Breast Cancer Wisconsin benchmark, the study evaluates clean performance, attack-induced degradation, and robustness improvement after adversarial training, with a classical Logistic Regression model used as a baseline.**

The project should be positioned around:

**Quantum Machine Learning + Adversarial Machine Learning + Trustworthy AI + Quantum Security**

rather than around claims of quantum advantage.

---

## 17. Technology Stack

- Python
- PennyLane 0.45.1
- PyTorch
- scikit-learn
- NumPy
- Pandas
- Matplotlib
- Google Colab

PennyLane supports differentiable quantum circuits and integration with PyTorch, making it suitable for hybrid quantum-classical machine learning workflows.

---

## 18. Reproducibility Notes

The main environment used during the experiment included:

```text
PennyLane 0.45.1
PyTorch 2.11.0+cpu
scikit-learn 1.6.1
NumPy 2.1.3
```

Core installation:

```bash
pip install pennylane torch scikit-learn matplotlib pandas
```

The experiment was run in Google Colab.


---

## 19. Final Snapshot

```text
                 ORIGINAL       DEFENDED
Clean accuracy   71.05%         83.33%
FGSM ε=0.05      68.42%         80.70%
FGSM ε=0.15      65.79%         72.81%
Malignant recall 29%            55%
```

### Bottom line

**The VQC was vulnerable to increasing FGSM perturbation strength, and adversarial training improved its performance under the evaluated attack. However, the defended model remained imperfect and did not outperform the classical baseline.**

This is a useful first research result, but stronger claims require additional attacks, repeated experiments, stronger baselines, and broader evaluation.
