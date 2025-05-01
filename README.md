# MyTomorrows
A case study for a scalable and secure clinet reporting for a growing patient Advocacy



# Task 1: Client-Facing Report – UnitedHealthcare

## 🎯 Objective

UnitedHealthcare needs a comprehensive reporting solution that:

- Standardizes treatment categories
- Highlights high-volume procedures and missed (uninsured) patients
- Enables secure data exports
- Supports visualization through Tableau without exposing PII

---

## 🧱 Data Modeling & Transformation

### ✅ Standardize Insurance Coverage & Treatment Categories

**View**: [`Z1_dim_procedure_category_mapping`](#z1_dim_procedure_category_mapping)

<details id="z1_dim_procedure_category_mapping">
<summary>SQL Logic</summary>

```sql
CREATE VIEW Z1_dim_procedure_category_mapping AS
SELECT 
  p.CODE AS procedure_code,
  p.DESCRIPTION,
  CASE
    WHEN LOWER(p.DESCRIPTION) LIKE '%assessment%' THEN 'Assessment'
    WHEN LOWER(p.DESCRIPTION) LIKE '%screening%' THEN 'Screening'
    WHEN LOWER(p.DESCRIPTION) LIKE '%dialysis%' THEN 'Chronic Care'
    WHEN LOWER(p.DESCRIPTION) LIKE '%injection%' THEN 'Medication Administration'
    WHEN LOWER(p.DESCRIPTION) LIKE '%therapy%' THEN 'Rehabilitation'
    WHEN LOWER(p.DESCRIPTION) LIKE '%chemotherapy%' THEN 'Oncology'
    WHEN LOWER(p.DESCRIPTION) LIKE '%colonoscopy%' THEN 'Diagnostic Procedure'
    WHEN LOWER(p.DESCRIPTION) LIKE '%examination%' THEN 'Diagnostic Procedure'
    WHEN LOWER(p.DESCRIPTION) LIKE '%biopsy%' THEN 'Diagnostic Procedure'
    WHEN LOWER(p.DESCRIPTION) LIKE '%ultrasound%' THEN 'Imaging'
    WHEN LOWER(p.DESCRIPTION) LIKE '%mammography%' THEN 'Imaging'
    WHEN LOWER(p.DESCRIPTION) LIKE '%echocardiography%' THEN 'Imaging'
    WHEN LOWER(p.DESCRIPTION) LIKE '%scan%' THEN 'Imaging'
    WHEN LOWER(p.DESCRIPTION) LIKE '%cardioversion%' THEN 'Surgical Procedure'
    WHEN LOWER(p.DESCRIPTION) LIKE '%ablation%' THEN 'Surgical Procedure'
    WHEN LOWER(p.DESCRIPTION) LIKE '%surgery%' THEN 'Surgical Procedure'
    WHEN LOWER(p.DESCRIPTION) LIKE '%vaccination%' THEN 'Preventive Care'
    WHEN LOWER(p.DESCRIPTION) LIKE '%test%' THEN 'Laboratory'
    WHEN LOWER(p.DESCRIPTION) LIKE '%measurement%' THEN 'Laboratory'
    ELSE 'Other'
  END AS standardized_group
FROM fct_procedures p;
```
</details>

---

### ✅ High-Volume Procedures & Missed Patients

**View**: [`C1_rpt_procedure_volume_uhc`](#c1_rpt_procedure_volume_uhc)

<details id="c1_rpt_procedure_volume_uhc">
<summary>SQL Logic</summary>

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
</details>

**View**: [`C2_rpt_high_volume_patients_uhc`](#c2_rpt_high_volume_patients_uhc)

<details id="c2_rpt_high_volume_patients_uhc">
<summary>SQL Logic</summary>

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
</details>

**View**: [`C4_rpt_uninsured_patients_all`](#c4_rpt_uninsured_patients_all)

<details id="c4_rpt_uninsured_patients_all">
<summary>SQL Logic</summary>

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
</details>

---

### ✅ Sorted Export by Procedure and Latest Date

**View**: [`C6_rpt_procedure_sorted_export_uhc`](#c6_rpt_procedure_sorted_export_uhc)

<details id="c6_rpt_procedure_sorted_export_uhc">
<summary>SQL Logic</summary>

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
</details>

---

### ✅ Tableau-Safe PII-Free Export

**View**: [`X2_rpt_tableau_export_masked`](#x2_rpt_tableau_export_masked)

<details id="x2_rpt_tableau_export_masked">
<summary>SQL Logic</summary>

```sql
CREATE VIEW X2_rpt_tableau_export_masked AS
SELECT 
  e.Id AS encounter_id,
  e.START,
  substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12) AS masked_patient_id,
  ...
  CASE
    WHEN LOWER(p.DESCRIPTION) LIKE '%test%' THEN 'Laboratory'
    WHEN LOWER(p.DESCRIPTION) LIKE '%measurement%' THEN 'Laboratory'
    ELSE 'Other'
  END AS standardized_group
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers pa ON e.PAYER = pa.Id
JOIN dim_organizations o ON e.ORGANIZATION = o.Id;
```
</details>

---

## 📊 Tableau Dashboard Design

**Filters**:
- Standardized Group
- Date Range
- Insurance Coverage

**Visuals**:
- KPI tiles: Total Encounters, High-Volume Patients, Uninsured Patients
- Bar chart: Volume by Procedure Group
- Stacked bar: Procedure Group × Insurance Status
- Exportable table view from `X2_rpt_tableau_export_masked`

---

## 🔐 Filters & Permissions

- **Row-Level Security**: Use user-org mapping in Tableau
- **SQL-Level**: Use only masked IDs and pre-joined, filtered views

---

## 🛡️ Summary Table

| Component       | Method                                     |
|----------------|--------------------------------------------|
| Anonymization  | `masked_patient_id` via deterministic hash |
| Grouping       | CASE logic in `Z1`, `X1`, `X2`             |
| Export         | Views `C6`, `X2`                           |
| Privacy        | No DOB, name, or direct PII                |

---


---

## 🧾 Summary Table Explained

| Component       | Method                                     |
|----------------|--------------------------------------------|
| **Anonymization**  | Masked patient identifiers using hash logic: `substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12)` – ensures patients are pseudonymous but consistent across records |
| **Grouping**       | Procedures are categorized using CASE logic into standard groups (e.g., Imaging, Oncology) via views like `Z1`, `X1`, and `X2` |
| **Export**         | Views like `C6` and `X2` are pre-filtered, anonymized, and sorted to deliver export-ready datasets for UnitedHealthcare |
| **Privacy**        | Direct identifiers like name and DOB are excluded entirely – only masked IDs are shown in Tableau and exports |

---

## 🧮 Snapshot of Tables and Views – What They Represent

### Base Tables (Fact + Dimension)
- **fct_procedures**: Raw procedure-level data (code, description, base cost, encounter ID)
- **fct_encounters**: Encounter metadata (start date, patient ID, claim cost, payer coverage)
- **dim_payers**: Insurance provider information (payer name, ID)
- **dim_organizations**: Organization or facility-level data
- **dim_procedure_category_mapping**: Mapping table to group procedures into categories

---

### Key Views for UnitedHealthcare Report

| View Name | Purpose |
|-----------|---------|
| **Z1_dim_procedure_category_mapping** | Defines grouping logic for raw procedure descriptions |
| **C1_rpt_procedure_volume_uhc** | Counts and cost summaries of procedures for UnitedHealthcare |
| **C2_rpt_high_volume_patients_uhc** | Flags patients with high numbers of visits under UHC |
| **C4_rpt_uninsured_patients_all** | Identifies patients without insurance coverage |
| **C6_rpt_procedure_sorted_export_uhc** | Provides export-ready data sorted by procedure and date |
| **X2_rpt_tableau_export_masked** | Final export view with masked patient IDs, procedure categories, and no PII |
| **V1_summary_uhc_dashboard** (optional) | Summary layer for Tableau performance (can be added) |

These views combine fact and dimension data, filter for UHC-specific logic, group treatments consistently, and ensure safe, compliant exports.

---

