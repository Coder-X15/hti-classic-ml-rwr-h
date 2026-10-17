# Methods Being Attempted

## Things to Take Note Of:
## The Edge-Level Transductive Split (Fixed Node Universe)

* **Positive Pairs ($P$):** Split known positive interactions into:
* **Train Positives ($P_{\text{train}}$):** 70%
* **Validation Positives ($P_{\text{val}}$):** 15%
* **Test Positives ($P_{\text{test}}$):** 15%


* **Unlabeled Pairs ($U$ / Background Pool):**
* Leave the bulk of unlabeled pairs in the training/graph background pool.
* For validation, draw a random sample of unlabeled pairs $U_{\text{val}}$ at a $1:10$ ratio to $P_{\text{val}}$.
* For testing, draw a random sample of unlabeled pairs $U_{\text{test}}$ at a $1:50$ or $1:100$ ratio to $P_{\text{test}}$ to realistically simulate sparsity.



---

## The Inductive Split (Cold-Start / Unseen Entities)

* If your goal is to predict interactions for a new, unseen entity $B$:
* Partition the nodes of $B$ into 70% Train, 15% Val, 15% Test.
* Any edge involving a test node is held out exclusively for testing.



---

## Evaluation Metrics

* **Do NOT use ROC-AUC or Accuracy:** Unlabeled pools are massive; ROC-AUC will look deceptively high ($0.95+$) even when the model fails to surface true positives.
* **Use PR-AUC (Precision-Recall AUC):** Measures precision across positive discovery.
* **Use Hits@$K$ (e.g., Hits@10, Hits@50) & MRR (Mean Reciprocal Rank):** Verifies whether the true held-out positives appear in the top-ranked hypothesis slots.

## Classical ML

These methods use pairwise feature vectors: $\mathbf{x}_{\text{pair}} = [\mathbf{x}_A, \mathbf{x}_B, \mathbf{x}_A \odot \mathbf{x}_B]$.

### 1. Two-Step PU Learning (Isolation Forest / One-Class SVM + XGBoost)

* **Step 1 (Find Reliable Negatives):** Train an Isolation Forest or One-Class SVM strictly on the feature vectors of $P_{\text{train}}$. Score all unlabeled pairs $U$. The bottom 10–20% of pairs (furthest from positives in feature space) are marked as **Reliable Negatives (RN)**.
* **Step 2 (Classifier Training):** Label $P_{\text{train}}$ as $1$ and $\text{RN}$ as $0$. Train an XGBoost model.
* **Step 3 (Iterative Expansion):** Predict probabilities on the remaining unlabeled pool ($U \setminus \text{RN}$). Items with $P(\text{positive}) < 0.01$ are added into $\text{RN}$. Retrain XGBoost. Repeat for 2–3 iterations.

### 2. Elkan & Noto Calibrated XGBoost / LightGBM

* Treat all $P_{\text{train}}$ as $s = 1$, and all training unlabeled pairs $U_{\text{train}}$ as $s = 0$.
* Train standard XGBoost with binary log-loss to predict $P(s = 1 \mid \mathbf{x})$.
* Compute label frequency constant $c$ on held-out positive validation set $P_{\text{val}}$:

$$c \approx \frac{1}{\vert{}P_{\text{val}}\vert{}} \sum_{\mathbf{x} \in P_{\text{val}}} \hat{P}(s = 1 \mid \mathbf{x})$$


* Calibrate test/unlabeled predictions:

$$P(\text{real interaction} \mid \mathbf{x}) = \min\left(1.0, \, \frac{\hat{P}(s = 1 \mid \mathbf{x})}{c}\right)$$



### 3. Cost-Sensitive (Biased) SVM

* Treat unlabeled pairs as negatives ($y = -1$) and known positives as positives ($y = +1$).
* Set asymmetric slack penalties ($C^+, C^-$) in the objective function:

$$\min_{\mathbf{w}, b} \frac{1}{2} \Vert{}\mathbf{w}\Vert{}^2 + C^+ \sum_{i \in P} \xi_i + C^- \sum_{j \in U} \xi_j$$


* Choose $C^+ / C^- \approx \vert{}U\vert{} / \vert{}P\vert{}$ (e.g., 50:1 to 100:1). The model heavily avoids false negatives while ignoring minor false positives in the unlabeled background.

## Heterogeneous Random Walk with Restart (RWR-H)

* Build the combined transition probability matrix $W$ of the entire C-A-B network.
* For a target node $a_0$, set the initial state vector $p_0$ (with $1.0$ at index $a_0$) and iterate until convergence:

$$p_{t+1} = (1 - c)W^T p_t + c p_0$$


* The values in $p$ corresponding to node set $B$ represent the topological proximity of each element in $B$ to $a_0$.