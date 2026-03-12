# Telecom Customer Churn Prediction
### Apache Kafka + PySpark Medallion Architecture on Google Colab

> **DSCI 632 Final Project** — Kafka ingestion, Pyspark layered preprocessing, exploratory data analysis, and Random Forest classification entirely on free Google Colab infrastructure.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack and Version Pinning](#3-tech-stack-and-version-pinning)
4. [Dataset](#4-dataset)
5. [Environment Setup](#5-environment-setup)
6. [Cell-by-Cell Execution Guide](#6-cell-by-cell-execution-guide)
   - [Cell 1 — Install Everything](#cell-1-3-install-everything)
   - [Cell 2 — Start Kafka and Zookeeper](#cell-4--start-kafka-and-zookeeper)
   - [Cell 3 — Initialize SparkSession](#cell-5--initialize-sparksession)
   - [Cell 4 — Upload CSV and Produce to Kafka](#cell-6--upload-csv-and-produce-to-kafka)
   - [Cell 5 — Bronze and Silver Layers](#cell-7--bronze-and-silver-layers)
   - [Cell 6 — Exploratory Data Analysis](#cell-8--exploratory-data-analysis)
   - [Cell 7 — Outlier Detection and Winsorization](#cell-9--outlier-detection-and-winsorization)
   - [Cell 8 — Gold Layer Feature Engineering](#cell-10--gold-layer-feature-engineering)
   - [Cell 9 — ML Pipeline Construction](#cell-11--ml-pipeline-construction)
   - [Cell 10 — Training, Evaluation, and Model Save](#cell-12--training-evaluation-and-model-save)
   - [Cell 11 — Export Predictions to CSV](#cell-13--export-predictions-to-csv)
7. [Medallion Architecture Deep Dive](#7-medallion-architecture-deep-dive)
8. [Exploratory Data Analysis](#8-exploratory-data-analysis)
9. [Feature Engineering](#9-feature-engineering)
10. [Machine Learning Pipeline](#10-machine-learning-pipeline)
11. [Results](#11-results)
12. [Known Issues and Fixes](#12-known-issues-and-fixes)
13. [Session Persistence Notes](#13-session-persistence-notes)
14. [Project Structure](#14-project-structure)

---

## 1. Project Overview

This project builds a pipeline for predicting which telecom customers are likely to cancel their service (churn). Instead of loading a CSV directly into a model, each customer record is published as a JSON event to an **Apache Kafka** topic running locally inside Colab, then consumed by **PySpark** through the Spark-Kafka structured connector.

From there, data flows through three processing layers (Medallion Architecture) before a **Random Forest** classifier is trained with cross-validated hyperparameter tuning. The final model achieves **AUC 0.8532**, **F1 0.7809**, **Precision 0.8205**, and **Recall 0.777** on the held-out test set.

**Why Kafka on Colab?**
Most churn prediction tutorials treat the problem as a batch job. This project demonstrates that streaming-first thinking — where each customer event is an independent message — is achievable without paid infrastructure. The entire pipeline runs on Colab's free tier.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GOOGLE COLAB (Free Tier)                     │
│                                                                     │
│  WA_Fn-UseC_-Telco-Customer-Churn.csv                               │
│            │                                                        │
│            ▼  (kafka-python KafkaProducer)                          │
│  ┌─────────────────────┐                                            │
│  │   Kafka Topic       │  telco-churn  (port 9092)                  │
│  │   Kafka 3.6.2       │  1 partition, replication-factor 1         │
│  │   Zookeeper 3.6.2   │                                            │
│  └──────────┬──────────┘                                            │
│             │  (spark.read.format("kafka"))                         │
│             ▼                                                       │
│  ┌─────────────────────┐                                            │
│  │   BRONZE LAYER      │  Raw ingestion — from_json + schema        │
│  └──────────┬──────────┘                                            │
│             │                                                       │
│             ▼                                                       │
│  ┌─────────────────────┐                                            │
│  │   SILVER LAYER      │  Cleaning — null handling, type cast,      │
│  │                     │  row drop, label encoding                  │
│  └──────────┬──────────┘                                            │
│             │  IQR Outlier Detection + Winsorization                │
│             ▼                                                       │
│  ┌─────────────────────┐                                            │
│  │   GOLD LAYER        │  Feature engineering — 4 new columns       │
│  └──────────┬──────────┘                                            │
│             │                                                       │
│             ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │   MLlib PIPELINE                                        │        │
│  │   StringIndexer → OneHotEncoder → VectorAssembler       │        │
│  │   → StandardScaler → RandomForestClassifier             │        │
│  │   (3-fold CrossValidator, 2×2 ParamGrid)                │        │
│  └──────────┬──────────────────────────────────────────────┘        │
│             │                                                       │
│             ▼                                                       │
│   churn_predictions.csv                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Tech Stack and Version Pinning

Version pinning here is **not optional** — mismatched versions will cause silent failures or `ClassNotFoundException` errors at runtime.

| Component | Version | Why pinned |
|---|---|---|
| Java | OpenJDK 17 (`openjdk-17-jdk-headless`) | Required by PySpark 3.5.x; Java 11 works but 17 is stable on Colab |
| Apache Kafka | 3.6.2 (`kafka_2.13-3.6.2`) | Zookeeper-bundled build; 3.7+ changes quorum defaults |
| PySpark | 3.5.3 | 4.0.x Kafka connector jars have a `commons-pool2` incompatibility |
| Scala build | 2.12 (all jars) | Must match PySpark 3.5.x internal Scala version |
| spark-sql-kafka connector | 3.5.3 (`_2.12`) | Must match PySpark version exactly |
| spark-token-provider-kafka | 3.5.3 (`_2.12`) | Dependency of the Kafka connector |
| kafka-clients | 3.4.1 | Connector-compatible Kafka client library |
| commons-pool2 | **2.11.1** | The exact version PySpark 3.5.3 expects — 2.12.x breaks at runtime |
| kafka-python | Latest via pip | Python producer/consumer client; no version pin needed |
| pandas | Colab default | Used only for CSV read and toPandas() |
| matplotlib / seaborn / scipy | Colab default | EDA visualizations |

---

## 4. Dataset

**Source:** IBM Telco Customer Churn Dataset  
**Download:** [https://www.kaggle.com/datasets/blastchar/telco-customer-churn](https://www.kaggle.com/datasets/blastchar/telco-customer-churn)  
**File name:** `WA_Fn-UseC_-Telco-Customer-Churn.csv`

| Property | Value |
|---|---|
| Total records | 7,043 |
| Features | 21 |
| Records after quality check | 7,032 (11 dropped) |
| Class distribution | 73.5% retained / 26.5% churned |
| Target column | `Churn` (Yes/No → encoded to 1/0) |

**21 columns:**

| Column | Type | Notes |
|---|---|---|
| customerID | String | Unique identifier |
| gender | String | Male / Female |
| SeniorCitizen | Integer | 0 or 1 |
| Partner | String | Yes / No |
| Dependents | String | Yes / No |
| tenure | Integer | Months as customer |
| PhoneService | String | Yes / No |
| MultipleLines | String | Yes / No / No phone service |
| InternetService | String | DSL / Fiber optic / No |
| OnlineSecurity | String | Yes / No / No internet service |
| OnlineBackup | String | Yes / No / No internet service |
| DeviceProtection | String | Yes / No / No internet service |
| TechSupport | String | Yes / No / No internet service |
| StreamingTV | String | Yes / No / No internet service |
| StreamingMovies | String | Yes / No / No internet service |
| Contract | String | Month-to-month / One year / Two year |
| PaperlessBilling | String | Yes / No |
| PaymentMethod | String | Electronic check / Mailed check / Bank transfer / Credit card |
| MonthlyCharges | Double | Monthly bill amount |
| TotalCharges | String | **Source type is String** — contains blank spaces for new customers |
| Churn | String | Yes / No — target variable |

> ⚠️ **TotalCharges is a String in the raw file**, not a number. Rows with zero tenure have a blank space `" "` rather than `"0"`. This is handled explicitly in the Silver layer.

---

## 5. Environment Setup

This project runs entirely in **Google Colab**. No local installation is needed beyond opening the notebook.

**Before running:**
1. Open [https://colab.research.google.com](https://colab.research.google.com)
2. Upload `632_Final_Project.ipynb`
3. Download `WA_Fn-UseC_-Telco-Customer-Churn.csv` from Kaggle — keep it ready to upload in Cell 4
4. Ensure the runtime type is set to **Python 3** (CPU runtime is sufficient — no GPU needed)

**Runtime reset behavior:**  
Colab's file system is ephemeral. Everything in `/kafka`, `/kafka_jars`, and installed packages disappears when the runtime resets. You must re-run Cell 1 and restart the runtime at the start of every new session.

---

## 6. Cell-by-Cell Execution Guide

> **Critical rule:** Cells must be run in the exact order shown. Do not skip any cell. After Cell 1, you must **restart the runtime** before continuing.

---

### Cell 1 — Install Everything

**Run once per session. Restart runtime immediately after.**


**What this does:**
- Installs OpenJDK 17 via apt
- Installs PySpark 3.5.3 and kafka-python via pip
- Downloads and extracts Kafka 3.6.2 to `/kafka`
- Downloads all 4 required jars to `/kafka_jars/`

**After this cell completes:** Go to **Runtime → Restart Runtime**. Do not click "Run All" — restart first.

**If Kafka download is corrupted** (tar extraction fails):
```python
!rm -rf /kafka
!rm -f kafka_2.13-3.6.2.tgz
!wget https://archive.apache.org/dist/kafka/3.6.2/kafka_2.13-3.6.2.tgz
!ls -lh kafka_2.13-3.6.2.tgz  # Must show ~109MB
!tar -xzf kafka_2.13-3.6.2.tgz
!mv kafka_2.13-3.6.2 /kafka
```

---

### Cell 2 — Start Kafka and Zookeeper

**Run after every runtime restart, before any other cell.**

```python
import os, subprocess, time

os.environ["JAVA_HOME"] = "/usr/lib/jvm/java-17-openjdk-amd64"
os.environ["PATH"] = os.environ["JAVA_HOME"] + "/bin:" + os.environ["PATH"]

zk = subprocess.Popen(
    ["/kafka/bin/zookeeper-server-start.sh", "/kafka/config/zookeeper.properties"],
    stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL
)
time.sleep(6)

kb = subprocess.Popen(
    ["/kafka/bin/kafka-server-start.sh", "/kafka/config/server.properties"],
    stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL
)
time.sleep(10)
!ss -tlnp | grep 9092
```

**What this does:**
- Sets `JAVA_HOME` to the OpenJDK 17 path Colab uses
- Starts Zookeeper (waits 6 seconds for it to be ready)
- Starts the Kafka broker (waits 10 seconds for it to be ready)
- Verifies port 9092 is open

**Expected output:** A line showing `LISTEN` on port `9092`.  
**If nothing is printed:** Kafka failed to start. Check `/kafka` exists with `!ls /kafka/bin/`. If not, re-run Cell 1.

---

### Cell 3 — Initialize SparkSession

**What this does:**
- Passes all 4 jars explicitly via `spark.jars` — this is required because Colab's PySpark install does not bundle the Kafka connector
- Enables Apache Arrow for faster `toPandas()` conversion during EDA
- Expected output: ` Spark version: 3.5.3`

---

### Cell 4 — Upload CSV and Produce to Kafka

**What this does:**
- Triggers a file upload dialog — select `WA_Fn-UseC_-Telco-Customer-Churn.csv`
- Creates the `telco-churn` Kafka topic (1 partition, replication factor 1)
- Reads the CSV into pandas and serializes each row as a UTF-8 encoded JSON dict
- Produces all 7,043 records to the topic one by one
- Calls `flush()` to ensure all messages are delivered before closing

**Expected output:** ` Produced 7043 records to Kafka`

---

### Cell 5 — Bronze and Silver Layers

**What this does:**

*Bronze:*
- Reads all messages from the `telco-churn` topic with `startingOffsets=earliest`
- Casts the binary Kafka `value` column to String
- Applies `from_json` with the manual StructType schema to parse JSON into columns
- Flattens the resulting struct with `.select("data.*")`

*Silver:*
- Replaces blank-space `" "` values in `TotalCharges` with `null` before casting — if the cast happens first, blanks become `NaN` instead of `null` and the drop step fails silently
- Casts `TotalCharges` from StringType to DoubleType
- Drops rows where `TotalCharges` or `customerID` is null (removes 11 records)
- Adds a binary `label` column: `Churn == "Yes"` → 1, else 0

**Expected output:**
```
--- [BRONZE] Ingesting from Kafka ---
   Records ingested: 7043
--- [SILVER] Cleaning & Enforcing Quality ---
   DQ Check: Dropped 11 rows due to data quality issues.
```

---

### Cell 6 — Exploratory Data Analysis

Full EDA runs on `silver_df.toPandas()`. Organized into five analytical phases:

```python
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import seaborn as sns
import pandas as pd
import numpy as np
from scipy import stats as scipy_stats

silver_pd = silver_df.toPandas()
sns.set_theme(style="whitegrid")
```

**Phase 1 — The Macro View (Who and What)**
- **Target distribution:** Bar chart + pie chart of Churn Yes/No counts and proportions
- **Categorical sweep:** Churn rate bar charts for Contract, InternetService, PaymentMethod, gender, SeniorCitizen, Partner, Dependents, PaperlessBilling
- **Strategic bubble chart:** Churn rate vs customer count per segment (Contract, InternetService, PaymentMethod), bubble size = absolute churner count, quadrant reference lines at 30% and 15%

**Phase 2 — Time Dynamics (When They Leave)**
- **Tenure cohort analysis:** Customers bucketed into 7 bins (0–6m, 7–12m, 13–24m, 25–36m, 37–48m, 49–60m, 61–72m); churn rate and customer count plotted per bin
- **CDF plots:** Cumulative distribution functions for MonthlyCharges, TotalCharges, and tenure, overlaid for churners vs retained

**Phase 3 — Financials and Interactions (Cost vs Loyalty)**
- **Distribution analysis:** Stacked histograms + box plots for tenure, MonthlyCharges, TotalCharges by churn status
- **Multivariate scatter:** tenure × MonthlyCharges and tenure × TotalCharges colored by churn status

**Phase 4 — Product Stickiness (Why They Stay)**
- **Service adoption comparison:** Grouped bar chart of adoption rates for OnlineSecurity, TechSupport, OnlineBackup, DeviceProtection, StreamingTV, StreamingMovies — churners vs retained
- **Stacked service composition by contract:** Stacked bar showing which services are adopted within each contract type

**Phase 5 — Statistical Rigor**
- **Correlation heatmap:** Pearson correlation matrix for tenure, MonthlyCharges, TotalCharges, label
- **Lorenz curve + Gini coefficient:** Cumulative churn concentration by contract type; area between the curve and perfect-equality line gives the Gini coefficient
- **Dispersion table:** `.describe()` output for each numerical feature, grouped by Churn status

---

### Cell 7 — Outlier Detection and Winsorization


**What this does:**
- Computes Q1 and Q3 with `approxQuantile` at 0.01 relative error for TotalCharges, MonthlyCharges, and tenure
- Reports outlier count and shows sample records before any modification
- Caps outliers to `[lower_bound, upper_bound]` using `withColumn` — rows are kept, values are capped (Winsorization)
- Runs Spark-native Pearson correlation across all numerical column pairs and prints any pair with `|r| > 0.5`

**Why Winsorization over dropping:** 
- Outliers here are **real customers** (long-tenured, high-spend accounts), not data entry errors — removing them deletes genuine signal

- Dropping rows would **worsen the class imbalance** — high-spend long-tenured customers skew toward the retained class, and removing them makes the 73.5/26.5 split even harder to handle

- The model needs exposure to **the full customer spectrum** at training time, including extreme spenders — a model trained only on average customers makes unreliable predictions for edge-case customers at inference

- Winsorization **preserves rank ordering** — a 99th percentile customer stays at the ceiling after capping, so the relative signal is intact, only the extreme leverage on variance is reduced

- The **IQR 1.5× threshold is conservative** — it only flags genuine tail values, not moderately high ones; for tenure specifically, the bounds cover nearly the full 0–72 month range so almost no rows are affected there

- Capping keeps the **row count fixed at 7,032** — no downstream impact on train/test split sizes or the class weight calculation, unlike deletion which would require recalculating the balancing ratio
---

### Cell 8 — Gold Layer Feature Engineering

```python
def process_gold(df):
    print("--- [GOLD] Feature Engineering ---")

    df_feat = df.withColumn("avg_daily_cost",
                            F.col("MonthlyCharges") / 30)

    df_feat = df_feat.withColumn("tenure_group",
                F.when(F.col("tenure") <= 12, 0)
                 .when(F.col("tenure") <= 36, 1)
                 .otherwise(2))

    df_feat = df_feat.withColumn("is_high_value",
                F.when(F.col("MonthlyCharges") > 70, 1).otherwise(0))

    df_feat = df_feat.withColumn("risk_score",
                F.col("MonthlyCharges") / (F.col("tenure") + 1))

    print("   Features added: avg_daily_cost, tenure_group, is_high_value, risk_score")
    return df_feat

gold_df = process_gold(silver_df)
gold_df.select("customerID", "avg_daily_cost", "tenure_group",
               "is_high_value", "risk_score", "label").show(5, truncate=False)
```

**Four engineered features and their EDA justification:**

| Feature | Formula | EDA finding that motivated it |
|---|---|---|
| `avg_daily_cost` | `MonthlyCharges / 30` | Expresses cost burden per day; improves interpretability of feature importance |
| `tenure_group` | 0 if ≤12m, 1 if ≤36m, 2 otherwise | Cohort analysis showed >50% churn in 0–6m, <10% after 36m — non-linear threshold |
| `is_high_value` | 1 if `MonthlyCharges > 70` else 0 | Churners average $74/month vs $61 for retained — $70 threshold splits the groups |
| `risk_score` | `MonthlyCharges / (tenure + 1)` | Scatter plots showed churners cluster at high charges + low tenure; +1 prevents division by zero |

---

### Cell 9 — ML Pipeline Construction
The model is built using the **Spark MLlib Pipeline API** to ensure reproducibility and prevent data leakage.


**Pipeline stage order:**

```
StringIndexer (×10) → OneHotEncoder (×10) → VectorAssembler → StandardScaler → RandomForestClassifier
```

**Why each stage:**
- **StringIndexer:** Maps string categories to frequency-ordered integers. `handleInvalid="skip"` prevents crashes on unseen values at inference time
- **OneHotEncoder:** Converts integer indices to binary vectors — prevents the model treating category labels as ordinal numbers
- **VectorAssembler:** Combines all 10 OHE vectors + 7 numerical columns into one `features_raw` sparse vector
- **StandardScaler:** Normalizes to zero mean and unit variance — guards against numerical instability from mixed-scale features
- **RandomForestClassifier:** `weightCol="classWeight"` applies the 2.76:1 class weight to handle the 73.5/26.5 imbalance

### 5.2 Class Imbalance Strategy
The dataset is imbalanced (2.76:1 ratio). We opted for **Cost-Sensitive Learning**:
- **Balancing Ratio:** 2.76
- **Weighting:** Churners receive a `classWeight` of 2.76; Retained customers receive 1.0. This tells the algorithm to penalize a "False Negative" 2.76x more than a "False Positive."

### 5.3 Hyperparameter Tuning
We utilized a **3-fold CrossValidator** with a `ParamGridBuilder`:
- `numTrees`: [20, 50]
- `maxDepth`: [5, 10]
- **Selection Metric:** Area Under ROC (AUC).

---

### Cell 10 — Training, Evaluation, and Model Save


**What this does:**
- Splits data 80/20 with `seed=42` for reproducibility
- Builds a 2×2 parameter grid: `numTrees` in [20, 50] × `maxDepth` in [5, 10] = 4 configurations
- Runs 3-fold cross-validation, training 12 models total, selecting by AUC
- Evaluates the best model on the held-out test set across 4 metrics
- Extracts and prints top 10 feature importances from the best Random Forest
- Saves the fitted pipeline to `/content/churn_model`

The final Random Forest model achieved strong discriminatory power across all folds.

### Final Results:
- **AUC:** `0.8512`
- **F1-Score:** `0.7770`
- **Precision:** `0.8199`
- **Recall:** `0.7645`

### Feature Importance:
1.  **PaymentMethod_vec:** 22.9%
2.  **MonthlyCharges:** 8.2%
3.  **avg_daily_cost:** 7.0%
4.  **Contract_vec:** 0.9% (Higher interaction effect with `tenure_group`)

---

### Cell 11 — Export Predictions to CSV


Downloads `churn_predictions.csv` containing the actual label, predicted label, and probability scores for both classes for every test-set customer.

---

## 7. Medallion Architecture Deep Dive

```
RAW CSV
  │
  ▼ (KafkaProducer — row-by-row JSON serialization)
KAFKA TOPIC: telco-churn
  │
  ▼ (spark.read.format("kafka") + from_json)
BRONZE — 7,043 rows, 21 columns, raw types, no transformations
  │
  ▼ (blank→null, cast, na.drop, label encoding)
SILVER — 7,032 rows, 22 columns (+ label), clean types
  │
  ▼ (IQR + Winsorization on 3 numerical columns)
SILVER (capped) — same 7,032 rows, outlier-bounded values
  │
  ▼ (4 withColumn expressions)
GOLD — 7,032 rows, 26 columns (+ 4 engineered features)
  │
  ▼ (MLlib Pipeline)
PREDICTIONS — test set rows with label, prediction, probability
```

Each layer has a single responsibility. Bronze never modifies data. Silver never adds features. Gold never drops rows.

---

## 8. Exploratory Data Analysis

### Key Findings

| Finding | Value | Modelling decision driven |
|---|---|---|
| Class imbalance | 73.5% retained / 26.5% churned | Class weighting 2.76:1; use AUC + F1, not accuracy |
| Tenure step-change | >50% churn in 0–6m, <10% after 36m | `tenure_group` feature; Random Forest over Logistic Regression |
| Contract type spread | 43% (month-to-month) vs 3% (two-year) | Contract is highest-priority categorical feature |
| Fiber optic churn | 42% churn rate | InternetService as high-priority feature |
| OnlineSecurity gap | 17% (churners) vs 42% (retained) adoption | Include all protective service features |
| Gender irrelevance | ~26% churn both groups | Expected low feature importance |
| Tenure–TotalCharges correlation | r = 0.83 | `risk_score` and `avg_daily_cost` to address multicollinearity |
| Churner charge cluster | Scatter shows churners at high charges + low tenure | `risk_score = MonthlyCharges / (tenure + 1)` |

---

## 9. Feature Engineering

| Feature | Formula | Type | Motivation |
|---|---|---|---|
| `avg_daily_cost` | `MonthlyCharges / 30` | Double | Daily cost unit for interpretability |
| `tenure_group` | 0/1/2 by tenure thresholds | Integer | Encodes the non-linear step-change in churn risk |
| `is_high_value` | 1 if `MonthlyCharges > 70` | Integer | Binary flag for high-spend churner segment |
| `risk_score` | `MonthlyCharges / (tenure + 1)` | Double | Compresses 2D churn cluster pattern into 1 variable |

---

## 10. Machine Learning Pipeline

### Input features (17 total)

**Categorical (10):** InternetService, Contract, PaymentMethod, gender, Partner, Dependents, PhoneService, OnlineSecurity, TechSupport, StreamingTV

**Numerical (7):** tenure, MonthlyCharges, TotalCharges, avg_daily_cost, tenure_group, is_high_value, risk_score

### Hyperparameter search space

| Parameter | Values searched |
|---|---|
| `numTrees` | 20, 50 |
| `maxDepth` | 5, 10 |
| CV folds | 3 |
| Selection criterion | AUC (BinaryClassificationEvaluator) |

### Best parameters found
- `numTrees`: 50
- `maxDepth`: 5

---

## 11. Results

| Metric | Value |
|---|---|
| AUC | 0.848 |
| F1 Score | 0.80 |
| Precision | 0.81 |
| Recall | 0.81 |
| Best numTrees | 50 |
| Best maxDepth | 5 |
| Training records | ~5,626 (80%) |
| Test records | ~1,406 (20%) |

**Interpretation:** 8 out of 10 customers flagged as at-risk are genuine churners (precision). The model finds 8 out of 10 actual churners in the test set (recall). The depth-5 model outperforming depth-10 confirms cross-validation was necessary — the deeper tree overfit the training data.

---

## 12. Known Issues and Fixes

### commons-pool2 version conflict
**Symptom:** `java.lang.NoSuchMethodError` or `ClassNotFoundException` when Spark tries to connect to Kafka.  
**Cause:** Using commons-pool2 version other than 2.11.1 with PySpark 3.5.3.  
**Fix:** Ensure only `commons-pool2-2.11.1.jar` is in `/kafka_jars/`. Delete any other version.

### Kafka download incomplete (unexpected end of file)
**Symptom:** `tar` fails with "unexpected end of file" or "Error is not recoverable".  
**Cause:** The `.tgz` file downloaded as a truncated partial file.  
**Fix:**
```python
!rm -rf /kafka
!rm -f kafka_2.13-3.6.2.tgz
!wget https://archive.apache.org/dist/kafka/3.6.2/kafka_2.13-3.6.2.tgz
!ls -lh kafka_2.13-3.6.2.tgz  # Must show ~109MB
```

### Port 9092 not open after Cell 2
**Symptom:** `ss -tlnp | grep 9092` returns nothing.  
**Cause:** Kafka failed to start, usually because Zookeeper was not ready in time.  
**Fix:** Run Cell 2 again, or increase the sleep values to 10 and 15 seconds.

### PySpark 4.x incompatibility
**Symptom:** `pip install pyspark` installs 4.x which causes Kafka connector failures.  
**Fix:** Always use `pip install pyspark==3.5.3` explicitly.

### TotalCharges null handling
**Symptom:** Silver layer drops 0 rows despite 11 known bad records.  
**Cause:** The blank-space replacement ran after the cast, so blanks became `NaN` not `null`.  
**Fix:** The `F.when(F.col("TotalCharges") == " ", None)` must happen before `.cast(DoubleType())` — this is already correct in the notebook. Do not reorder these operations.

---

## 13. Session Persistence Notes

Everything below resets when Colab runtime disconnects:

| What resets | Impact | Fix |
|---|---|---|
| `/kafka` binaries | Kafka won't start | Re-run Cell 1 |
| `/kafka_jars` | SparkSession fails | Re-run Cell 1 |
| Python packages (pyspark, kafka-python) | Import errors | Re-run Cell 1 |
| Kafka broker process | No topic, no messages | Re-run Cell 2 |
| Uploaded CSV | Cannot produce to Kafka | Re-upload in Cell 4 |
| All Spark DataFrames | All variables undefined | Re-run Cells 3–8 in order |

**To avoid re-downloading jars every session**, mount Google Drive and persist them:

```python
from google.colab import drive
drive.mount('/content/drive')

import os
os.makedirs('/content/drive/MyDrive/kafka_setup/jars', exist_ok=True)

# Run once to save jars to Drive
!cp /kafka_jars/*.jar /content/drive/MyDrive/kafka_setup/jars/
!cp -r /kafka /content/drive/MyDrive/kafka_setup/kafka_3.6.2

# In future sessions, copy from Drive instead of re-downloading
!cp -r /content/drive/MyDrive/kafka_setup/kafka_3.6.2 /kafka
!mkdir -p /kafka_jars
!cp /content/drive/MyDrive/kafka_setup/jars/*.jar /kafka_jars/
```

---

## 14. Project Structure

```
632_Final_Project.ipynb          # Main notebook — all cells in order
WA_Fn-UseC_-Telco-Customer-Churn.csv  # Dataset (download from Kaggle)
churn_predictions.csv            # Generated output — test set predictions
/content/churn_model/            # Saved MLlib pipeline (Spark format)
requirements.txt                 # Python package versions for reference
README.md                        # This file
```

**Runtime file system (ephemeral — lost on reset):**
```
/kafka/                          # Kafka 3.6.2 binaries
/kafka_jars/
  ├── spark-sql-kafka-0-10_2.12-3.5.3.jar
  ├── spark-token-provider-kafka-0-10_2.12-3.5.3.jar
  ├── kafka-clients-3.4.1.jar
  └── commons-pool2-2.11.1.jar
```

---

*DSCI 632 — Drexel University*
