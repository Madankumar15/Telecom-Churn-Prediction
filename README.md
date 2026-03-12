# 📡 Telco Customer Churn: End-to-End Predictive Pipeline
> **A PySpark-driven Medallion Architecture and Random Forest Classifier for Churn Mitigation.**


## 📋 Table of Contents
1. [Project Overview](#project-overview)
2. [Data Architecture (Medallion)](#data-architecture-medallion)
3. [Exploratory Data Analysis (EDA)](#exploratory-data-analysis-eda)
4. [Feature Engineering (Gold Layer)](#feature-engineering-gold-layer)
5. [Machine Learning Pipeline](#machine-learning-pipeline)
6. [Model Performance](#model-performance)
7. [Installation & Usage](#installation--usage)

---

## 1. Project Overview
This project addresses the business challenge of customer attrition in the telecommunications industry. The objective is to identify high-risk customers using behavioral and financial indicators. We hypothesize that churn is a **non-linear event** driven by the interaction of high monthly costs and low contractual commitment.

---

## 2. Data Architecture (Medallion)
We implement a **Medallion Architecture** to ensure data lineage and ACID compliance:

* **Bronze (Raw):** Ingestion of the `Telco-Customer-Churn.csv` into Spark DataFrames with minimal transformation.
* **Silver (Cleaned):** Type casting (e.g., `TotalCharges` to float), handling missing values via median imputation, and schema enforcement.
* **Gold (Enriched):** Feature derivation and business logic implementation (e.g., `risk_score`, `tenure_group`).

---

## 3. Exploratory Data Analysis (EDA)
The EDA provides the "Statistical Rationale" for the Machine Learning model.

### Key Insights:
* **The 12-Month Cliff:** Churn risk peaks at **>50%** in the first 6 months and decays non-linearly. Linear models (Logistic Regression) are likely to underfit this step-change behavior.
* **The Geometry of Churn:** Scatter plots of `Tenure` vs `MonthlyCharges` reveal a dense cluster of churners in the **high-cost / low-loyalty** quadrant.
* **Lorenz Curve & Gini Coefficient:** We calculated a high Gini coefficient for **Contract Type**, proving that churn is not evenly distributed but concentrated in Month-to-Month accounts.

---

## 4. Feature Engineering (Gold Layer)
Based on the EDA, we engineered four "Signal Amplifiers":

| Feature | Logic | Rationale |
| :--- | :--- | :--- |
| `avg_daily_cost` | `MonthlyCharges / 30` | Normalizes spend for granular comparison. |
| `tenure_group` | `Binned [0, 12, 36, 72]` | Explicitly encodes the 12-month risk cliff. |
| `is_high_value` | `MonthlyCharges > 70` | Flags customers above the churner median spend. |
| `risk_score` | `MonthlyCharges / (tenure + 1)` | Compresses the 2D cluster found in EDA into a 1D predictor. |

---

## 5. Machine Learning Pipeline
The model is built using the **Spark MLlib Pipeline API** to ensure reproducibility and prevent data leakage.

### 5.1 Pipeline Stages
1.  **Categorical Processing:** 10 columns (e.g., `Contract`, `InternetService`) pass through `StringIndexer` and `OneHotEncoder`.
2.  **Vectorization:** `VectorAssembler` consolidates all 17 features into a single `features_raw` vector.
3.  **Standardization:** `StandardScaler` normalizes features to zero mean and unit variance.
4.  **Classification:** A **RandomForestClassifier** optimized via Cross-Validation.

### 5.2 Class Imbalance Strategy
The dataset is imbalanced (2.76:1 ratio). We opted for **Cost-Sensitive Learning**:
* **Balancing Ratio:** 2.76
* **Weighting:** Churners receive a `classWeight` of 2.76; Retained customers receive 1.0. This tells the algorithm to penalize a "False Negative" 2.76x more than a "False Positive."

### 5.3 Hyperparameter Tuning
We utilized a **3-fold CrossValidator** with a `ParamGridBuilder`:
* `numTrees`: [20, 50]
* `maxDepth`: [5, 10]
* **Selection Metric:** Area Under ROC (AUC).

---

## 6. Model Performance
The final Random Forest model achieved strong discriminatory power across all folds.

### Final Results:
* **AUC:** `0.8512`
* **F1-Score:** `0.7770`
* **Precision:** `0.8199`
* **Recall:** `0.7645`

### Feature Importance:
1.  **PaymentMethod_vec:** 22.9%
2.  **MonthlyCharges:** 8.2%
3.  **avg_daily_cost:** 7.0%
4.  **Contract_vec:** 0.9% (Higher interaction effect with `tenure_group`)

---

## 7. Installation & Usage

### Prerequisites
* Java 8/11
* Spark 3.x
* Python 3.8+
