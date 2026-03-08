# DEGO Project - Team 9 (TXA)

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Repository Structure](#repository-structure)
3. [Team & Roles](#team--roles)
4. [Dataset Overview](#dataset-overview)
5. [Data Quality Findings](#data-quality-findings)
6. [Bias Analysis Results](#bias-analysis-results)
7. [Privacy Assessment](#privacy-assessment)
8. [GDPR & EU AI Act Compliance](#gdpr--eu-ai-act-compliance)
9. [Governance Recommendations](#governance-recommendations)
10. [How to Run](#how-to-run)

## Executive Summary

NovaCred is a fintech startup that uses machine learning to automate credit decisions. Following a regulatory inquiry into potential discrimination in its lending practices, our team was engaged as a **Data Governance Task Force** to audit the data pipeline end to end.

Our analysis of **502 raw credit applications** uncovered significant issues across three dimensions:

- **Data Quality:** 13 distinct quality issues were identified and quantified, spanning all five data quality dimensions (completeness, consistency, validity, uniqueness, and timeliness). 500 of 502 records were retained after cleaning, with issues flagged transparently rather than silently removed.

- **Algorithmic Bias:** A **Disparate Impact ratio of 0.77** was found for gender which is below the 0.8 legal threshold (four-fifths rule), indicating statistically significant disadvantage for female applicants. Age-based patterns were also identified, with applicants under 35 showing notably lower approval rates. Income and credit history were flagged as likely **proxy variables** for both gender and age.

- **Privacy & Governance:** Four direct identifiers (full name, email, SSN, IP address) are stored in plain text with no protection. Critical GDPR governance fields like including consent timestamps, retention periods, processing purpose, and data source are entirely absent. The system qualifies as a **High-Risk AI system** under the EU AI Act (credit scoring), triggering strict obligations that are currently unmet.

**Bottom line:** NovaCred's data pipeline contains material data quality, fairness, and regulatory compliance failures that require immediate remediation before the system can be considered fit for regulated use.

## Repository Structure
```
dego-project-team9/
├── README.md
├── data/
│   ├── raw/
│   │   └── raw_credit_applications.json
│   └── processed/
│       ├── applications_clean.csv
│       ├── cleaning_log.csv
│       ├── data_quality_report.csv
│       └── spending_clean.csv
├── notebooks/
│   ├── 01-data-quality.ipynb
│   ├── 02-bias-analysis.ipynb
│   └── 03-privacy-demo.ipynb 
└── presentation/
```

## Team & Roles 
| Role | Name | Primary Contribution |
| :--- | :--- | :--- |
| **Data Engineer** | Selin Gencer | Data loading, cleaning pipeline, repository structure |
| **Data Scientist** | Matilde Ferreira | Bias analysis, fairness metrics, statistical testing, Proxy Variable Identification, Intersectional Bias Analysis. |
| **Governance Officer** | Bernardo Baptista | GDPR mapping, policy recommendations, compliance analysis |
| **Product Lead** | Miguel Xu | Documentation, governance narrative, project coordination, Presentation, README documentation |

## Dataset Overview

- **Source file:** `raw_credit_applications.json`
- **Total records:** 502 (nested JSON, flattened for analysis)
- **Structure:** Each record contains applicant info, financial data, spending behavior (array), and a loan decision

| Field Group | Key Fields |
|-------------|-----------|
| Identity | `_id`, `full_name`, `email`, `ssn`, `ip_address` |
| Demographics | `gender`, `date_of_birth`, `zip_code` |
| Financials | `annual_income`, `credit_history_months`, `debt_to_income`, `savings_balance` |
| Decision | `loan_approved`, `interest_rate`, `approved_amount`, `rejection_reason` |
| Metadata | `processing_timestamp`, `loan_purpose`, `notes` |

---

## Data Quality Analysis (DQA)

**Notebook:** `01-data-quality.ipynb`

### Cleaning Strategy: Flag & Retain

500/502 final records are retained. Rather than silently dropping rows, original values are preserved and quality issues are recorded as boolean `flag_` columns. Cleaned values are stored with a `_clean` suffix. This approach supports both audit trails and flexible downstream filtering.

> [!IMPORTANT]
> **34.2% (171 records)** carry at least one quality flag. These records require downstream attention (filtering or manual review) before use in risk modeling. All original values are preserved alongside cleaned fields and audit flags to maintain a 100% verifiable trail.

### Issues Identified by Dimension

#### Uniqueness

| Issue | Records Affected |
|-------|-----------------|
| Duplicate `_id` (application ID) | 4 rows (2 pairs) |
| Duplicate SSN (non-empty only) | 6 rows (3 pairs) |

**Duplicate `_id` resolution:**
- `app_001`: Kept row 383 (complete record); dropped row 455 (`notes = DUPLICATE_ENTRY_ERROR`, missing SSN/IP/DOB/ZIP/gender)
- `app_042`: Kept row 354 (`notes = RESUBMISSION`, corrected submission); dropped row 8

**Duplicate SSN handling:** Three SSN pairs were found. One pair corresponds to the `app_001` duplicate (resolved above). The other two pairs appear under different applicant names and IDs which may indicate **identity fraud or data entry errors** and were **retained and flagged** for case-by-case investigation rather than silently deleted.

#### Completeness

| Field | Missing / Blank |
|-------|----------------|
| `processing_timestamp` | ~88% of records — **critical audit trail gap** |
| `ssn`, `date_of_birth`, `ip_address`, `annual_income` | ~5 records each |
| `gender` | 3 records |
| `zip_code` | 2 records |

> Note: `decision.interest_rate` and `decision.approved_amount` are structurally null for denied applications; `decision.rejection_reason` is structurally null for approved ones — these are not flagged as quality issues.

#### Consistency

| Issue | Records Affected |
|-------|-----------------|
| `gender` encoded inconsistently (M / Male / F / Female / blank) | 114 records |
| `date_of_birth` in mixed formats (ISO, DD/MM/YYYY, YYYY/MM/DD) | 157 records |
| `annual_income` stored as string instead of numeric | 8 records |
| Credit history implies start before age 18 | 1 records |

#### Validity

| Issue | Records Affected |
|-------|-----------------|
| `annual_income == 0` | 1 records |
| `credit_history_months < 0` (values: −10, −3) | 2 records |
| `debt_to_income > 1` | 1 records |
| `savings_balance < 0` (value: −5,000) | 1 record |
| Invalid email format | 4 records |

#### Timeliness

| Issue | Records Affected |
|-------|-----------------|
| `processing_timestamp` set in the future (post 2026-03-02) | 2 records |

### Cleaning Rules Applied

| Field | Rule | Handling |
|-------|------|----------|
| `gender` | Normalize M/F → Male/Female; else Unknown | New `gender_clean` column |
| `date_of_birth` | Parse all formats → YYYY-MM-DD | New `date_of_birth_clean` column |
| `annual_income` | Coerce string → numeric; zero → `pd.NA` | New `annual_income_clean` column |
| `credit_history_months` | Negative → `pd.NA` | New `credit_history_months_clean` column |
| `debt_to_income` | > 1 → `pd.NA` | New `debt_to_income_clean` column |
| `savings_balance` | Negative → `pd.NA` | New `savings_balance_clean` column |

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

## Bias Analysis Results

> **Notebook:** `02-bias-analysis.ipynb`

### Gender Disparate Impact

| Group | Approval Rate |
|-------|--------------|
| Male | ~66% |
| Female | ~51% |

**Disparate Impact Ratio = 0.7667 → FAIL**

A DI ratio of 0.77 falls below the 0.8 threshold of the four-fifths rule, indicating potential disparate impact against female applicants. A chi-square test confirmed this gap is **statistically significant (p < 0.05)**.

### Age-Based Discrimination Patterns

| Age Group | Approval Rate |
|-----------|--------------|
| 18–25 | Below average |
| 26–35 | ~52.5% |
| 36–45 | ~67.9% |
| 46–55 | Above average |
| 56–65 | Above average |

The most pronounced gap occurs between the 26–35 and 36–45 groups. Using 35 as the cutoff, the approval rate jumps from ~52% (≤35) to ~68% (>35). The age **DI ratio = 0.753 → FAIL** (below the 0.8 threshold), and a chi-square test confirmed the gap is **statistically significant (p = 0.0006)**, providing strong evidence of age-based discrimination against applicants aged 35 and under.

### Proxy Variable Analysis

**Gender proxies:** T-tests found **no statistically significant difference** in income, credit history, debt-to-income, or savings between male and female applicants (all p > 0.05). This means financial variables are **not acting as gender proxies** which means that men and women in this dataset have similar financial profiles on average. This is a critical finding: the gender gap in approval rates **cannot be explained by financial differences**, which strengthens the case for **direct gender bias** in the model.

**Age proxies:** Pearson correlations showed that **income**, **credit history**, and **savings balance** all correlate meaningfully with age (|r| > 0.2, p < 0.05). These variables likely act as proxies for age in the model.

### Interaction Effects

## Gender x Age

A gender × age group approval rate matrix revealed that the lowest approval rates are concentrated among **young female applicants (18–35)**, suggesting a compounding disadvantage at the intersection of both characteristics. The largest gender gap occurs in the **26–35 group (21.9% difference)**, while both genders face significant age-based disadvantage in the 18–25 group. Single-attribute bias analysis would understate the actual risk.

### Gender × Income Level Interaction

Since financial variables were found to be non-significant gender proxies, a further test examined whether men and women at the **same income level** are approved at the same rate. Applicants were split into three equal income bands (low, mid, high) and approval rates compared by gender within each band.

| Income Level | Male Approval Rate | Female Approval Rate | Gap |
|---|---|---|---|
| Low income | ~53.6% | ~35.4% | ~18.2pp |
| Mid income | ~64.1% | ~52.5% | ~11.6pp |
| High income | ~72.5% | ~62.7% | ~9.8pp |

The gender gap **persists across all income levels** (p = 0.001), confirming the interaction is statistically significant. Even when women earn the same amount as men, their approval rate is still lower. This confirms that the discrimination **cannot be explained by income differences** and is consistent with direct gender bias in NovaCred's lending model.


## Privacy Assessment

> **Notebook:** `03-privacy-demo.ipynb`

### Direct PII Identifiers — stored in plain text

| Field | Risk |
|-------|------|
| `full_name` | Directly identifies the individual |
| `email` | Unique contact point per individual |
| `ssn` | National unique identifier |
| `ip_address` | Linkable to an individual via ISP |

### Quasi-Identifiers (Indirect PII) — Re-identification Risk

| Combination | Unique Individuals | Risk Level |
|-------------|-------------------|------------|
| `date_of_birth` alone | **489 / 502** | Critical |
| `zip_code` alone | 51 / 502 | Moderate |
| `gender` alone | 0 / 502 | Low |
| `dob + zip` | **499 / 502** | Critical |
| `dob + gender` | 495 / 502 | Critical |
| `zip + gender` | 181 / 502 | High |
| `dob + zip + gender` | **499 / 502** | Critical |

Using only three quasi-identifiers, **99.4% of applicants can be uniquely re-identified** without access to any direct identifiers.

### Pseudonymization Demonstration - Privacy protection

SHA-256 hashing was applied to `full_name`, `email`, `ssn`, and `ip_address`.

**Vulnerability identified:** Deterministic hashing without a salt is reversible via brute force. An attacker who knows the input space (e.g., IP address ranges) can reproduce the exact hash and recover the original value where we demonstrated in the notebook.

**Fix applied:** Salted hashing using `secrets.token_hex(16)`. The salt is generated per-record, stored separately, and prepended before hashing. This makes reversal computationally infeasible.

---

## GDPR & EU AI Act Compliance

> **Notebook:** `03-privacy-demo.ipynb`

### GDPR Article 5 Mapping

| Article | Principle | Status | Key Finding |
|---------|-----------|:------:|-------------|
| 5(1)(a) | Lawfulness, Fairness & Transparency | FAIL | No consent timestamp or processing purpose; DI = 0.77; rejection reason is non-explanatory |
| 5(1)(b) | Purpose Limitation | FAIL | No `processing_purpose` or `data_source` fields |
| 5(1)(c) | Data Minimisation | FAIL | `ip_address`, `full_name`, `spending_behavior` not necessary for credit assessment |
| 5(1)(d) | Accuracy | FAIL | Mixed date formats, invalid financial values, inconsistent gender encoding |
| 5(1)(e) | Storage Limitation | FAIL | No `retention_until` or `consent_timestamp` — deletion enforcement impossible |
| 5(1)(f) | Integrity & Confidentiality | FAIL | Direct identifiers stored in plain text without pseudonymization |
| 5(2) | Accountability | FAIL | No `reviewer_id`, `decision_timestamp`, or `model_version` |

### EU AI Act — High-Risk Classification

NovaCred's system qualifies as a **High-Risk AI system** under Annex III (credit scoring). Current compliance gaps:

| Requirement | Status |
|-------------|:------:|
| Data Governance & Quality | FAIL |
| Human Oversight | FAIL |
| Record Keeping & Logging | FAIL |
| Accuracy & Robustness | FAIL |
| Cybersecurity | FAIL |

---

## Governance Recommendations

### 1. Input Validation & Format Standardization

Implement automated validation at the point of data entry to enforce a single gender encoding, a single date format (YYYY-MM-DD), and logical range checks on financial fields (no negative savings, no negative credit history, DTI ≤ 1).

**Regulatory basis:** GDPR Article 5(1)(d) — Accuracy

### 2. Data Minimisation

Remove `ip_address`, `full_name`, and `spending_behavior` from the dataset. These fields are not necessary for creditworthiness assessment and their collection exceeds the stated processing purpose.

**Regulatory basis:** GDPR Article 5(1)(c) — Data Minimisation

### 3. Add Governance Metadata Fields

| Field | Purpose |
|-------|---------|
| `consent_timestamp` | Records when and how consent was obtained |
| `retention_until` | Defines the deletion deadline per record |
| `data_source` | Documents the origin of the data |
| `processing_purpose` | Justifies the lawful basis for processing |

**Regulatory basis:** GDPR Articles 5(1)(a), 5(1)(b), 5(1)(e)

### 4. Meaningful Decision Transparency

Replace the rejection reason `algorithm_risk_score` with structured, human-readable explanations such as `insufficient_income`, `high_debt_to_income_ratio`, `short_credit_history`, or `low_savings_balance`. Include `model_version` in the decision record to support traceability.

**Regulatory basis:** GDPR Article 5(1)(a) — Transparency; EU AI Act — Explainability

### 5. Human Oversight Fields

Add the following fields to document human review of model decisions:

| Field | Purpose |
|-------|---------|
| `reviewer_id` | Identifies the human reviewer |
| `decision_override` | Records if a human changed the model's output |
| `review_timestamp` | When the review occurred |

**Regulatory basis:** EU AI Act — Human Oversight for High-Risk systems

### 6. Bias Monitoring

Establish a recurring bias audit (at minimum quarterly) computing disparate impact ratios across gender, age group, and their intersections. Any DI ratio below 0.8 should trigger mandatory review before the next model deployment.

**Regulatory basis:** GDPR Article 5(1)(a) — Fairness; EU AI Act — Risk Management

### 7. Audit Logging

Implement system-level logging to capture:

| Log Type | Fields |
|----------|--------|
| Access logs | Who accessed which records and when |
| Model versioning | `model_version` tied to each decision |
| Decision audit | `decision_override` and rationale for overrides |

**Regulatory basis:** GDPR Article 5(2) — Accountability; EU AI Act — Record Keeping

---

## How to Run

### Prerequisites

```bash
pip install pandas numpy matplotlib seaborn scipy
```

### Execution Order

Run notebooks in sequence — each depends on outputs from the previous:

```
01-data-quality.ipynb   →  generates processed CSVs in data/processed/
02-bias-analysis.ipynb  →  reads applications_clean.csv
03-privacy-demo.ipynb   →  reads raw_credit_applications.json
```

### Notes

- All notebooks run without errors from top to bottom
- The cleaning pipeline in `01-data-quality.ipynb` is fully deterministic — the same input always produces the same output
- No external data sources are used; only the provided `raw_credit_applications.json`



*DEGO 2606 | MSc Business Analytics | Nova SBE | Team 9 – TXA |
