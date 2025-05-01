
# 💊 MyTomorrows – Scalable & Secure Client Reporting  
_A case study for a growing Patient Advocacy Group_

---

## 🎯 Objective  
Design a scalable, secure, and privacy-compliant reporting workflow to support both internal and client-facing analytics in a healthcare setting.

---

## 📌 Deliverables Naming Convention  

Each deliverable is named as required:  
- Task 1 → `D1.A`, `D1.B`, etc.  
- Task 2 → `D2.A`, `D2.B`, etc.

---

## 🧰 Tooling Overview

| Layer         | Tools Used                               |
|---------------|-------------------------------------------|
| Data Storage  | SQLite (sandbox), PostgreSQL (planned)   |
| Transformation| Manual SQL Views, DBT (proposed)         |
| Orchestration | Airflow (planned)                        |
| Visualization | Tableau (Excel workaround for local dev) |

---
## 🗺️ Entity Relationship Diagram

The diagram below illustrates the structure used in the data model:

![Entity Relationship Diagram](data/erd_diagram.png)



---

## 🧠 Task 1: Build a Client-Facing Report (United Healthcare)

UnitedHealthcare requested a secure, standardized view of their patient data. The objectives were:

- Standardize insurance coverage for treatments (group similar procedures)  
- Identify high-volume procedures and patients missed (especially uninsured)  
- Export the data sorted by procedure and latest procedure date  
- Receive data visualizations via Tableau, without exposing patient PII  

---

# 🧩 D1.A – How I Would Model and Transform the Data

<details>
<summary><strong>1️⃣ Source Layer – Staging Models</strong></summary>

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

</details>

<details>
<summary><strong>2️⃣ Intermediate Layer – Grouping Logic</strong></summary>

```sql
CREATE VIEW X1_rpt_encounter_procedure_with_grouping AS
SELECT 
  e.Id AS encounter_id,
  e.START,
  e.PATIENT,
  p.CODE AS procedure_code,
  p.DESCRIPTION AS procedure_description,
  p.BASE_COST,
  e.PAYER,
  pa.NAME AS payer_name,
  o.NAME AS org_name,
  CASE
    WHEN LOWER(p.DESCRIPTION) LIKE '%assessment%' THEN 'Assessment'
    WHEN LOWER(p.DESCRIPTION) LIKE '%screening%' THEN 'Screening'
    -- (remaining cases shortened for brevity)
    ELSE 'Other'
  END AS standardized_group
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers pa ON e.PAYER = pa.Id
JOIN dim_organizations o ON e.ORGANIZATION = o.Id;
```

</details>

<details>
<summary><strong>3️⃣ Reporting Layer – Final Views</strong></summary>

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

<details>
<summary><strong>4️⃣ Privacy & Masking Logic</strong></summary>

```sql
substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12) AS masked_patient_id
```

</details>

---


---

## ✅ Why This Structure?

- Each layer is clean, logical, and reusable  
- Modular views make debugging and future development easier  
- Mimics DBT-style transformations, enabling lineage tracking  
- Designed with privacy and export-readiness in mind  
- Ideal for use in Tableau or other BI tools  

---

# 📊 D1.B – Tableau Dashboard Mockup Plan

<details>
<summary><strong>📋 Filters</strong></summary>

- Procedure Group  
- Date Range  
- Insurance Coverage  

</details>

<details>
<summary><strong>📌 KPI Tiles</strong></summary>

- Total Encounters  
- High-Volume Patients  
- Uninsured Patients  

</details>

<details>
<summary><strong>📊 Bar Chart</strong></summary>

- X-axis: Procedure Group  
- Y-axis: Encounter Count  

</details>

<details>
<summary><strong>📈 Line Chart</strong></summary>

- X-axis: Procedure Date  
- Y-axis: Monthly Encounter Volume  

</details>

<details>
<summary><strong>📄 Table Export</strong></summary>

- Procedure Group  
- Cost  
- Procedure Date  
- Organization Name  
- Payer Name  

</details>

---

# 🔐 D1.C – Filters and Permissions for Secure Client Access

<details>
<summary><strong>🧮 SQL-Level Row-Level Security</strong></summary>

```sql
WHERE client = 'United Healthcare'
```

Apply via:
- Secure View in PostgreSQL  
- DBT Model logic  

</details>

<details>
<summary><strong>📊 Tableau-Level Permissions</strong></summary>

- Assign users to group: `Client_UHC`  
- Filter using:  
```tableau
ISMEMBEROF('Client_UHC')
```
- Drop or mask PII fields before Tableau ingestion  

</details>

<details>
<summary><strong>📤 Export Limitations</strong></summary>

- CSV export from sanitized datasets only  
- Restrict access via Tableau Server permissions  

</details>

<details>
<summary><strong>🧾 Access Control Summary</strong></summary>

| Layer   | Technique                         | Purpose                            |
|---------|-----------------------------------|------------------------------------|
| SQL     | RLS via View/DBT                  | Client-level filtering             |
| Tableau | Groups + Filters                  | Secure data delivery               |
| Export  | Server controls                   | Prevent PII exposure               |

</details>

# D1.D: Anonymization, Grouping, and Export Logic

## 🔐 Anonymization

To comply with HIPAA and internal privacy regulations, the following steps are implemented in the DBT transformation layer:

- **Patient ID Hashing:** Replace patient identifiers with a salted hash using SHA256 for consistent but anonymous tracking.

<details>
<summary>SQL Example Hypothetical</summary>

```sql 
SELECT
  SHA256(CONCAT(patient_id, 'secure_salt')) AS hashed_patient_id
FROM patients
```
</details>

- **DOB Bucketing:** Convert `patients.date_of_birth` into non-identifiable age buckets for analysis.

<details>
<summary>SQL Example Hypothetical</summary>

```sql
CASE
  WHEN DATE_PART('year', AGE(CURRENT_DATE, date_of_birth)) BETWEEN 20 AND 29 THEN '20-29'
  WHEN DATE_PART('year', AGE(CURRENT_DATE, date_of_birth)) BETWEEN 30 AND 39 THEN '30-39'
  ELSE '40+'
END AS age_bucket
```
</details>

- **PII Exclusion:** Drop fields such as `name`, `address`, and exact `date_of_birth` from reporting views.

## 📦 Grouping (Procedure Standardization)

Standardize procedure types for clearer client reporting and analytics by grouping procedure codes.

Using the `procedures` table:

<details>
<summary>SQL Example Hypothetical</summary>

```sql
SELECT
  procedure_id,
  encounter_id,
  procedure_code,
  CASE
    WHEN procedure_code IN ('X123', 'X124') THEN 'Knee Surgery'
    WHEN procedure_code IN ('Y201', 'Y202') THEN 'Heart Monitoring'
    ELSE 'Other'
  END AS procedure_group,
  procedure_date
FROM procedures
```
</details>

This view (`procedure_grouped`) can then be joined with `encounters`, `payers`, and `transformed_patients`.

## 📤 Export Logic (Client View)

For example, to meet the needs of United Healthcare, a DBT model (`client_uhc_export`) aggregates and exports a de-identified, grouped dataset:

<details>
<summary>SQL Example Hypothetical</summary>

```sql
SELECT DISTINCT
  pg.procedure_group,
  tp.hashed_patient_id,
  p.coverage_type,
  MAX(pg.procedure_date) AS latest_procedure_date
FROM procedure_grouped pg
JOIN encounters e ON pg.encounter_id = e.encounter_id
JOIN transformed_patients tp ON e.patient_id = tp.patient_id
JOIN payers p ON e.payer_id = p.payer_id
WHERE p.client = 'United Healthcare'
GROUP BY 1, 2, 3
ORDER BY 1, 4 DESC
```
</details>

This model serves as a Tableau extract source, ensuring client visibility without compromising patient privacy.
