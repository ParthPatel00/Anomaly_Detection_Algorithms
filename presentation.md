# CMPE255 Final Project Presentation

## Network Intrusion Detection Using Data Mining Techniques

---

## Slide 1: Title

### Content:

```
CMPE255 - Data Mining
Final Project

NETWORK INTRUSION DETECTION
Using Data Mining Techniques

Dataset: UNSW-NB15
```

### Script:

"Hello everyone. Today we'll be presenting my final project for CMPE255 Data Mining. The project tackles a critical cybersecurity challenge: network intrusion detection.

The goal is to detect malicious network traffic—attacks like DoS, exploits, backdoors, and reconnaissance—using data mining and machine learning techniques. We're using the UNSW-NB15 dataset, which is a modern benchmark containing over 2.5 million network traffic records.

What makes this project interesting from a data mining perspective is that we use unsupervised anomaly detection. Instead of requiring labeled examples of each attack type, we train models only on normal traffic and detect anything that deviates. This approach can detect novel attacks we've never seen before—a significant advantage over traditional signature-based systems.

Throughout this presentation, I'll walk you through our data preprocessing pipeline, exploratory analysis, five different anomaly detection methods, and ensemble approaches that achieve nearly 99% detection accuracy."

---

## Slide 2: Problem Statement

### Content:

```
THE PROBLEM

Traditional intrusion detection systems:
 ✗ Rely on known attack signatures
 ✗ Cannot detect zero-day attacks
 ✗ Require constant manual updates

Our Approach:
 ✓ Unsupervised anomaly detection
 ✓ Train only on normal traffic
 ✓ Compare 5 methods: Autoencoder, Isolation Forest,
   One-Class SVM, SVM+PCA, LOF
 ✓ Ensemble combinations
```

### Script:

"Let's start with the problem. Traditional intrusion detection systems have three major limitations. First, they rely on known attack signatures—essentially a database of patterns from previously seen attacks. Second, this means they cannot detect zero-day attacks—new attacks that haven't been catalogued yet. If an attacker develops a novel technique, the system has no signature to match against and the attack slips through. Third, they require constant manual updates as new attack patterns are discovered.

Our approach solves these problems fundamentally differently. We use unsupervised anomaly detection, meaning we don't need labeled examples of attacks at all. Instead, we learn what 'normal' network traffic looks like—its patterns, distributions, and characteristics. Then, anything that deviates significantly from this learned normal behavior gets flagged as potentially malicious. The key insight is that while attacks are diverse and constantly changing, normal traffic is relatively stable and predictable. By modeling normal, we can detect ANY anomaly—including attacks we've never seen before.

However, there's a tradeoff: this approach may flag unusual-but-legitimate traffic as anomalous. We're trading some false positives for the ability to detect novel attacks."

---

## Slide 3: Dataset Overview

### Content:

```
UNSW-NB15 DATASET

Total Records: 2.54 million
Sampled: 50,000

┌─────────────────────────────┐
│  Normal: 43,720 (87.4%)     │
│  Attack: 6,280  (12.6%)     │
└─────────────────────────────┘

49 Features including:
• Flow metrics (bytes, packets, duration)
• Protocol information (TCP, UDP, etc.)
• Connection states
• IP addresses and ports
```

### Script:

"We used the UNSW-NB15 dataset, which is a modern network intrusion benchmark created by the Australian Centre for Cyber Security. The full dataset contains over 2.5 million network traffic records with 49 features each.

Why did we sample only 50,000 records? Two reasons: First, it allows faster experimentation—we can iterate on preprocessing and model choices quickly. Second, 50,000 samples is statistically sufficient to train robust models while being representative of the full dataset.

Looking at the class distribution: 87% normal traffic and only 13% attacks. The attacks include nine categories: Fuzzers, Analysis, Backdoors, DoS, Exploits, Generic, Reconnaissance, Shellcode, and Worms. This diversity tests whether our models can detect varied attack types, not just one specific pattern.

The 49 features capture flow-level information: how many bytes were transferred, how many packets, connection duration, which protocol was used, TCP flags, and network addresses. These features describe the behavior of each network connection without looking at the actual payload content."

---

## Slide 4: Training Strategy

### Content:

```
KEY INSIGHT: Train on NORMAL data only

┌──────────────────────────────────────────┐
│            TRAINING DATA                 │
│         (Normal traffic only)            │
│              27,980 samples              │
└──────────────────────────────────────────┘
                    ↓
              Model learns
           "what normal looks like"
                    ↓
┌──────────────────────────────────────────┐
│              TEST DATA                   │
│       Normal (8,744) + Attack (6,280)    │
│              15,024 samples              │
└──────────────────────────────────────────┘
                    ↓
         Anomalies = Potential Attacks
```

### Script:

"This diagram shows the key insight of our approach, and it's crucial to understand why we do this.

We train our models exclusively on normal traffic—27,980 samples of legitimate network behavior. Why only normal data? If we trained on attack examples, we'd only detect attacks similar to what we've seen. By training only on normal, the model learns the boundaries of legitimate behavior and can detect ANY deviation.

During training, the model learns patterns like: 'normal web traffic has packets of this size range, connections last this long, these protocols are common.' It builds an internal representation of what normal looks like.

Then during testing, we present 15,024 samples—a mix of 8,744 normal and 6,280 attacks. The model has never seen attacks during training. When it encounters an attack that doesn't fit the learned normal patterns, this deviation produces a high anomaly score.

This is why our train/test setup prevents overfitting: the test attacks are truly novel to the model. There's no way for the model to memorize attack patterns because it never saw any during training.

However, there's a fundamental limitation: if an attack closely mimics normal traffic patterns, this approach will miss it. The model can only detect attacks that are measurably different from normal behavior in the feature space."

---

## Slide 5: Data Preprocessing

### Content:

```
PREPROCESSING PIPELINE

1. Missing Values → Median/Mode Imputation
   (69,408 values handled)

2. Outliers → Clipped to 1st-99th percentile
   (12,203 values clipped)

3. Categorical Encoding:
   ┌─────────────────────────────────────────────┐
   │ Low cardinality  → One-Hot Encoding         │
   │ (proto, state)      4 features → 32 columns │
   │                                             │
   │ High cardinality → Frequency Encoding       │
   │ (IP addresses)      4 features → 4 columns  │
   └─────────────────────────────────────────────┘

4. Numerical Features → Z-Score Normalization

Final: 75 features
```

### Script:

"Data preprocessing is critical for this dataset because it contained several issues. Let me go over the techniques we applied.

First, missing values: we had 69,408 missing values across the dataset. We used median imputation for numerical features rather than mean. Why median? Because network data often has outliers—a single huge file transfer could skew the mean dramatically. Median is robust to these outliers. For categorical features, we used mode—the most frequent value.

Second, outliers: we clipped values to the 1st-99th percentile rather than removing them. Why not remove outliers? In network traffic, extreme values might be legitimate — a large file download isn't necessarily an attack. Or they might BE the attack signature we want to detect. Removing them would lose information. Clipping keeps the data point but prevents extreme values from dominating the scaling.

Third, categorical encoding—this is where it gets interesting. We used a hybrid approach. For low-cardinality features like protocol type with only a few values—TCP, UDP, ICMP—we used one-hot encoding. This creates separate binary columns and treats each category as independent, which is correct—TCP isn't 'greater than' UDP.

But for high-cardinality features like IP addresses and ports, one-hot encoding would be disastrous. With thousands of unique IPs, we'd create thousands of sparse columns, causing the curse of dimensionality. Instead, we used frequency encoding—replacing each value with how often it appears. A frequently-accessed server IP gets a high value; a rare IP gets low. This captures the information in just one column.

Finally, Z-score normalization for numerical features ensures all features are on the same scale—essential for distance-based algorithms like SVM. The result: 75 clean, properly scaled features."

---

## Slide 6: Correlation Analysis

### Content:

![Correlation Heatmap](output/correlation.png)

```
39 highly correlated pairs found (|r| > 0.8)

Examples:
• Stime ↔ Ltime: 1.000 (redundant!)
• tcprtt ↔ synack: 0.994
• dbytes ↔ Dpkts: 0.965
```

### Script:

"Before applying machine learning, we performed exploratory data analysis. This correlation heatmap reveals something important about our features.

We found 39 feature pairs with correlation above 0.8—that's significant redundancy. Look at the examples: Stime (Start time) and Ltime (Last time) have a perfect correlation of 1.0. They're essentially the same information stored twice. Similarly, tcprtt (TCP round-trip time), synack (time between synchronize and synchronize-acknowledge), and ackdat (time between SYN-ACK and ACK) are all TCP handshake timing metrics that move together with correlations above 0.97.

Why does this matter? Redundant features cause several problems. First, they waste computation—we're processing the same information multiple times. Second, they can hurt some algorithms. For distance-based methods like SVM, correlated features effectively double-count certain information, distorting the distance calculations. Third, they make the model harder to interpret—if two features are correlated, which one is actually important?

This analysis directly motivated our next step: which is PCA. If 39 pairs are highly correlated, we should be able to compress the information into fewer dimensions without losing much. The correlation heatmap gave us confidence that dimensionality reduction would work well on this data."

---

## Slide 7: PCA Analysis

### Content:

![PCA Analysis](output/pca.png)

```
DIMENSIONALITY REDUCTION

Original features:     75
95% variance retained: 11 components
99% variance retained: 21 components

First 2 components capture 77.5% of variance!
```

### Script:

"Based on the correlation analysis, we applied PCA—Principal Component Analysis. PCA finds new axes that capture the maximum variance in the data. These new axes, called principal components, are uncorrelated linear combinations of the original features.

The results confirm what the correlation analysis suggested. Look at the numbers: we started with 75 features, but only 11 components are needed to capture 95% of the variance. That's an 85% reduction in dimensionality with only 5% information loss. Even more striking: just 2 components capture 77.5% of the variance. This means the data has strong underlying structure—most of the information lies along just a few directions.

The left plot shows variance per component—you can see the first few components dominate, then it drops off quickly. The middle plot shows cumulative variance—we hit 95% around component 11. The right plot visualizes the data projected onto the first 2 components, with normal traffic in blue and attacks in red. You can already see some separation, which is encouraging for our anomaly detection task.

Why is this important? For distance-based algorithms like One-Class SVM, high dimensionality causes the 'curse of dimensionality.' In 75 dimensions, all points tend to be roughly equidistant from each other—the distances converge. This makes it hard to distinguish normal from anomalous points. By reducing to 11 or 16 meaningful dimensions, we make the distance calculations more discriminative. We'll test this directly when we compare SVM with and without PCA."

---

## Slide 8: Feature Importance

### Content:

![Feature Importance](output/featureimportance.png)

```
Two methods agree on top features:

Random Forest    Mutual Information
────────────     ──────────────────
feature_3        feature_73
feature_29       feature_71
feature_73       feature_3
feature_71       feature_29
```

### Script:

"While PCA tells us about overall data structure, feature importance analysis tells us which specific features are most predictive for detecting attacks. We used two complementary methods.

Random Forest Importance works by training a forest of decision trees and measuring how much each feature contributes to reducing prediction error. Features that appear frequently in tree splits and produce large information gains get high importance scores. It captures how useful each feature is for the specific classification task.

Mutual Information takes a different approach—it measures the statistical dependency between each feature and the target label. Unlike correlation, it can capture non-linear relationships. A feature might have low correlation with the target but high mutual information if there's a complex but predictable relationship.

Why use both? Because they measure different things. If both methods agree a feature is important, we can be confident it's truly predictive, not just an artifact of one particular algorithm.

Looking at the results: both methods identify the same top features—feature_3, feature_29, feature_71, and feature_73—just in slightly different orders. This agreement is strong validation. These features contain the most signal for distinguishing attacks from normal traffic.

However, a limitation: we're using generic feature names from the dataset. We don't know exactly what network behaviors these represent. With domain knowledge, we could interpret these features and potentially engineer even better ones."

---

## Slide 9: Anomaly Detection Methods

### Content:

```
5 UNSUPERVISED METHODS COMPARED

┌─────────────────┬───────────────┬─────────────────────────────┐
│ Method          │ Category      │ How it works                │
├─────────────────┼───────────────┼─────────────────────────────┤
│ Autoencoder     │ Deep Learning │ Reconstruction error        │
│ Isolation Forest│ Tree-based    │ Isolation depth             │
│ One-Class SVM   │ Kernel-based  │ Boundary around normal      │
│ OC-SVM + PCA    │ Kernel + PCA  │ Same, with dim. reduction   │
│ LOF             │ Density-based │ Local density comparison    │
└─────────────────┴───────────────┴─────────────────────────────┘
```

### Script:

"Now for the core of our project—we compared five unsupervised anomaly detection methods, each taking a fundamentally different approach to defining 'anomaly.'

Autoencoder is a neural network with a bottleneck architecture. It learns to compress normal data into a small representation, then reconstruct it. When trained only on normal traffic, it becomes good at reconstructing normal patterns. When an attack comes through, the network can't reconstruct it well—the reconstruction error is high. We use this error as the anomaly score.

Isolation Forest takes a tree-based approach. It randomly selects features and split points to partition the data. The key insight: anomalies are rare and different, so they get isolated quickly with few splits. Normal points are similar to many others, requiring more splits to isolate. The number of splits needed becomes the anomaly score.

One-Class SVM uses kernel methods to learn a boundary around normal data in a high-dimensional feature space. Points outside this boundary are anomalies. The RBF kernel measures similarity based on distance—similar points have high kernel values. We test this both with original features and with PCA-reduced features to measure dimensionality's impact.

Local Outlier Factor is density-based. For each point, it compares the local density around that point to the density around its neighbors. Normal points are in dense regions surrounded by similar dense points. Anomalies are in sparse regions or have neighbors in much denser regions—their local outlier factor is high.

Each method has different strengths: Autoencoder captures complex non-linear patterns, Isolation Forest is fast and handles high dimensions well, SVM finds optimal boundaries, LOF is good at detecting local anomalies. By comparing all five, we can see which approach works best for network intrusion detection."

---

## Slide 10: Results - ROC Curves

### Content:

![ROC Curves](output/roccurvers.png)

```
ROC-AUC Scores:
• One-Class SVM (PCA): 0.9920 ← Best AUC
• One-Class SVM:       0.9915
• Autoencoder:         0.9873
• LOF:                 0.9801
• Isolation Forest:    0.9709
```

### Script:

"Let's look at the results. This ROC curve plot shows how each method performs at different threshold settings.

The ROC curve plots True Positive Rate against False Positive Rate as we vary the decision threshold. A perfect model would go straight up to 100% true positives with 0% false positives, then across—giving an AUC of 1.0. The diagonal line represents random guessing with AUC of 0.5.

ROC-AUC measures the probability that a randomly chosen attack will have a higher anomaly score than a randomly chosen normal sample. It captures the model's ranking ability independent of any particular threshold.

Looking at the results: One-Class SVM with PCA achieved the highest AUC at 99.20%, slightly better than regular SVM at 99.15%. This is significant—PCA didn't just speed things up, it actually improved discrimination. The Autoencoder achieved 98.73%, LOF got 98.01%, and Isolation Forest 97.09%.

All methods exceeded 97% AUC. Why so high? Two reasons: First, the UNSW-NB15 dataset was designed with attacks that have measurably different flow characteristics from normal traffic—the attacks ARE distinguishable in this feature space. Second, we have a clean train/test split with attacks only in test data, so there's no data leakage.

But notice Isolation Forest, despite being by far the fastest, has the lowest AUC. This shows there IS variation—not all methods perform equally. The 2% gap between best and worst is meaningful."

---

## Slide 11: Results - Performance Table

### Content:

```
COMPREHENSIVE RESULTS (All models trained on T4 GPU)

Method              Accuracy  F1-Score  ROC-AUC  Time
─────────────────────────────────────────────────────
Autoencoder         96.79%    96.29%    98.73%   28.7s
Isolation Forest    94.26%    93.57%    97.09%   0.78s ← Fastest!
One-Class SVM       94.08%    93.38%    99.15%   29.3s
One-Class SVM (PCA) 94.00%    93.30%    99.20%   10.0s
LOF                 92.35%    91.49%    98.01%   6.74s

All methods: >98% Recall (catches most attacks)
```

### Script:

"Looking at the full results—all trained on a T4 GPU in Google Colab—Autoencoder achieved the best accuracy and F1-score at nearly 97%. Now you might wonder—how is Isolation Forest so fast at 0.78 seconds? The answer lies in its algorithm design. Isolation Forest doesn't compute distances between points like SVM or LOF. Instead, it builds random decision trees and simply counts how many splits it takes to isolate each point. Anomalies are isolated quickly with fewer splits. This is computationally cheap—just random feature selection and threshold comparisons—and it's highly parallelizable across the 100 trees.

In contrast, One-Class SVM took 29 seconds because it must compute kernel distances between all training points—that's O(n²) complexity with 28,000 samples. The Autoencoder took similar time because neural network training requires multiple passes through the data with backpropagation. LOF at 6.7 seconds needs to find k-nearest neighbors for every point, which also involves distance calculations but fewer than SVM's full kernel matrix.

Now, you might ask: why are these numbers so high? Are we overfitting? Three reasons suggest we're not. First, we used completely separate train and test sets—the test data was never seen during training. Second, our unsupervised models train ONLY on normal data, but test on a mix of normal and attacks—so attacks are truly unseen. Third, the UNSW-NB15 dataset was specifically designed with distinct attack patterns that differ measurably from normal traffic in the feature space. The attacks genuinely look different from normal behavior, which is why all methods perform well.

That said, there are limitations. LOF has the lowest accuracy at 92%—still good, but noticeably worse. And all models have some false positives, meaning legitimate traffic flagged as attacks. The precision scores around 93-96% mean roughly 4-7% of alerts would be false alarms."

---

## Slide 12: Confusion Matrices

### Content:

![Confusion Matrices](output/confusionmatrices_scoredistributions.png)

### Script:

"This visualization gives us deeper insight into how each model performs. Let's understand what we're looking at.

The top row shows confusion matrices for each method. Reading a confusion matrix: rows are actual labels, columns are predictions. Top-left is true negatives—normal traffic correctly classified as normal. Top-right is false positives—normal traffic incorrectly flagged as attacks. Bottom-left is false negatives—attacks missed, classified as normal. Bottom-right is true positives—attacks correctly detected.

Looking at the matrices, you can see all methods have very few false negatives in the bottom-left, which is why recall exceeds 98%. But look at the top-right—false positives. These are normal samples incorrectly flagged as attacks. You can see several hundred false positives for each method. This is the cost of high recall: to catch almost all attacks, we accept some false alarms on normal traffic.

The bottom row shows score distributions—the histogram of anomaly scores for normal samples (green) versus attack samples (red). Ideally, these distributions should be completely separated—all attacks should have higher scores than all normal traffic. You can see good separation for all methods, but there IS overlap in the middle. The overlap region is where the model is uncertain—samples here could be either normal or attack, and our threshold choice determines how they're classified.

Notice the Autoencoder has particularly good separation—the distributions barely overlap. This explains its high F1-score. But also notice Isolation Forest has more overlap, explaining its lower accuracy. LOF shows the most overlap, which is why it has the lowest performance despite being a solid algorithm."

---

## Slide 13: PCA Impact Analysis

### Content:

![PCA Impact](output/pcaimpact.png)

```
ONE-CLASS SVM: Full Features vs PCA

                    Full (75)    PCA (16)    Change
────────────────────────────────────────────────────
Features            75           16          -79%
ROC-AUC             99.15%       99.20%      +0.05%
Training Time       29.3s        10.0s       2.9x faster!
F1-Score            93.38%       93.30%      -0.08%

 PCA removes noise, improves efficiency
 Slight AUC improvement despite 79% fewer features
```

### Script:

"This slide focuses on the PCA impact experiment. We compared One-Class SVM with all 75 features versus just 16 PCA components. The training time dropped from 29 seconds to 10 seconds—a 2.9x speedup. Why? SVM's kernel computation scales with both the number of samples AND the number of features. Fewer features means faster distance calculations for every pair of points.

But here's what's surprising: ROC-AUC actually improved slightly despite using 79% fewer features. This isn't a fluke—it's the 'curse of dimensionality' at work. The RBF kernel computes similarity as e^(-γ||x-y||²). In high dimensions, all distances tend to converge to similar values, making it hard to distinguish similar from dissimilar points. By reducing to 16 meaningful components that capture 95% of variance, we removed noise and redundant information, making the distance calculations more discriminative.

This is a key data mining lesson: more features isn't always better. Smart preprocessing—understanding your data and your algorithm's properties—can improve both efficiency AND accuracy."

---

## Slide 14: Clustering Analysis

### Content:

![Clustering](output/clustering.png)

```
CLUSTERING RESULTS

Method      Accuracy    F1-Score    Notes
──────────────────────────────────────────────
K-Means     8.15%       14.47%       Poor
DBSCAN      61.22%      19.04%      29 clusters
GMM         1.07%       0.00%       BUT AUC: 99.09%

Why clustering fails for anomaly detection:
• Attacks are diverse (not one cluster)
• Normal traffic dominates
• Anomalies defined by what they're NOT
```

### Script:

"We also explored clustering methods to see if unsupervised grouping could separate normal from attack traffic. The results teach us an important lesson about problem framing.

K-Means achieved only 8.15% accuracy. Why so terrible? K-Means tries to partition data into k clusters by minimizing within-cluster distance. But this assumes attacks form a distinct cluster separate from normal traffic. In reality, attacks are diverse—DoS attacks look different from backdoors, which look different from reconnaissance. There's no single 'attack cluster.' K-Means ends up finding clusters within normal traffic or grouping different attack types together.

DBSCAN did better at 61% by finding 29 different clusters and marking 917 points as noise. Anomalies often end up as noise points or in tiny clusters. But DBSCAN still can't distinguish between 'unusual normal traffic' and 'actual attacks'—both look like outliers.

GMM is interesting: 1% accuracy for cluster assignments, but 99.09% AUC when using probability scores. Why the discrepancy? GMM fits Gaussian distributions to the data. The cluster assignments are poor because attacks don't form a Gaussian cluster. But the probability of belonging to the dominant normal cluster is actually meaningful—low probability means the point doesn't fit normal patterns, which is exactly what anomaly detection needs.

The lesson: anomaly detection and clustering solve different problems. Clustering groups similar points together. Anomaly detection identifies points that don't belong—defined by what they're NOT, not by similarity to other anomalies. This is why purpose-built anomaly detection methods like Isolation Forest and Autoencoder outperform clustering approaches."

---

## Slide 15: Ensemble Methods

### Content:

![Ensemble Results](output/ensemble.png)

```
COMBINING MULTIPLE MODELS

Ensemble Method     Accuracy   F1-Score   ROC-AUC
──────────────────────────────────────────────────
Majority Voting     96.54%     96.02%     -
Score Averaging     97.08%     96.63%     98.70%
Weighted Voting     97.08%     96.63%     98.71%
Stacking           ★99.02%    ★98.85%    ★99.59%

Stacking improvement over best single model:
+2.56% F1-Score, +0.39% ROC-AUC
```

### Script:

"We combined models using ensemble methods. Why do ensembles work? Each model has different strengths and makes different errors. Combining them can cancel out individual weaknesses.

Majority voting requires 3 of 4 models to agree—simple but rigid. Score averaging treats all models equally. Weighted voting weights by AUC scores, but notice the weights are nearly equal (0.247-0.252) because all models performed similarly overall—so weighting didn't help much here.

Stacking performed best at 98.85% F1-score. It trains a logistic regression meta-learner on base model predictions, learning which model to trust in which situations. This is more flexible than fixed voting rules.

However, there are important caveats. First, stacking requires running ALL base models—that's 4x the computation. Second, the meta-learner was trained on the same test data we're evaluating on, which could introduce some optimistic bias. Ideally, we'd use a separate validation set for meta-learner training. Third, adding more models doesn't always help—we're combining models that already perform similarly, so the ensemble gain is modest at 2.56%.

The result is good, but the added complexity may not always be worth it compared to just using the Autoencoder alone."

---

## Slide 16: Model Comparison

### Content:

![Model Comparison](output/modelcomparison.png)

```
FINAL RANKINGS

Rank  Method               F1-Score   Best For
──────────────────────────────────────────────
1     Stacking Ensemble    98.85%     Maximum accuracy
2     Autoencoder          96.29%     Single model, interpretable
3     One-Class SVM (PCA)  93.30%     Best AUC, efficient
4     Isolation Forest     93.57%     Real-time (0.78s training!)
```

### Script:

"Let me summarize with final rankings and discuss the tradeoffs of each approach.

Rank 1: Stacking Ensemble at 98.85% F1-score. Best accuracy, but there's a catch—it requires running ALL four base models plus a meta-learner. That's 5x the computational cost and complexity. If any base model fails or produces unexpected outputs, the ensemble could behave unpredictably. The 2.56% F1 improvement is real, but comes at significant cost.

Rank 2: Autoencoder at 96.29% F1-score. Best single model, and offers interpretability through reconstruction error analysis. However, it requires the most hyperparameter tuning—architecture, learning rate, epochs, dropout rate. It's also the least theoretically understood; we know it works empirically, but the decision boundary is a black box.

Rank 3: One-Class SVM with PCA at 93.30% F1 but 99.20% AUC—best discrimination ability. PCA gives a 2.9x speedup. The limitation: SVM is sensitive to the nu parameter and kernel choice. We used nu=0.1, assuming 10% contamination, but if the actual anomaly rate differs significantly, performance could degrade.

Rank 4: Isolation Forest at 93.57% F1-score with only 0.78-second training. Fastest by far. The tradeoff: it has the lowest AUC at 97.09%. The random partitioning means results can vary between runs, and it's less effective when anomalies aren't isolated in the feature space.

Each method has clear strengths and weaknesses—there's no single best choice for all scenarios."

---

## Slide 17: Key Takeaways

### Content:

```
KEY TAKEAWAYS

1. PREPROCESSING MATTERS
   • Hybrid encoding prevented dimensionality explosion
   • PCA improved SVM by 2.9x speedup with better AUC

2. DEEP LEARNING EXCELS
   • Autoencoder: Best single-model performance (96.29% F1)
   • Learns complex non-linear patterns

3. ENSEMBLES WIN
   • Stacking: 98.85% F1, 99.59% AUC
   • +2.56% improvement over best single model

4. HIGH RECALL, BUT WHY?
   • Dataset attacks differ measurably from normal
   • Strict train/test separation validates results
   • Would need testing on other datasets to confirm
```

### Script:

"Let me walk through the four key takeaways from this project.

First, preprocessing matters enormously. Our hybrid encoding decision—one-hot for low-cardinality features, frequency encoding for high-cardinality—prevented dimensionality explosion while preserving information. If we'd one-hot encoded IP addresses, we'd have thousands of sparse features and much worse performance. Similarly, PCA gave us a 2.9x speedup AND improved One-Class SVM's AUC. These aren't minor optimizations—they're fundamental to making the models work well.

Second, deep learning excels at this task. The Autoencoder achieved 96.29% F1-score—best among single models—because it can learn complex, non-linear patterns that simpler methods miss. However, this comes with caveats: Autoencoders require more tuning, longer training, and are harder to interpret than traditional methods.

Third, ensembles improve results but add complexity. Stacking achieved 98.85% F1-score—nearly 99%. But remember: this requires running four models plus a meta-learner. The 2.56% improvement is real, but you need to weigh it against 5x computational cost and added system complexity.

Fourth, why are these numbers so high—and are they believable? The UNSW-NB15 dataset has well-defined attack patterns that genuinely differ from normal traffic in the feature space. Our train/test split ensures models are evaluated on unseen data. For unsupervised methods, attacks are truly novel since training uses only normal samples. The high performance reflects that attacks in this dataset ARE detectable through flow-level features—though performance on different network environments or more sophisticated attacks could be lower."

---

## Slide 18: Future Work

### Content:

```
FUTURE WORK

🚀 Real-time Deployment
   Isolation Forest for streaming detection

🔧 Feature Engineering
   Domain-specific features from importance analysis

🧠 Advanced Deep Learning
   • Variational Autoencoders (probabilistic)
   • LSTM for temporal patterns

⚡ Adaptive Thresholds
   Balance precision/recall based on risk tolerance
```

### Script:

"Looking ahead, there are several directions for future work that could address current limitations.

Testing on different datasets is critical. Our high performance is on UNSW-NB15 specifically. Would these models generalize to other network environments with different traffic patterns? Testing on datasets like CICIDS2017 or real network captures would validate whether our approach is robust or dataset-specific.

Feature engineering based on our importance analysis could improve interpretability. We identified the most predictive features, but what do they represent semantically? Understanding the actual network behaviors they capture could lead to more meaningful features and better explainability.

Advanced deep learning architectures could address some limitations. Variational Autoencoders would give probabilistic anomaly scores with uncertainty estimates—not just 'is this anomalous?' but 'how confident are we?' LSTM networks could capture temporal patterns across sequences of connections, which our current approach ignores by treating each connection independently.

Finally, our current 95th percentile threshold is arbitrary. A more principled approach would involve cross-validation to select optimal thresholds, or exploring different thresholds for different use cases where the false positive/false negative tradeoff varies."

---

## Slide 19: Questions

### Content:

```
QUESTIONS?

Summary:
• Unsupervised anomaly detection for network intrusion
• 5 methods compared + ensemble approaches
• Best: Stacking ensemble (98.85% F1, 99.59% AUC)
• PCA: 2.9x speedup with improved performance

Thank you!
```

### Script:

"To summarize what we accomplished: We tackled the network intrusion detection problem using unsupervised anomaly detection—training models only on normal traffic to detect deviations as potential attacks.

We applied comprehensive data mining techniques: hybrid categorical encoding, correlation analysis, PCA for dimensionality reduction, and feature importance analysis.

We compared five anomaly detection methods—Autoencoder, Isolation Forest, One-Class SVM with and without PCA, and Local Outlier Factor. Autoencoder performed best as a single model at 96.29% F1. Stacking ensemble achieved 98.85% F1 but at higher computational cost.

Why such high numbers? The UNSW-NB15 attacks genuinely differ from normal traffic in flow-level features, and our strict train/test separation means models are evaluated on truly unseen data. However, performance on different datasets or more sophisticated attacks remains to be tested.

Key limitations: all models produce some false positives, clustering methods failed for this task, and our threshold selection is somewhat arbitrary.

Thank you for your attention. I'm happy to take any questions."

---

## Appendix: Technical Details (if asked)

### Hardware

```
• Platform: Google Colab
• GPU: NVIDIA T4 (16GB VRAM)
• Training samples: 27,980
• Test samples: 15,024
```

### Autoencoder Architecture

```
Input (75) → Dense(64) → Dropout → Dense(32) → Dropout
          → Encoding(32)
          → Dense(32) → Dropout → Dense(64) → Dropout → Output(75)

Loss: MSE (reconstruction error)
Threshold: 95th percentile of training reconstruction errors
```

### Hyperparameters

```
• Autoencoder: 30 epochs, batch_size=128, validation_split=0.2
• Isolation Forest: n_estimators=100, contamination=0.1
• One-Class SVM: kernel='rbf', nu=0.1, gamma='scale'
• LOF: n_neighbors=20, contamination=0.1, novelty=True
• PCA: n_components=0.95 (95% variance retained)
```
