# CMPE255 Final Project: Network Intrusion Detection Using Data Mining Techniques

**Course:** CMPE255 - Data Mining  
**Dataset:** UNSW-NB15 Network Intrusion Detection Dataset  
**Objective:** Detect network traffic anomalies (attacks) using unsupervised learning and data mining techniques

---

## Table of Contents

1. [Introduction](#introduction)
2. [Dataset Overview](#dataset-overview)
3. [Data Preprocessing](#data-preprocessing)
4. [Exploratory Data Analysis](#exploratory-data-analysis)
5. [Dimensionality Reduction (PCA)](#dimensionality-reduction-pca)
6. [Feature Importance Analysis](#feature-importance-analysis)
7. [Anomaly Detection Methods](#anomaly-detection-methods)
8. [PCA Impact Analysis](#pca-impact-analysis)
9. [Clustering Analysis](#clustering-analysis)
10. [Data Balancing Techniques](#data-balancing-techniques)
11. [Ensemble Methods](#ensemble-methods)
12. [Results Summary](#results-summary)
13. [Conclusions](#conclusions)

---

## Introduction

Network intrusion detection is a critical cybersecurity challenge. Traditional signature-based systems fail to detect novel attacks because they rely on known attack patterns. This project explores **unsupervised anomaly detection**, which learns patterns of "normal" network traffic and flags deviations as potential attacks. The key advantage of this approach is that it can detect zero-day attacks that have never been seen before, since it doesn't require labeled examples of each attack type.

We train models exclusively on normal traffic data, allowing them to learn what "normal" looks like. During inference, any traffic that deviates significantly from normal patterns is flagged as a potential attack. This unsupervised approach is particularly valuable in cybersecurity where new attack vectors emerge constantly.

---

## Dataset Overview

### UNSW-NB15 Dataset

The UNSW-NB15 dataset is a comprehensive network intrusion detection benchmark containing modern attack types including Fuzzers, Analysis, Backdoors, DoS, Exploits, Generic, Reconnaissance, Shellcode, and Worms.

**Data Loading Results:**

```
Total records loaded: 2,540,047
├── UNSW-NB15_1.csv: 700,001 records
├── UNSW-NB15_2.csv: 700,001 records
├── UNSW-NB15_3.csv: 700,001 records
└── UNSW-NB15_4.csv: 440,044 records

Sampled for analysis: 50,000 records
├── Normal samples: 43,720 (87.4%)
└── Attack samples: 6,280 (12.6%)
```

We sampled 50,000 records from the full dataset to enable faster experimentation while maintaining statistical significance. The sample preserves the original class distribution.

### Data Split Strategy

For unsupervised anomaly detection, the models are trained on **normal data only**. This is intentional—by learning only what normal traffic looks like, the model treats anything that doesn't fit this learned distribution as anomalous. This mirrors real-world scenarios where we have abundant examples of normal traffic but attacks are rare and varied.

```
Training set (NORMAL ONLY): 27,980 samples
Validation set: 6,996 samples
Test set (Normal + Attack): 15,024 samples
├── Normal in test: 8,744 (58.2%)
└── Attack in test: 6,280 (41.8%)
```

---

## Data Preprocessing

### Feature Types Identified

- **Numerical Features:** 39
- **Categorical Features:** 8

### Hybrid Categorical Encoding Strategy

A critical data mining decision was choosing the appropriate encoding for categorical features. We use a **hybrid approach** because different features have different characteristics:

| Encoding Type          | Features                                  | Rationale                                                                 |
| ---------------------- | ----------------------------------------- | ------------------------------------------------------------------------- |
| **One-Hot Encoding**   | `proto`, `state`, `service`, `ct_ftp_cmd` | Low cardinality (≤15 unique values), preserves category independence      |
| **Frequency Encoding** | `srcip`, `sport`, `dstip`, `dsport`       | High cardinality (IP addresses, ports), prevents dimensionality explosion |

**Why not One-Hot Encoding for everything?** Features like IP addresses (`srcip`, `dstip`) and ports (`sport`, `dsport`) have thousands of unique values. One-hot encoding these would create thousands of sparse binary columns, causing the "curse of dimensionality"—the model would struggle to find meaningful patterns in such a high-dimensional sparse space, and training would become computationally expensive. Frequency encoding instead captures how common each value is (e.g., frequently accessed IPs might indicate different behavior than rare ones) while adding only one column per feature.

**Why One-Hot for low-cardinality features?** For features like `proto` (protocol: TCP, UDP, etc.) with few unique values, one-hot encoding is ideal because it treats each category as independent without implying any ordinal relationship. Label encoding (0, 1, 2...) would incorrectly suggest that UDP is "between" TCP and ICMP, which is meaningless.

**Results:**

- One-hot encoded: 4 features → 32 columns
- Frequency encoded: 4 features → 4 columns
- **Total features after encoding: 75**

### Data Cleaning

We applied **outlier clipping** using the 1st-99th percentile range rather than removing outliers entirely. In network traffic data, extreme values might be legitimate (e.g., a large file transfer) or indicative of attacks. Clipping preserves these data points while preventing extreme values from dominating the scaling process.

```
Missing values handled: 69,408 (median/mode imputation)
Outliers clipped (1st-99th percentile): 12,203 values
```

Median imputation was chosen over mean imputation for numerical features because median is robust to outliers—network traffic data often contains extreme values that would skew the mean.

---

## Exploratory Data Analysis

### Correlation Analysis

Correlation analysis was performed to identify redundant features. Highly correlated features provide the same information, so one could potentially be removed to reduce dimensionality and prevent multicollinearity issues in some algorithms.

![Correlation Heatmap](output/correlation.png)

**Highly Correlated Feature Pairs (|r| > 0.8):**

| Feature 1   | Feature 2        | Correlation |
| ----------- | ---------------- | ----------- |
| trans_depth | ct_flw_http_mthd | 1.000       |
| Stime       | Ltime            | 1.000       |
| swin        | dwin             | 0.997       |
| tcprtt      | synack           | 0.994       |
| dloss       | Dpkts            | 0.992       |
| dbytes      | dloss            | 0.990       |
| tcprtt      | ackdat           | 0.989       |
| synack      | ackdat           | 0.973       |
| dbytes      | Dpkts            | 0.965       |
| Spkts       | Dpkts            | 0.965       |

**Finding:** 39 highly correlated feature pairs were identified. For example, `Stime` (start time) and `Ltime` (last time) have perfect correlation of 1.0, meaning they provide identical information. Similarly, TCP round-trip time metrics (`tcprtt`, `synack`, `ackdat`) are highly correlated as they all measure aspects of connection timing. This redundancy suggests that dimensionality reduction techniques like PCA could be effective.

---

## Dimensionality Reduction (PCA)

Principal Component Analysis (PCA) was applied to understand the intrinsic dimensionality of the data. PCA transforms correlated features into a smaller set of uncorrelated components that capture most of the variance in the data.

**Why apply PCA?** The correlation analysis revealed significant redundancy in the features. PCA can compress this information into fewer dimensions, which is particularly beneficial for distance-based algorithms like One-Class SVM and LOF. These algorithms compute distances between points, and in high-dimensional spaces, all points tend to become equidistant (the "curse of dimensionality"), making it difficult to distinguish normal from anomalous points.

![PCA Analysis](output/pca.png)

### PCA Results

| Metric                      | Value          |
| --------------------------- | -------------- |
| Total features              | 75             |
| Components for 95% variance | **11**         |
| Components for 99% variance | 21             |
| First 2 components          | 77.5% variance |

**Key Insight:** The 75 features can be reduced to just 11 components while retaining 95% of the information. This dramatic reduction (from 75 to 11) confirms the high redundancy identified in the correlation analysis. The fact that just 2 components capture 77.5% of variance indicates that the data has strong underlying structure that PCA can exploit.

---

## Feature Importance Analysis

Two complementary methods were used to identify the most predictive features. Understanding feature importance helps interpret model decisions and could guide future feature engineering efforts.

**Random Forest Importance** measures how much each feature contributes to reducing prediction error across all trees in the forest. Features that frequently appear in decision splits with high information gain receive higher importance scores.

**Mutual Information** measures the statistical dependency between each feature and the target variable. Unlike correlation, it can capture non-linear relationships—a feature might have low correlation but high mutual information if it has a complex but predictable relationship with the target.

![Feature Importance](output/featureimportance.png)

### Top 10 Features by Random Forest Importance

| Rank | Feature    | Importance |
| ---- | ---------- | ---------- |
| 1    | feature_3  | 0.1496     |
| 2    | feature_29 | 0.1445     |
| 3    | feature_73 | 0.1220     |
| 4    | feature_71 | 0.1209     |
| 5    | feature_8  | 0.0686     |
| 6    | feature_2  | 0.0476     |
| 7    | feature_24 | 0.0450     |
| 8    | feature_15 | 0.0371     |
| 9    | feature_4  | 0.0337     |
| 10   | feature_1  | 0.0334     |

### Top 10 Features by Mutual Information

| Rank | Feature    | MI Score |
| ---- | ---------- | -------- |
| 1    | feature_73 | 0.6414   |
| 2    | feature_71 | 0.6348   |
| 3    | feature_3  | 0.6329   |
| 4    | feature_29 | 0.6316   |
| 5    | feature_1  | 0.5737   |
| 6    | feature_7  | 0.5091   |
| 7    | feature_15 | 0.4988   |
| 8    | feature_4  | 0.4748   |
| 9    | feature_2  | 0.4342   |
| 10   | feature_8  | 0.4017   |

**Observation:** Both methods agree on the top features (feature_3, feature_29, feature_71, feature_73), which validates their importance. When two independent methods identify the same features as important, we can be more confident these features genuinely carry predictive signal rather than being artifacts of a particular algorithm.

---

## Anomaly Detection Methods

Five unsupervised anomaly detection methods were trained and compared. Each method takes a different approach to defining "normal" and detecting deviations.

### Methods Overview

| Method                  | Category      | Description                                                                                |
| ----------------------- | ------------- | ------------------------------------------------------------------------------------------ |
| **Autoencoder**         | Deep Learning | Neural network learns to reconstruct normal data; anomalies have high reconstruction error |
| **Isolation Forest**    | Tree-based    | Randomly partitions features; anomalies are easier to isolate (require fewer splits)       |
| **One-Class SVM**       | Kernel-based  | Learns a boundary around normal data using RBF kernel                                      |
| **One-Class SVM (PCA)** | Kernel + PCA  | Same as above but with PCA-reduced features (tests impact of dimensionality reduction)     |
| **LOF**                 | Density-based | Compares local density with neighbors; anomalies have lower density than their neighbors   |

**Why include One-Class SVM both with and without PCA?** This allows us to directly measure the impact of dimensionality reduction on a distance-based algorithm. One-Class SVM uses an RBF kernel that computes distances between points, making it susceptible to the curse of dimensionality. By comparing both versions, we can quantify whether PCA helps or hurts performance.

**Why not apply PCA to Autoencoder?** Autoencoders inherently perform dimensionality reduction—the bottleneck layer (encoding dimension = 32) forces the network to learn a compressed representation. Applying PCA beforehand would be redundant and might remove information that the autoencoder could have learned to use. The autoencoder essentially learns its own optimal "PCA-like" transformation that's tailored to reconstruction.

### Results Comparison

![ROC Curves](output/roccurvers.png)

| Method              | Accuracy   | Precision  | Recall     | F1-Score   | ROC-AUC    | Training Time |
| ------------------- | ---------- | ---------- | ---------- | ---------- | ---------- | ------------- |
| **Autoencoder**     | **0.9679** | **0.9297** | 0.9986     | **0.9629** | 0.9873     | 28.73s        |
| Isolation Forest    | 0.9426     | 0.8797     | 0.9992     | 0.9357     | 0.9709     | **0.78s**     |
| One-Class SVM       | 0.9408     | 0.8759     | **1.0000** | 0.9338     | 0.9915     | 29.30s        |
| One-Class SVM (PCA) | 0.9400     | 0.8746     | 0.9997     | 0.9330     | **0.9920** | 10.01s        |
| LOF                 | 0.9235     | 0.8550     | 0.9839     | 0.9149     | 0.9801     | 6.74s         |

### Confusion Matrices & Score Distributions

![Confusion Matrices](output/confusionmatrices_scoredistributions.png)

### Key Findings:

1. **Autoencoder achieves the best F1-Score (96.29%)** — The deep learning approach effectively learns the complex, non-linear patterns of normal traffic that simpler methods miss.

2. **One-Class SVM (PCA) achieves the best ROC-AUC (99.20%)** — Dimensionality reduction actually improved the kernel-based method by removing noise and redundant features.

3. **Isolation Forest is fastest (0.78s)** — Nearly 37x faster than Autoencoder while maintaining good performance, making it ideal for real-time applications.

4. **All methods achieve >98% recall** — This is critical for security applications where missing attacks (false negatives) is more costly than false alarms. A missed attack could lead to a breach, while a false alarm only requires investigation.

---

## PCA Impact Analysis

A dedicated comparison was conducted to evaluate the impact of PCA on One-Class SVM. This analysis demonstrates a key data mining principle: preprocessing choices can significantly affect model performance.

![PCA Impact](output/pcaimpact.png)

### One-Class SVM: Full Features vs PCA

| Metric        | Full Features (75) | PCA (16)   | Difference      |
| ------------- | ------------------ | ---------- | --------------- |
| Features      | 75                 | 16         | -59             |
| Accuracy      | 0.9408             | 0.9400     | -0.08%          |
| Precision     | 0.8759             | 0.8746     | -0.13%          |
| Recall        | 1.0000             | 0.9997     | -0.03%          |
| F1-Score      | 0.9338             | 0.9330     | -0.09%          |
| ROC-AUC       | 0.9915             | **0.9920** | **+0.05%**      |
| Training Time | 29.30s             | 10.01s     | **2.9x faster** |

### Conclusion on PCA:

**PCA is BENEFICIAL for One-Class SVM:**

- Reduced features from 75 to 16 (79% reduction)
- Training time improved by **2.9x** (29.30s → 10.01s)
- ROC-AUC actually **improved** by 0.05%
- Minimal impact on other metrics (<0.15% difference)

**Why does PCA help SVM?** The RBF kernel in One-Class SVM computes similarity based on Euclidean distance. In high-dimensional spaces, distances become less meaningful—all points appear roughly equidistant. By reducing to 16 dimensions that capture 95% of variance, we remove noise and redundant features, making the distance calculations more meaningful. The slight ROC-AUC improvement suggests that some of the removed dimensions were actually noise that was hurting the model.

---

## Clustering Analysis

Unsupervised clustering methods were evaluated for their ability to separate normal and attack traffic without using labels. The goal was to see if natural clusters in the data correspond to normal vs. attack traffic.

![Clustering Analysis](output/clustering.png)

### Clustering Results

| Method        | Accuracy | F1-Score | Notes                                 |
| ------------- | -------- | -------- | ------------------------------------- |
| K-Means (k=2) | 0.0815   | 0.1447   | Poor separation                       |
| DBSCAN        | 0.6122   | 0.1904   | 29 clusters, 917 noise points         |
| GMM           | 0.0107   | 0.0000   | AUC: 0.9909 (good probability scores) |

### Analysis:

Clustering methods performed poorly for direct classification because anomaly detection is fundamentally different from clustering:

1. **Attack traffic is not a single cluster** — Attacks are diverse (DoS, exploits, backdoors, etc.) and may appear in different regions of feature space
2. **Normal traffic dominates** — With 87% normal samples, clustering algorithms tend to find subclusters within normal traffic rather than separating normal from attack
3. **Anomalies are defined by what they're NOT** — Anomaly detection identifies points that don't fit the normal pattern, while clustering tries to group similar points together

However, **GMM's anomaly probability scores achieved 0.9909 AUC**, indicating that the probability of belonging to the dominant cluster is a useful anomaly score—points with low probability of being "normal" are likely anomalies.

---

## Data Balancing Techniques

To address class imbalance (87% normal, 13% attack), three data balancing techniques were evaluated. Class imbalance can cause models to be biased toward the majority class.

![Data Balancing](output/databalancing.png)

**Why address class imbalance?** When one class dominates, a model can achieve high accuracy by simply predicting the majority class. For intrusion detection, this would mean missing most attacks. Balancing techniques help the model pay equal attention to both classes.

### Original Distribution

- Normal (0): 8,744
- Attack (1): 6,280

### Balancing Results (with Random Forest classifier)

| Technique         | Class Distribution | Accuracy | F1-Score |
| ----------------- | ------------------ | -------- | -------- |
| **SMOTE**         | 6,120 / 6,120      | 0.9933   | 0.9921   |
| **ADASYN**        | 6,120 / 6,216      | 0.9933   | 0.9921   |
| **Undersampling** | 4,396 / 4,396      | 0.9933   | 0.9921   |

**SMOTE** (Synthetic Minority Over-sampling Technique) creates synthetic examples by interpolating between existing minority class samples. **ADASYN** is adaptive SMOTE that focuses on harder-to-learn regions. **Undersampling** simply removes majority class examples.

All three techniques achieved identical performance (99.33% accuracy), suggesting that for this dataset with a Random Forest classifier, the original imbalance (87%/13%) was not severe enough to significantly hurt performance. The anomaly detection methods used earlier are inherently robust to imbalance since they only train on normal data.

---

## Ensemble Methods

Four ensemble strategies were evaluated to combine the strengths of individual anomaly detectors. Ensemble methods often outperform individual models by reducing variance and capturing different aspects of the data.

![Ensemble Results](output/ensemble.png)

**Why use ensembles?** Different anomaly detection methods have different strengths—Autoencoder captures complex non-linear patterns, Isolation Forest is fast and handles high dimensions well, One-Class SVM finds optimal boundaries, and LOF captures local density variations. Combining them can leverage all these strengths while mitigating individual weaknesses.

### Methods Combined

Autoencoder, Isolation Forest, One-Class SVM, and LOF were combined. One-Class SVM (PCA) was excluded to avoid having two variants of the same algorithm, which would bias the ensemble toward SVM-like behavior.

### Ensemble Results

| Ensemble Method             | Accuracy   | F1-Score   | ROC-AUC    |
| --------------------------- | ---------- | ---------- | ---------- |
| Majority Voting (≥3/4)      | 0.9654     | 0.9602     | -          |
| Score Averaging             | 0.9708     | 0.9663     | 0.9870     |
| Weighted Voting             | 0.9708     | 0.9663     | 0.9871     |
| **Stacking (Meta-Learner)** | **0.9902** | **0.9885** | **0.9959** |

### Weights for Weighted Voting

Based on individual model AUC scores:

- Autoencoder: 0.251
- One-Class SVM: 0.252
- Isolation Forest: 0.247
- LOF: 0.249

The weights are nearly equal because all models performed similarly well, so weighting provided minimal benefit over simple averaging.

### Key Finding:

**Stacking ensemble achieves the best overall performance:**

- 98.85% F1-Score (vs. 96.29% for best individual model)
- 99.59% ROC-AUC (vs. 99.20% for best individual model)
- 2.56% improvement in F1-Score over the best single model

**Why does stacking work best?** Stacking uses a meta-learner (logistic regression) that learns the optimal way to combine base model predictions. Instead of fixed rules like voting or averaging, the meta-learner discovers patterns like "when Autoencoder and SVM disagree, trust Autoencoder more" or "LOF's confidence matters more in certain score ranges." This learned combination is more sophisticated than manual rules.

---

## Results Summary

### Model Comparison

![Model Comparison](output/modelcomparison.png)

### Final Rankings

| Rank | Method                | F1-Score | ROC-AUC | Recommended Use Case                            |
| ---- | --------------------- | -------- | ------- | ----------------------------------------------- |
| 1    | **Stacking Ensemble** | 0.9885   | 0.9959  | Best overall performance, offline batch         |
| 2    | Autoencoder           | 0.9629   | 0.9873  | When single model needed, good interpretability |
| 3    | One-Class SVM (PCA)   | 0.9330   | 0.9920  | Best ROC-AUC with efficient training            |
| 4    | Isolation Forest      | 0.9357   | 0.9709  | Real-time detection, fastest training           |
| 5    | One-Class SVM         | 0.9338   | 0.9915  | When PCA preprocessing not available            |
| 6    | LOF                   | 0.9149   | 0.9801  | When local context matters                      |

---

## Conclusions

### Summary of Findings

1. **Autoencoder provides the best single-model performance** for network intrusion detection, achieving 96.79% accuracy and 96.29% F1-score. Its ability to learn complex non-linear patterns through the encoding-decoding process makes it well-suited for capturing the diverse characteristics of normal network traffic.

2. **PCA significantly benefits distance-based algorithms** — reducing features from 75 to 16 improved One-Class SVM's training time by 2.9x while slightly improving ROC-AUC. This confirms that removing redundant features helps kernel-based methods by making distance calculations more meaningful.

3. **Ensemble methods outperform individual models** — Stacking achieved 98.85% F1-score, a 2.56% improvement over the best individual method. The meta-learner effectively combines the diverse perspectives of different anomaly detection approaches.

4. **All anomaly detection methods achieve >98% recall** — This high recall is critical for security applications where detecting attacks is more important than avoiding false alarms. Missing an attack could lead to a security breach.

5. **Hybrid categorical encoding prevents dimensionality explosion** — Using frequency encoding for high-cardinality features (IP addresses, ports) kept the feature space at 75 dimensions instead of potentially thousands, enabling effective model training.

### Future Work

1. **Real-time deployment** — Optimize Isolation Forest for streaming detection given its 0.78s training time
2. **Feature engineering** — Create domain-specific features based on the importance analysis
3. **Deep learning variants** — Explore variational autoencoders (VAE) for probabilistic anomaly scores and LSTM for temporal pattern detection
4. **Threshold optimization** — Develop adaptive thresholds that balance precision/recall based on operational requirements

---

## References

- UNSW-NB15 Dataset: [https://research.unsw.edu.au/projects/unsw-nb15-dataset](https://research.unsw.edu.au/projects/unsw-nb15-dataset)
- Scikit-learn Documentation: [https://scikit-learn.org/](https://scikit-learn.org/)
- TensorFlow/Keras Documentation: [https://www.tensorflow.org/](https://www.tensorflow.org/)
- Imbalanced-learn Documentation: [https://imbalanced-learn.org/](https://imbalanced-learn.org/)
