
# 💊 MyTomorrows – Scalable & Secure Client Reporting
_A case study for a growing Patient Advocacy Group_

## 🎯 Objective
Design a scalable, secure, and privacy-compliant reporting workflow to support both internal and client-facing analytics in a healthcare-focused environment.

---

## 📌 Very Important Information About Deliverables

Each task deliverable has been named **exactly as requested**, following the required naming convention:
- Task 1 → `D1.A`, `D1.B`, etc.
- Task 2 → `D2.A`, `D2.B`, etc.

This ensures full alignment with submission guidelines and allows for quick referencing and review.

---

## 🧰 Tooling & Workflow Adjustments

<details>
<summary><strong>🛠️ Initial Plan</strong></summary>

The original architecture involved:
- Loading the dataset into **PostgreSQL**
- Transforming it using **DBT**
- Orchestrating the pipeline with **Airflow**
- Visualizing with **Tableau**

</details>

<details>
<summary><strong>🔄 Adapted Approach</strong></summary>

Due to local environment constraints and time sensitivity, I adapted the setup to focus on delivering the **core value** of the assignment using accessible tools.

</details>

---

## 🗂️ Data Modeling & Transformation

<details>
<summary><strong>🧪 Tools Used</strong></summary>

- **DBeaver** (as SQL interface)
- **SQLite** (lightweight local database)
- **CSV files** (as source data)

</details>

<details>
<summary><strong>🧠 Why DBeaver + SQLite?</strong></summary>

This combination allowed me to:
- Quickly **explore** and **understand** the dataset
- Test **joins**, **filters**, and **business logic**
- Visualize relationships before moving to a production-grade setup

> I treated this as a "sandbox phase" — experiment first, then scale to PostgreSQL + DBT as a theoretical proposal.

</details>

<details>
<summary><strong>⚙️ Implementation Notes</strong></summary>

- Created schemas manually in DBeaver
- Modeled key views to simulate reporting layers
- Converted views to tables where SQLite limitations required it

</details>

---

## 📊 Visualization Layer

<details>
<summary><strong>🖥️ Tools Used</strong></summary>

- **Tableau (local version)**
- **Excel (as data connector)**

</details>

<details>
<summary><strong>📈 Why Excel?</strong></summary>

My local Tableau version does not support live database connections. As a workaround:
- Transformed views were exported as Excel files
- Used those as the data source for dashboarding
- Created mockups and walkthroughs where interactivity was limited

</details>

---

## 🔍 Focus Areas

- ✅ Data pipeline design and modeling strategy
- ✅ Privacy-aware reporting (e.g., anonymization for external use)
- ✅ Dashboard logic, filters, and UX for internal/external stakeholders

---

Even though I didn’t implement a full production-grade stack, this case study reflects **my analytical thinking**, **data design decisions**, and how I **adapt to constraints** to deliver insight-driven solutions.

---

# 📁 Task 1 - Client-Facing Report for UnitedHealthcare

## 📌 Summary of the Assignment

UnitedHealthcare, as a client of a Patient Advocacy Group, requested a secure and standardized report that helps them:

- Understand and group procedures into consistent categories
- Identify high-volume procedures
- Flag missed patients, especially those without insurance
- Export the data sorted by procedure and latest date
- Receive insights via Tableau dashboards without exposing PII

---

## 🧩 D1.A - How I Would Model and Transform the Data

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
<summary><strong>2️⃣ Intermediate Layer – Grouping Logic-Standardize Insurance Coverage</strong></summary>

```sql
CREATE VIEW X1_rpt_encounter_procedure_with_grouping AS
SELECT
e.Id AS encounter_id,
e.START,
e.PATIENT,
e.DESCRIPTION AS encounter_description,
p.CODE AS procedure_code,
p.DESCRIPTION AS procedure_description,
p.BASE_COST,
e.PAYER AS payer,
pa.NAME AS payer_name,
o.NAME AS org_name,
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
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers pa ON e.PAYER = pa.Id
JOIN dim_organizations o ON e.ORGANIZATION = o.Id;
```

</details>

<details>
<summary><strong>3️⃣ Reporting Layer – Final Views</strong></summary>

**Volume Report for UHC**
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

**High Volume Patients**
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

**Uninsured Patients**
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

**Export the data by procedure and latest procedure date**
```sql
CREATE VIEW C8_rpt_latest_procedure_summary_uhc AS
WITH X2_filtered_united AS (
SELECT *
FROM X2_rpt_tableau_export_masked
WHERE LOWER(payer_name) LIKE '%united%'
),

C8_latest_dates_per_patient AS (
SELECT masked_patient_id, MAX(START) AS max_start
FROM X2_filtered_united
GROUP BY masked_patient_id
),

C8_latest_procedures_per_patient AS (
SELECT fd.*
FROM X2_filtered_united fd
JOIN C8_latest_dates_per_patient ld
ON fd.masked_patient_id = ld.masked_patient_id
AND fd.START = ld.max_start
),

C8_final_classified_data AS (
SELECT
*,
CASE
WHEN LOWER(payer) LIKE '%no_insurance%' OR LOWER(payer) LIKE '%no insurance%' THEN 'Uninsured'
ELSE 'Insured'
END AS insurance_status
FROM C8_latest_procedures_per_patient
)

SELECT
standardized_group,
COUNT(DISTINCT masked_patient_id) AS distinct_patient_count,
COUNT(encounter_id) AS encounter_count,
insurance_status,
substr(START, 1, 10) AS start_ymd
FROM C8_final_classified_data
GROUP BY standardized_group, insurance_status, start_ymd
ORDER BY distinct_patient_count DESC;
```

<details>
<summary><strong>4️⃣ Privacy & Anonymization Logic</strong></summary>

```sql
substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12) AS masked_patient_id
```

**Final Tableau Export Example Masked Patient ID**
```sql
CREATE VIEW X2_rpt_tableau_export_masked AS
SELECT
e.Id AS encounter_id,
e.START,
substr(HEX(abs(e.PATIENT * 100000007 % 1000000007)), 1, 12) AS masked_patient_id,
e.DESCRIPTION AS encounter_description,
p.CODE AS procedure_code,
p.DESCRIPTION AS procedure_description,
p.BASE_COST,
e.TOTAL_CLAIM_COST,
e.PAYER_COVERAGE,
e.payer,
pa.NAME AS payer_name,
o.NAME AS org_name,
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
FROM fct_procedures p
JOIN fct_encounters e ON p.ENCOUNTER = e.Id
JOIN dim_payers pa ON e.PAYER = pa.Id
JOIN dim_organizations o ON e.ORGANIZATION = o.Id;
```


</details>

<details>
<summary><strong>✅ Why This Structure?</strong></summary>

- Each layer is clean, logical, and reusable
- Modular views make debugging and future development easier
- Mimics DBT-style transformations, enabling lineage tracking
- Designed with privacy and export-readiness in mind
- Ideal for use in Tableau or other BI tools



---

## D1.B 📊 Tableau Dashboard Mockup Plan

<details>
<summary><strong>📋 Filters (Top of Dashboard)</strong></summary>

- **Standardized Procedure Group** (`procedure_group`)
- **Date Range** (on `procedure_date`)
- **Insurance Coverage** (`payer_name`)

</details>

<details>
<summary><strong>📌 KPI Tiles</strong></summary>

- **Total Encounters:** `COUNT(encounter_id)` from C1
- **Uninsured Patients:** Derived from C4
- **High-Volume Patients:** From aggregation in C2

</details>

<details>
<summary><strong>📊 Bar Chart</strong></summary>

- **X-axis:** `procedure_group`
- **Y-axis:** `encounter_count`
- *Data Source:* Use V1 or C3

</details>

<details>
<summary><strong>📈 Line Chart</strong></summary>

- **X-axis:** `procedure_date` (monthly aggregation)
- **Y-axis:** Monthly encounter volume
- *Data Source:* Use X2

</details>

<details>
<summary><strong>📄 Table Export</strong></summary>

**Data Source:** `X2_rpt_tableau_export_masked`
**Columns to Show:**
- `procedure_group`
- `cost`
- `procedure_date`
- `organization_name`
- `payer_name`

</details>


# 🔐 D1.C – Filters and Permissions for Secure Client Access

## 🧮 SQL-Level Row-Level Security (RLS)

- `WHERE client = 'United Healthcare'`
- Apply via:
- **Secure View** in PostgreSQL
- or within a **DBT model** with client-specific logic
- Ensures data is **filtered before reaching Tableau**
- Supports compliance with **HIPAA** and internal data segmentation rules

---

## 📊 Tableau-Level Permissions

### 👥 User Group Segmentation
- Create user group: `Client_UHC`
- Assign **United Healthcare users** to this group via Tableau Server

### 🔎 Data Source Filters
- Apply formula:
```tableau
ISMEMBEROF('Client_UHC')
```

- Restricts dashboard visibility to rows matching that client

### 🕶️ Hide PII Fields
- Drop or mask Personally Identifiable Information:
- Done at **SQL level** (DBT or source view)
- Tableau **never sees** raw PII data

---

## 📤 Export Limitation & Access Control

- **CSV export** allowed *only* from sanitized, aggregated tables
- Avoid joins to any table containing PII
- Use **Tableau Server permissions** to:
- Restrict export capabilities by role
- Limit access to download options per dashboard

---

## 🧾 Summary of Access Control Layers

| Layer   | Technique                         | Purpose                            |
|---------|-----------------------------------|------------------------------------|
| SQL     | Row-Level Security in DBT/View    | Pre-filter data by client          |
| Tableau | User Groups + Data Source Filters | Restrict visibility per user group |
| Export  | Server-level CSV restrictions     | Prevent PII leakage on download    |
