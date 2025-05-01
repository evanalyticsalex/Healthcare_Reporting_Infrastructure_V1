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
