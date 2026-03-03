# DEGO Project - Team 9
## Team Members
- Selin Gencer 71414
- Miguel Xu 56323
- Bernardo Baptista 56783
## Project Description
Credit scoring bias analysis for DEGO course .
## Structure
- ‘ data /‘ - Dataset files
- ‘ notebooks /‘ - Jupyter analysis notebooks
- ‘ src /‘ - Python source code
- ‘ reports /‘ - Final deliverables

## Week 23-02-2026/28-02-2026
- Summary: Finished the data quality acessment and started to work on bias detection.

## 📊 Data Quality Analysis (DQA)

### Executive Summary
We audited **502 raw credit application records** from NovaCred’s lending pipeline across five key dimensions: *Completeness, Consistency, Validity, Uniqueness,* and *Timeliness*. After resolving **2 exact duplicate records**, the cleaned dataset contains **500 unique applications**.

> [!IMPORTANT]
> **34.2% (171 records)** carry at least one quality flag. These records require downstream attention (filtering or manual review) before use in risk modeling. All original values are preserved alongside cleaned fields and audit flags to maintain a 100% verifiable trail.

---

### 🔍 Key Findings

#### **1. Consistency (High Impact)**
* **Date of Birth:** 31.4% (157 records) stored in non-ISO formats (e.g., DD/MM/YYYY) — standardized to `YYYY-MM-DD` in `date_of_birth_clean`.
* **Gender Coding:** 22.8% (114 records) used inconsistent coding (M/F vs. Male/Female) — normalized to `Male/Female/Unknown` in `gender_clean`.
* **Data Types:** 8 income records were stored as strings instead of numeric values — coerced in `annual_income_clean`.

#### **2. Completeness (Governance Concern)**
* **Audit Trail Gap:** `processing_timestamp` is missing in **87.65%** of records. This indicates a lack of a reliable audit trail for historical credit decisions.
* **PII Missingness:** Minor gaps (~1.0%) identified in Email, SSN, IP Address, DOB, and Annual Income.

#### **3. Uniqueness & Validity (High Risk)**
* **Identity Fraud Flags:** 4 duplicate SSNs were flagged across different applicants. These were retained for investigation rather than deleted, as they may indicate identity fraud or data entry errors.
* **Financial Anomalies:** Identified 1 case of zero annual income, 1 Debt-to-Income (DTI) ratio $> 1$, and 1 negative savings balance.
* **Temporal Logic:** 2 processing timestamps were dated in the future, flagged as unreliable for time-series analysis.

---

### 🛠 Cleaning & Transformation Approach
Our pipeline follows a **deterministic and non-destructive** methodology:
1. **Preservation:** Original raw fields are never overwritten.
2. **Standardization:** Cleaned/formatted values are written to new `*_clean` columns.
3. **Auditability:** Specific quality issues are recorded in boolean `flag_` columns, allowing downstream analysts to apply context-appropriate filters.

#### **Data Schema Mapping**
| Feature | Row Count | Grain | Note |
| :--- | :--- | :--- | :--- |
| **Applications** | 500 | Applicant | Unique records after deduplication. |
| **Spending** | 827 | Transaction | Flattened "one-to-many" spending behavior records. |

---

### 📂 Pipeline Outputs
* `applications_clean.csv`: The primary cleaned dataset (500 rows).
* `spending_clean.csv`: Flattened spending history for granular analysis (827 rows).
* `data_quality_report.csv`: Row-level summary of all flags triggered.
* `cleaning_log.csv`: Technical log of every transformation applied.