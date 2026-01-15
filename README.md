# 🛡️ End-to-End Fraud Detection Pipeline on Databricks

<div align="center">

<img src="https://upload.wikimedia.org/wikipedia/commons/5/51/Google_Cloud_logo.svg" alt="Google Cloud" width="150" style="margin-right: 20px;"/>
<img src="https://upload.wikimedia.org/wikipedia/commons/6/63/Databricks_Logo.png" alt="Databricks" width="150" style="margin-right: 20px;"/>
<img src="https://upload.wikimedia.org/wikipedia/commons/f/f3/Apache_Spark_logo.svg" alt="Apache Spark" width="110" style="margin-right: 20px;"/>
<img src="https://upload.wikimedia.org/wikipedia/commons/d/df/Delta_Lake_logo.png" alt="Delta Lake" width="150"/>

</div>

## 📌 Project Overview
This project demonstrates a scalable, production-grade Data Engineering pipeline built to detect fraudulent mobile money transactions. Using **Databricks** and the **Medallion Architecture**, the system processes over **6.3 million transactions**, enforces data quality, and utilizes Machine Learning to identify fraud patterns with near-perfect precision.

**Business Goal:**
The primary objective was to maximize fraud detection (Recall) while maintaining **Near-Perfect Precision (>99.9%)** to ensure that genuine customer transactions are almost never blocked (minimizing customer friction).

![Data Sample](images/data_sample.png)
*Snapshot of the processed transaction data.*

---

## 📂 Data Dictionary

The dataset consists of **11 columns** representing transaction details. Below are the key features used in this project:

| Column Name | Description |
| :--- | :--- |
| **step** | Maps to a unit of time in the real world. In this simulation, **1 step is 1 hour**. |
| **type** | Type of transaction (CASH-IN, CASH-OUT, DEBIT, PAYMENT, TRANSFER). *Only TRANSFER and CASH_OUT are used for fraud detection.* |
| **amount** | Amount of the transaction in local currency. |
| **nameOrig** | Customer who started the transaction. |
| **oldbalanceOrg** | Initial balance before the transaction. |
| **newbalanceOrig** | New balance after the transaction. |
| **nameDest** | Customer who is the recipient of the transaction. |
| **oldbalanceDest** | Initial balance recipient before the transaction. |
| **newbalanceDest** | New balance recipient after the transaction. |
| **isFraud** | **Target Variable.** Actual fraud status (1 = Fraud, 0 = Normal). |
| **isFlaggedFraud** | The business rule controls massive transfers (>200.000). *Not used in this model as we built our own rules.* |

---

## 🏗️ Architecture & Tech Stack

The pipeline follows the **Medallion Architecture** design pattern (Bronze $\to$ Silver $\to$ Gold) ensuring robust data lineage:

![Architecture Design](images/architecture_diagram.png)

1.  **Ingestion (Bronze):** Securely ingests raw CSV data from Google Cloud Storage (GCS) into Delta Lake using **Spark Native Read**.
2.  **Transformation (Silver):** Implements a strategic 3-way data split:
    * **Silver:** Valid transactions for ML (Transfer/Cash-out).
    * **Others:** Out-of-scope data (Payment/Debit) archived for analytics.
    * **Quarantine:** Technical errors (e.g., negative amounts) isolated for auditing.
3.  **Feature Engineering (Gold):** Creates behavioral features such as `amountRatio` (emptying accounts) and `errorBalance` (mathematical anomalies).
4.  **Machine Learning:** Trains a Random Forest Classifier using **Strict Time-Series Splitting** to prevent data leakage.

**🛠️ Technologies:**
* **Cloud Source:** Google Cloud Storage (GCS)
* **Platform:** Databricks (Spark Engine)
* **Storage:** Delta Lake (ACID Transactions)
* **Language:** Python (PySpark)
* **ML Library:** Spark MLlib
* **Security:** Key-based authentication (No hard-coded credentials).

---

## 📉 Data Processing Statistics

The pipeline successfully processed the entire dataset of **6.3 million records**. Below is the breakdown of how data flowed through the Medallion Architecture:

| Layer / Status | Row Count | Description |
| :--- | :--- | :--- |
| **Raw Ingestion** | **6,362,620** | Full dataset ingested from GCS. |
| **Others (Archived)** | 3,592,211 | Payment & Debit transactions (Out of scope). |
| **Silver (ML Ready)** | 2,770,409 | Valid TRANSFER & CASH_OUT transactions. |
| **Quarantine** | **0** | *Data Quality Check Passed.* No records violated the "Non-Negative Amount" rule, confirming source data integrity. |

**Model Sampling Strategy:**
To optimize resource usage on the Community Edition while maintaining model performance, a **Strategic Sampling** method was applied:
* **Training Set:** 114,547 rows (100% of Fraud + 5% of Normal).
* **Test Set:** 31,448 rows (Unseen future data for evaluation).

---

## 🚀 Key Features Implementation

### 1. Robust Data Engineering
* **Delta Lake Integration:** All layers utilize Delta Lake format for Schema Enforcement, Audit History, and ACID guarantees.
* **Secure Ingestion:** Implemented a secure file-based key authentication mechanism, ensuring no sensitive credentials are exposed in the codebase.

### 2. Advanced Feature Engineering
I engineered specific features to capture "thief behavior":
* **`amountRatio`**: *(Amount / OldBalance)* - Detects "account emptying" behavior (Fraudsters often drain the exact remaining balance).
* **`errorBalanceOrig`**: *(NewBal - (OldBal - Amount))* - Identifies mathematical anomalies in the origin account.
* **`hourOfDay`**: Extracts transaction time patterns.

### 3. Professional ML Strategy
* **Strict Time-Series Split:** Instead of random splitting, data was split by time step (Train on Past, Test on Future) to realistically simulate production environments and prevent **Data Leakage**.
* **Strategic Sampling:** To handle extreme class imbalance and optimize resource usage, I trained on **100% of Fraud cases** while downsampling **Normal cases to 5%**, creating a balanced learning environment.

---

## 📊 Model Performance & Results

The Random Forest model was evaluated on an unseen test set of **31,448 transactions** (representing the "Future" timeline).

### 🎯 Key Metrics

| Metric | Score | Interpretation |
| :--- | :--- | :--- |
| **Precision** | **99.93%** | **High Confidence.** When the model predicts fraud, it is correct 99.93% of the time (Only 3 False Positives). |
| **Recall** | **99.98%** | **High Detection.** The model caught nearly every single fraudster in the test set (Missed only 1). |
| **F1-Score** | **0.9995** | Perfect balance between Precision and Recall. |

*(Note: These results are based on the latest run using the optimized feature set including `errorBalance` and `amountRatio`.)*

### 🔍 Confusion Matrix Breakdown

The confusion matrix confirms the model's reliability in a realistic scenario:

| | Predicted: Normal | Predicted: Fraud |
| :--- | :---: | :---: |
| **Actual: Normal** | 27,187 (TN) | **3 (FP)** |
| **Actual: Fraud** | **1 (FN)** | **4,257 (TP)** |

* **Caught Fraud (TP):** 4,257 cases successfully detected.
* **Missed Fraud (FN):** Only 1 case missed.
* **False Alarms (FP):** Only 3 legitimate transactions blocked.

### 📈 Feature Importance
The analysis reveals that **`newbalanceOrig`** (account balance becoming zero) and **`amountRatio`** are the strongest predictors, confirming that "emptying the account" is the primary behavior of fraudsters in this dataset.

![Feature Importance](images/feature_importance.png)

---

## 💡 Conclusion

This project demonstrates a **Production-Ready** approach to fraud detection. By moving beyond simple "accuracy" and focusing on **Data Engineering best practices** (Delta Lake, Secure Ingestion) and **Scientific Rigor** (Time-Series Splitting), the pipeline achieves reliable and explainable results.

The use of **Delta Lake** ensures that the data foundation is solid, allowing the Machine Learning model to perform at its peak potential.
