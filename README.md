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

## Data Engineering - Data Quality Pipeline

### Scope
The Data Engineer built the ingestion, normalization, data-quality audit, and cleaning pipeline for ⁠ raw_credit_applications.json ⁠.

### What We Implemented
1.⁠ ⁠Loaded the nested raw JSON as immutable source data.
2.⁠ ⁠Flattened data into:
   - application-level table (one row per application)
   - spending-level table (one row per spending item)
3.⁠ ⁠Audited data quality issues across:
   - completeness
   - consistency
   - validity
4.⁠ ⁠Applied deterministic cleaning rules and added row-level quality flags.
5.⁠ ⁠Exported cleaned datasets and audit artifacts for downstream bias/privacy analysis.

### Key Findings (from profiling)
•⁠  ⁠Total records: ⁠ 502 ⁠
•⁠  ⁠Duplicate IDs(uniqueness): ⁠ 2 ⁠
• Timeliness: 2
•⁠  ⁠Missing values:
  - email  7 ⁠
  - SSN ⁠ 5 ⁠
  - IP address ⁠ 5 ⁠
  - gender ⁠ 3 ⁠
  - date of birth ⁠ 5 ⁠
  - ZIP code ⁠ 2 ⁠
  - annual income ⁠ 5 ⁠
  - processing timestamp ⁠ 440 ⁠
•⁠  ⁠Type inconsistencies:
  - ⁠ annual_income ⁠ stored as string in ⁠ 8 ⁠ rows
  - ⁠ annual_income ⁠ stored as float in ⁠ 1 ⁠ row
•⁠  ⁠Format inconsistencies:
  - non-ISO DOB values: ⁠ 157 ⁠
  - invalid emails: ⁠ 4 ⁠
  - inconsistent gender coding (⁠ Male/Female/M/F/blank ⁠)
•⁠  ⁠Invalid numeric values:
  - negative credit history months: ⁠ 2 ⁠
  - debt-to-income ratio > 1: ⁠ 1 ⁠
  - negative savings balance: ⁠ 1 ⁠
•⁠  Accuracy:
  - 66.3% of data is clean (333 rows), while 33.7% requires fixes (169 rows).


### Cleaning Rules Applied
•⁠  ⁠Gender normalization: ⁠ M/F/Male/Female ⁠ -> ⁠ Male/Female ⁠, others -> ⁠ Unknown ⁠
•⁠  ⁠DOB normalization: parse mixed formats and standardize to ⁠ YYYY-MM-DD (31.3% of the data required Date of Birth normalization)
•⁠  ⁠Numeric coercion for income/credit history/DTI/savings
•⁠  ⁠Quality flags for duplicates and invalid values

### Outputs
•⁠  ⁠⁠ data/processed/applications_clean.csv ⁠
•⁠  ⁠⁠ data/processed/spending_clean.csv ⁠
•⁠  ⁠⁠ data/processed/data_quality_report.csv ⁠
•⁠  ⁠⁠ data/processed/cleaning_log.csv ⁠
