# MyTomorrows
A case study for a scalable and secure clinet reporting for a growing patient Advocacy


# Task 1 - Client-Facing Report for UnitedHealthcare
# MODELLING

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

- These models clean and standardize raw inputs from source tables like:
  - fct_procedures
  - fct_encounters
  - dim_payers
- Basic SQL transformations include renaming, casting, and removing unnecessary fields

```sql
SELECT
  CODE AS procedure_code,
  DESCRIPTION AS procedure_description,
  BASE_COST::NUMERIC
FROM {{ source('raw', 'fct_procedures') }}
```

---

### - 2. Intermediate Layer (Join + Enrichment)

- Models join encounters and procedures with dimension tables (payers, orgs)
- Apply CASE logic to group procedures into categories such as 'Imaging', 'Oncology', etc.
- Create flags for uninsured patients and filter by payer name for UnitedHealthcare

```sql
SELECT
  p.CODE AS procedure_code,
  e.START AS procedure_date,
  pa.NAME AS payer_name,
  CASE
    WHEN LOWER(p.DESCRIPTION) LIKE '%therapy%' THEN 'Rehabilitation'
    ELSE 'Other'
  END AS standardized_group
FROM stg_fct_procedures p
JOIN stg_fct_encounters e ON p.ENCOUNTER = e.Id
JOIN stg_dim_payers pa ON e.PAYER = pa.Id
```

---

### - 3. Reporting Layer (Final Views or DBT Models)

- These models are directly used by stakeholders or Tableau:
  - C1_rpt_procedure_volume_uhc: volume and cost aggregation
  - C2_rpt_high_volume_patients_uhc: patients with frequent encounters
  - C6_rpt_procedure_sorted_export_uhc: export sorted by code and date
  - X2_rpt_tableau_export_masked: masked output ready for Tableau use

Each view is modular and follows naming conventions for clarity and reuse.

---

### - 4. Data Privacy and Security

- Patient identifiers are masked using deterministic hash logic:
```sql
substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12)
```
- All direct PII (e.g., name, DOB) is excluded from exports and Tableau views
- Role-based access can be layered via SQL views or Tableau Row-Level Security filters

---

### - Why This Structure

- Keeps logic modular and testable, ideal for DBT
- Makes it easy to audit and trace data from source to report
- Aligns with HIPAA data privacy standards
- Easily extensible to other clients or internal reports

---

## Objective

UnitedHealthcare needs a comprehensive reporting solution that:

- Standardizes treatment categories
- Highlights high-volume procedures and missed (uninsured) patients
- Enables secure data exports
- Supports visualization through Tableau without exposing PII

---

## Data Modeling & Transformation

[The SQL view logic with collapsible sections is intentionally omitted here to avoid duplication. See original linked markdown if needed.]

---

## Tableau Dashboard Design

- Filters: Standardized Group, Date Range, Insurance Coverage
- Visuals:
  - KPI tiles: Total Encounters, High-Volume Patients, Uninsured Patients
  - Bar chart: Volume by Procedure Group
  - Stacked bar: Procedure Group × Insurance Status
  - Exportable table view from `X2_rpt_tableau_export_masked`

---

## Filters & Permissions

- Row-Level Security: User-organization mapping in Tableau
- SQL-Level: Masked IDs and pre-filtered views only

---

## Summary Table

| Component       | Method                                     |
|----------------|--------------------------------------------|
| Anonymization  | `masked_patient_id` via deterministic hash |
| Grouping       | CASE logic in `Z1`, `X1`, `X2`             |
| Export         | Views `C6`, `X2`                           |
| Privacy        | No DOB, name, or direct PII                |

---

## Summary Table Explained

| Component       | Method                                     |
|----------------|--------------------------------------------|
| Anonymization  | Uses hash logic: `substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12)` – ensures pseudonymity |
| Grouping       | Procedures grouped using CASE logic into categories like Imaging, Oncology |
| Export         | Views like `C6` and `X2` are filtered, anonymized, and sorted |
| Privacy        | No names, DOBs, or patient-identifying fields shown |

---

## Snapshot of Tables and Views – What They Represent

### Base Tables (Fact + Dimension)

- fct_procedures: procedure data (code, cost)
- fct_encounters: patient visits with timestamps and payer info
- dim_payers: payer metadata
- dim_organizations: facility metadata
- dim_procedure_category_mapping: maps raw procedures to groups

### Key Views

| View Name | Purpose |
|-----------|---------|
| Z1_dim_procedure_category_mapping | Grouping logic for procedures |
| C1_rpt_procedure_volume_uhc | Volume and cost by procedure |
| C2_rpt_high_volume_patients_uhc | High-visit patients |
| C4_rpt_uninsured_patients_all | Patients without insurance |
| C6_rpt_procedure_sorted_export_uhc | Export-ready data sorted |
| X2_rpt_tableau_export_masked | Tableau-safe, PII-free output |

------



