# MyTomorrows (private mode)
A case study for scalable and secure client reporting for a growing Patient Advocacy Group

---

# Task 1 - Client-Facing Report for UnitedHealthcare

## Summary of the Assignment

UnitedHealthcare, as a client of a Patient Advocacy Group, requested a secure and standardized report that helps them:

- Understand and group procedures into consistent categories
- Identify high-volume procedures
- Flag missed patients, especially those without insurance
- Export the data sorted by procedure and latest date
- Receive insights via Tableau dashboards without exposing PII

This report outlines how the data was modeled, transformed, and secured to meet these requirements.

---

## How I Would Model and Transform the Data (DBT or SQL Logic)

### - 1. Source Layer (Staging Models)

These models clean and standardize raw inputs from source tables. I used direct SQL views to prepare these. 
See this SQL from `A2_rpt_encounter_procedure_flat_ymd`:

```sql
CREATE VIEW A2_rpt_encounter_procedure_flat_ymd AS
SELECT 
  e.Id AS encounter_id,
  STRFTIME('%Y-%m-%d', e.START) AS START,
  e.PATIENT,
  e.DESCRIPTION AS encounter_description,
  p.CODE AS procedure_code,
  p.DESCRIPTION AS procedure_description,
  p.BASE_COST,
  pa.NAME AS payer_name,
  o.NAME AS org_name
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers pa ON e.PAYER = pa.Id
JOIN dim_organizations o ON e.ORGANIZATION = o.Id;
```

This helps unify and format raw procedure and encounter data for all reporting views.

---

### - 2. Intermediate Layer (Join + Enrichment)

I enriched and grouped procedures using CASE logic to classify them into standardized categories. 
Example from `X1_rpt_encounter_procedure_with_grouping`:

```sql
CREATE VIEW X1_rpt_encounter_procedure_with_grouping AS
SELECT 
  e.Id AS encounter_id,
  e.START,
  e.PATIENT,
  p.CODE AS procedure_code,
  p.DESCRIPTION AS procedure_description,
  p.BASE_COST,
  pa.NAME AS payer_name,
  o.NAME AS org_name,
  CASE
    WHEN LOWER(p.DESCRIPTION) LIKE '%therapy%' THEN 'Rehabilitation'
    WHEN LOWER(p.DESCRIPTION) LIKE '%ultrasound%' THEN 'Imaging'
    WHEN LOWER(p.DESCRIPTION) LIKE '%biopsy%' THEN 'Diagnostic Procedure'
    ELSE 'Other'
  END AS standardized_group
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers pa ON e.PAYER = pa.Id
JOIN dim_organizations o ON e.ORGANIZATION = o.Id;
```

This makes the dataset easier to analyze by consolidating terminology.

---

### - 3. Reporting Layer (Final Views or DBT Models)

These are outputs used by Tableau or for client exports.

- **Volume report for UnitedHealthcare** – `C1_rpt_procedure_volume_uhc`:

```sql
CREATE VIEW C1_rpt_procedure_volume_uhc AS
SELECT
  p.CODE AS procedure_code,
  p.DESCRIPTION AS procedure_description,
  COUNT(*) AS procedure_count,
  SUM(p.BASE_COST) AS total_base_cost
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers py ON e.PAYER = py.Id
WHERE py.NAME = 'UnitedHealthcare'
GROUP BY p.CODE, p.DESCRIPTION;
```

- **High-volume patients** – `C2_rpt_high_volume_patients_uhc`:

```sql
CREATE VIEW C2_rpt_high_volume_patients_uhc AS
SELECT 
  e.PATIENT,
  COUNT(*) AS total_visits
FROM fct_encounters e
JOIN dim_payers py ON e.PAYER = py.Id
WHERE py.NAME = 'UnitedHealthcare'
GROUP BY e.PATIENT
HAVING COUNT(*) > 50
ORDER BY total_visits DESC;
```

- **Uninsured patients** – `C4_rpt_uninsured_patients_all`:

```sql
CREATE VIEW C4_rpt_uninsured_patients_all AS
SELECT 
  e.PATIENT,
  COUNT(*) AS uninsured_visits,
  SUM(e.TOTAL_CLAIM_COST) AS total_cost
FROM fct_encounters e
JOIN dim_payers py ON e.PAYER = py.Id
WHERE LOWER(py.NAME) LIKE '%insurance%'
GROUP BY e.PATIENT
ORDER BY uninsured_visits DESC;
```

- **Sorted export** – `C6_rpt_procedure_sorted_export_uhc`:

```sql
CREATE VIEW C6_rpt_procedure_sorted_export_uhc AS
SELECT 
  e.PATIENT,
  p.CODE AS procedure_code,
  p.DESCRIPTION AS procedure_description,
  p.BASE_COST,
  e.START AS procedure_date,
  e.TOTAL_CLAIM_COST,
  e.PAYER_COVERAGE
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers py ON e.PAYER = py.Id
WHERE py.NAME = 'UnitedHealthcare'
ORDER BY p.CODE, procedure_date DESC;
```

---

### - 4. Data Privacy and Security

Patient identifiers are anonymized before export using consistent hashing logic:

```sql
substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12) AS masked_patient_id
```

Example from `X2_rpt_tableau_export_masked`:

```sql
CREATE VIEW X2_rpt_tableau_export_masked AS
SELECT 
  e.Id AS encounter_id,
  e.START,
  substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12) AS masked_patient_id,
  p.CODE AS procedure_code,
  p.DESCRIPTION AS procedure_description,
  CASE
    WHEN LOWER(p.DESCRIPTION) LIKE '%therapy%' THEN 'Rehabilitation'
    ELSE 'Other'
  END AS standardized_group
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers pa ON e.PAYER = pa.Id
JOIN dim_organizations o ON e.ORGANIZATION = o.Id;
```

This ensures Tableau visuals are compliant with HIPAA and do not expose PII.

---

### - Why This Structure

- Each layer is clean, logical, and reusable
- Modular views make debugging and extension easier
- Ideal for DBT workflows with lineage tracking
- Fully privacy-compliant and production-ready

------

