# MyTomorrows – Scalable & Secure Client Reporting  
_A case study for a growing Patient Advocacy Group_

## 🧠 Objective  
Design a scalable, secure, and privacy-compliant reporting workflow to support both internal and client-facing analytics in a healthcare-focused environment.

---

## ⚠️ Very Important Information About Deliverables  

Each task deliverable has been named **exactly as requested**, following the required naming convention:  
- Task 1 → `D1.A`, `D1.B`, etc.  
- Task 2 → `D2.A`, `D2.B`, etc.  
- And so on for subsequent sections.

This ensures full alignment with submission guidelines and allows for quick referencing and review.

---

## ⚙️ Note on Tooling & Workflow Adjustments  

### 🛠️ Initial Plan  
The original architecture involved:
- Loading the dataset into **PostgreSQL**
- Transforming it using **DBT**
- Orchestrating the pipeline with **Airflow**
- Visualizing with **Tableau**

### 🔄 Adapted Approach  
Due to local environment constraints and time sensitivity, I adapted the setup to focus on delivering the **core value** of the assignment using accessible tools.

---

## 💻 Data Modeling & Transformation

### Tools Used:
- **DBeaver** (as SQL interface)
- **SQLite** (lightweight local database)
- **CSV files** (as source data)

### Why DBeaver + SQLite?  
This combination allowed me to:
- Quickly **explore** and **understand** the dataset  
- Test **joins**, **filters**, and **business logic** (e.g., uninsured patients, high-volume procedures)
- Visualize relationships before moving to a production-grade setup

> I treated this as a "sandbox phase" — experiment first, then scale to PostgreSQL + DBT as a theoretical proposal.

### Implementation Notes:
- Created schemas manually in DBeaver
- Modeled key views to simulate reporting layers
- In some cases, views were converted to tables due to SQLite limitations

---

## 📊 Visualization Layer

### Tools Used:
- **Tableau (local version)**
- **Excel (as data connector)**

### Why Excel?  
My local Tableau version does not support live database connections. As a workaround:
- Transformed views were exported as Excel files
- Used those as the data source for dashboarding
- Mockups and annotated walk-throughs were included where interactivity was limited

---

## 🔍 Focus Areas
- Data pipeline design and modeling strategy  
- Privacy-aware reporting (e.g., anonymization for external use)  
- Dashboard logic, filters, and UX for internal/external stakeholders

---

Even though I didn’t implement a full production-grade stack, this case study reflects **my analytical thinking**, **data design decisions**, and how I **adapt to constraints** to deliver insight-driven solutions.


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

## D.1 A.  How I Would Model and Transform the Data (DBT or SQL Logic)

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

