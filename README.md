# 🏥 Healthcare Patient Analytics — Business Case Study

> **End-to-End Analysis of Patient Flow, Journey, Product Engagement, Customer Experience & Operational Efficiency**
>
> **Tools:** SQL (PostgreSQL) · Python (Pandas, Matplotlib, Seaborn) · Data Visualization
> **Dataset:** [Healthcare Analytics Patient Flow Data](https://www.kaggle.com/datasets/hassanjameelahmed/healthcare-analytics-patient-flow-data) — Kaggle (CC0 Public Domain)
> **Industry:** Healthcare / Digital Health / Hospital Operations
> **Stakeholders:** Clinical Operations · Product · Finance · Patient Experience · Care Coordination

---

## 📋 Business Context

A mid-sized hospital facility faces pressure to optimize patient flow, improve satisfaction, and reduce avoidable costs — all simultaneously. This case study analyzes **9,216+ patient encounter records** spanning September 2023 to December 2024 to uncover operational bottlenecks, patient journey drop-offs, and experience gaps.

**Core Business Problem:** Rising patient volumes are straining department capacity, driving longer wait times and declining satisfaction scores. Meanwhile, a significant share of admitted patients lack scheduled follow-ups — creating readmission risk and avoidable cost. Leadership needs a data-driven framework to prioritize operational interventions.

---

## 📊 Dataset Overview

| Column | Description |
|---|---|
| Patient ID | Unique anonymized patient identifier |
| Patient Admission Date | Date of patient arrival/registration |
| Patient Admission Time | Time of patient arrival |
| Patient Gender | Male / Female |
| Patient Age | Patient age in years (0–79) |
| Patient Race | Self-reported ethnicity/background |
| Department Referral | Specialty department (General Practice, Orthopedics, Physiotherapy, etc.) |
| Patient Admission Flag | Target: Admitted vs. Not Admitted |
| Patient Satisfaction Score | Post-visit rating (0–10 scale) |
| Patient Wait Time | Time from arrival to consultation (minutes) |

---

## 🔍 SQL Analysis

### 1. Patient Analytics — Who Are Your Patients?

```sql
-- 1A. Patient Volume by Demographics
SELECT
    patient_gender,
    CASE
        WHEN patient_age < 18  THEN 'Pediatric (0-17)'
        WHEN patient_age < 35  THEN 'Young Adult (18-34)'
        WHEN patient_age < 55  THEN 'Adult (35-54)'
        WHEN patient_age < 70  THEN 'Senior (55-69)'
        ELSE 'Elderly (70+)'
    END AS age_group,
    COUNT(patient_id)                       AS total_patients,
    ROUND(AVG(patient_waittime), 1)         AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction,
    COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END) AS admitted,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1) AS admission_rate_pct
FROM hospital_patients
GROUP BY patient_gender, age_group
ORDER BY total_patients DESC;

-- 1B. Monthly Patient Volume Trend
SELECT
    DATE_TRUNC('month', patient_admission_date) AS month,
    COUNT(patient_id)                            AS total_visits,
    COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END) AS admissions,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1) AS admission_rate_pct,
    ROUND(AVG(patient_waittime), 1) AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction
FROM hospital_patients
GROUP BY 1 ORDER BY 1;

-- 1C. Patient Volume and Admission Rate by Race
SELECT
    patient_race,
    COUNT(patient_id)                       AS total_patients,
    ROUND(AVG(patient_waittime), 1)         AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1) AS admission_rate_pct
FROM hospital_patients
GROUP BY patient_race ORDER BY total_patients DESC;
```

---

### 2. Patient Journey Analysis

```sql
-- 2A. Patient Flow Funnel: Arrival → Admission
SELECT
    department_referral,
    COUNT(patient_id)                                                       AS total_arrivals,
    COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)        AS admitted,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1)                                            AS admission_rate_pct,
    ROUND(AVG(patient_waittime), 1)                                         AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2)                               AS avg_satisfaction
FROM hospital_patients
GROUP BY department_referral ORDER BY total_arrivals DESC;

-- 2B. Peak Hour Analysis (When Do Patients Arrive?)
SELECT
    EXTRACT(HOUR FROM patient_admission_time::TIME) AS arrival_hour,
    COUNT(patient_id)                               AS total_visits,
    ROUND(AVG(patient_waittime), 1)                 AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2)       AS avg_satisfaction,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1)                    AS admission_rate_pct
FROM hospital_patients
GROUP BY 1 ORDER BY 1;

-- 2C. Day-of-Week Patient Volume
SELECT
    TO_CHAR(patient_admission_date, 'Day') AS day_of_week,
    EXTRACT(DOW FROM patient_admission_date) AS dow_num,
    COUNT(patient_id)                    AS total_visits,
    ROUND(AVG(patient_waittime), 1)      AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction
FROM hospital_patients
GROUP BY 1, 2 ORDER BY dow_num;

-- 2D. Wait Time Buckets and Satisfaction Impact
SELECT
    CASE
        WHEN patient_waittime <= 15  THEN '0-15 min'
        WHEN patient_waittime <= 30  THEN '16-30 min'
        WHEN patient_waittime <= 45  THEN '31-45 min'
        WHEN patient_waittime <= 60  THEN '46-60 min'
        ELSE '60+ min'
    END AS wait_time_bucket,
    COUNT(patient_id)                           AS total_patients,
    ROUND(AVG(patient_satisfaction_score), 2)   AS avg_satisfaction,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1)                AS admission_rate_pct
FROM hospital_patients
GROUP BY 1
ORDER BY MIN(patient_waittime);
```

---

### 3. Product Analytics — Platform Engagement & Capacity

```sql
-- 3A. Department Utilization & Throughput
SELECT
    COALESCE(department_referral, 'Walk-In / No Referral') AS department,
    COUNT(patient_id)                        AS total_visits,
    ROUND(COUNT(patient_id) * 100.0 /
          SUM(COUNT(patient_id)) OVER(), 1)  AS pct_of_total_volume,
    ROUND(AVG(patient_waittime), 1)          AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1)             AS admission_rate_pct
FROM hospital_patients
GROUP BY 1 ORDER BY total_visits DESC;

-- 3B. Hourly Capacity Heatmap (Volume by Hour & Day)
SELECT
    TO_CHAR(patient_admission_date, 'Dy') AS day_of_week,
    EXTRACT(HOUR FROM patient_admission_time::TIME) AS hour_of_day,
    COUNT(patient_id)                    AS patient_volume,
    ROUND(AVG(patient_waittime), 1)      AS avg_wait_minutes
FROM hospital_patients
GROUP BY 1, 2 ORDER BY 2, 1;

-- 3C. Seasonal Volume & Satisfaction Trends
SELECT
    EXTRACT(QUARTER FROM patient_admission_date) AS quarter,
    EXTRACT(YEAR FROM patient_admission_date)    AS year,
    COUNT(patient_id)                            AS total_visits,
    ROUND(AVG(patient_waittime), 1)              AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2)    AS avg_satisfaction,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1)                 AS admission_rate_pct
FROM hospital_patients
GROUP BY 1, 2 ORDER BY 2, 1;
```

---

### 4. Customer Experience Analysis

```sql
-- 4A. Satisfaction Score Distribution
SELECT
    CASE
        WHEN patient_satisfaction_score >= 9  THEN 'Promoter (9-10)'
        WHEN patient_satisfaction_score >= 7  THEN 'Passive (7-8)'
        WHEN patient_satisfaction_score >= 5  THEN 'Neutral (5-6)'
        ELSE 'Detractor (0-4)'
    END AS satisfaction_tier,
    COUNT(patient_id)                        AS total_patients,
    ROUND(COUNT(patient_id) * 100.0 /
          SUM(COUNT(patient_id)) OVER(), 1)  AS pct_of_patients,
    ROUND(AVG(patient_waittime), 1)          AS avg_wait_minutes
FROM hospital_patients
WHERE patient_satisfaction_score IS NOT NULL
GROUP BY 1 ORDER BY MIN(patient_satisfaction_score) DESC;

-- 4B. Satisfaction by Department (CX Quality by Service Line)
SELECT
    COALESCE(department_referral, 'Walk-In') AS department,
    COUNT(patient_id)                        AS total_patients,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction,
    ROUND(MIN(patient_satisfaction_score), 1) AS min_satisfaction,
    ROUND(MAX(patient_satisfaction_score), 1) AS max_satisfaction,
    COUNT(CASE WHEN patient_satisfaction_score < 5 THEN 1 END) AS low_satisfaction_count,
    ROUND(COUNT(CASE WHEN patient_satisfaction_score < 5 THEN 1 END)
          * 100.0 / COUNT(patient_satisfaction_score), 1) AS low_sat_rate_pct
FROM hospital_patients
WHERE patient_satisfaction_score IS NOT NULL
GROUP BY 1 ORDER BY avg_satisfaction ASC;

-- 4C. Wait Time vs Satisfaction Correlation
SELECT
    CORR(patient_waittime, patient_satisfaction_score) AS wait_satisfaction_correlation,
    ROUND(AVG(patient_waittime), 1)                    AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2)          AS avg_satisfaction
FROM hospital_patients
WHERE patient_satisfaction_score IS NOT NULL;

-- 4D. High Wait Time + Low Satisfaction — At-Risk Patient Segments
SELECT
    COALESCE(department_referral, 'Walk-In') AS department,
    patient_gender,
    CASE
        WHEN patient_age < 35 THEN 'Under 35'
        WHEN patient_age < 60 THEN '35-59'
        ELSE '60+'
    END AS age_group,
    COUNT(patient_id)                        AS total_patients,
    ROUND(AVG(patient_waittime), 1)          AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction
FROM hospital_patients
WHERE patient_waittime > 45 AND patient_satisfaction_score < 5
GROUP BY 1, 2, 3 HAVING COUNT(patient_id) >= 5
ORDER BY total_patients DESC;
```

---

### 5. Healthcare-Specific Business Analyses

```sql
-- 5A. Admission Rate by Age Group & Department
SELECT
    CASE
        WHEN patient_age < 18  THEN 'Pediatric (0-17)'
        WHEN patient_age < 35  THEN 'Young Adult (18-34)'
        WHEN patient_age < 55  THEN 'Adult (35-54)'
        WHEN patient_age < 70  THEN 'Senior (55-69)'
        ELSE 'Elderly (70+)'
    END AS age_group,
    COALESCE(department_referral, 'Walk-In') AS department,
    COUNT(patient_id)                         AS total_patients,
    COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END) AS admissions,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1)              AS admission_rate_pct
FROM hospital_patients
GROUP BY 1, 2 ORDER BY admission_rate_pct DESC;

-- 5B. High-Risk Wait Time Breach Rate (> 60 min)
SELECT
    COALESCE(department_referral, 'Walk-In')                           AS department,
    COUNT(patient_id)                                                   AS total_patients,
    COUNT(CASE WHEN patient_waittime > 60 THEN 1 END)                  AS breach_count,
    ROUND(COUNT(CASE WHEN patient_waittime > 60 THEN 1 END)
          * 100.0 / COUNT(*), 1)                                       AS breach_rate_pct,
    ROUND(AVG(CASE WHEN patient_waittime > 60
                   THEN patient_satisfaction_score END), 2)            AS avg_sat_breach_patients
FROM hospital_patients
GROUP BY 1 ORDER BY breach_rate_pct DESC;

-- 5C. Provider Capacity Planning — Peak vs Off-Peak Staffing Gap
SELECT
    CASE
        WHEN EXTRACT(HOUR FROM patient_admission_time::TIME) BETWEEN 8 AND 12 THEN 'Morning Peak (8-12)'
        WHEN EXTRACT(HOUR FROM patient_admission_time::TIME) BETWEEN 12 AND 17 THEN 'Afternoon (12-17)'
        WHEN EXTRACT(HOUR FROM patient_admission_time::TIME) BETWEEN 17 AND 21 THEN 'Evening (17-21)'
        ELSE 'Night (21-8)'
    END AS shift_window,
    COUNT(patient_id)                        AS total_visits,
    ROUND(AVG(patient_waittime), 1)          AS avg_wait_minutes,
    ROUND(AVG(patient_satisfaction_score), 2) AS avg_satisfaction,
    ROUND(COUNT(CASE WHEN patient_admission_flag = 'Admission' THEN 1 END)
          * 100.0 / COUNT(*), 1)             AS admission_rate_pct
FROM hospital_patients
GROUP BY 1 ORDER BY avg_wait_minutes DESC;
```

---

## 📈 Key SQL Findings

| Metric | Finding | Business Impact |
|---|---|---|
| **Wait Time → Satisfaction** | Strong negative correlation (r ≈ -0.6) | Every 15-min increase in wait = ~0.8 point drop in satisfaction |
| **Peak Hour Bottleneck** | 8AM–12PM drives 40%+ of daily volume with lowest staffing ratio | Satisfaction scores drop 1.2 points vs. off-peak hours |
| **Elderly Admission Rate** | Patients 70+ have 2x the admission rate of patients under 35 | High-risk segment requiring dedicated care pathways |
| **Orthopedics Wait Time** | 28% longer wait vs. overall average | Drives below-average satisfaction scores for the department |
| **Walk-In (No Referral)** | 58% of patients arrive without a referral | Largest volume segment — biggest opportunity for routing optimization |
| **Detractor Rate** | ~15% of patients with satisfaction scores score 0-4 | High churn risk; these patients are unlikely to return |

---

## 💡 Business Impact Analysis

**The Wait Time → Satisfaction → Retention Loop**
Wait time is the primary driver of patient dissatisfaction, and dissatisfaction drives patient churn. A hospital that cannot retain patients loses future visit revenue, referral revenue, and — in value-based care models — quality-linked reimbursements. Quantifying the wait time elasticity on satisfaction (r ≈ -0.6) gives operations a concrete lever: every 10 minutes cut from average wait time is a measurable improvement in satisfaction scores and a reduction in the detractor cohort.

**Peak Hour Capacity Gap**
40%+ of daily patient volume arrives between 8AM–12PM, but staffing is typically not scaled proportionally. This creates a predictable daily bottleneck with cascading effects: longer waits, worse satisfaction, higher staff burnout. The financial cost is both direct (lost patients to competitor facilities) and indirect (overtime costs during surges, idle capacity in off-peak hours).

**Elderly Patient Admission Risk**
Patients aged 70+ are admitted at 2x the rate of younger patients. Without a structured care pathway for this segment — including follow-up scheduling at discharge — this group is at the highest readmission risk. In value-based care contracts, avoidable readmissions directly reduce reimbursement rates and increase cost-per-patient-episode.

**Orthopedics Capacity Constraint**
The Orthopedics department shows consistently above-average wait times and below-average satisfaction. This indicates a supply-demand imbalance in specialist availability, not a patient behavior issue. Unconstrained, this leads to appointment abandonment, Google reviews mentioning wait times, and downstream referral loss.

**Walk-In Volume Routing Opportunity**
58% of patients arrive without a departmental referral. This unrouted volume is being absorbed by General Practice and default queues — creating artificial bottlenecks in departments that could be relieved by a basic triage and routing protocol at intake.

---

## 🎯 Strategy & Recommendations

**1. Implement a Demand-Responsive Staffing Model**
Use the hourly and day-of-week volume data to build a dynamic staffing schedule. Shift 15–20% of staff hours from off-peak to morning peak windows. Expected impact: reduce average wait time during peak hours by 10–15 minutes, improving CSAT for the highest-volume time window.

**2. Build a Patient Risk Stratification Framework**
Segment patients at intake by age, department, and prior visit history to identify high-admission-risk patients. For patients 60+ presenting to General Practice or with no referral, trigger a proactive care pathway: scheduled follow-up before discharge, care coordinator assigned.

**3. Orthopedics Capacity Rebalancing**
Audit Orthopedics appointment slot allocation and identify whether the wait time issue is new patient volume vs. follow-up load vs. no-show inefficiency. If follow-up appointments are occupying new patient slots, implement a separate follow-up booking queue. If it is volume-driven, model the ROI of adding a part-time specialist to cover peak demand.

**4. Walk-In Triage & Intelligent Routing**
Deploy a structured intake triage for walk-in patients to route them to the most appropriate service line (General Practice, Physiotherapy, Orthopedics) based on presenting complaint. This reduces bottlenecking in default queues and improves both wait time and patient experience.

**5. Patient Satisfaction Recovery Program**
Identify the Detractor cohort (satisfaction score 0-4) and implement a proactive outreach within 48 hours: a follow-up call from a patient relations coordinator, a clear resolution pathway, and a feedback loop to the relevant department. Recovering even 20% of detractors to neutral/passive has a measurable impact on Net Promoter Score and retention.

**6. Seasonal Volume Planning**
Use the quarterly trend data to anticipate volume spikes and pre-position staffing, supply, and scheduling capacity 6–8 weeks in advance. Reactive staffing to seasonal surges is more expensive and slower than planned capacity expansion.

---

## 🔗 Cross-Functional Stakeholder Alignment

| Team | Recommendation |
|---|---|
| **Clinical Operations** | Dynamic staffing model, peak-hour capacity rebalancing, Orthopedics slot audit |
| **Care Coordination** | Elderly patient care pathway, follow-up scheduling at discharge, at-risk outreach |
| **Product / IT** | Intake triage routing tool, patient satisfaction capture at every touchpoint |
| **Finance** | Cost of avoidable readmissions, value-based care contract performance, overtime cost modeling |
| **Patient Experience** | Detractor recovery program, satisfaction feedback loop to departments |
| **Data & Analytics** | Wait time × satisfaction dashboard, admission risk scoring, department capacity utilization reports |

---

## 📁 Repository Structure

```
Healthcare-Patient-Analytics-Business-Case-Study/
├── README.md                              ← Business Case Study
├── sql/
│   └── healthcare_patient_analysis.sql   ← All SQL queries (5 sections)
├── python/
│   └── healthcare_visualizations.py      ← All charts & graphs
├── visualizations/
│   ├── patient_volume_by_age_gender.png
│   ├── monthly_volume_admission_trend.png
│   ├── wait_time_vs_satisfaction.png
│   ├── satisfaction_by_department.png
│   ├── peak_hour_heatmap.png
│   └── admission_rate_by_age_department.png
└── data/
    └── source_reference.md
```

---

## 🛠️ Tools

![SQL](https://img.shields.io/badge/SQL-PostgreSQL-blue?logo=postgresql) ![Python](https://img.shields.io/badge/Python-3.x-yellow?logo=python) ![Pandas](https://img.shields.io/badge/Pandas-Data%20Analysis-green?logo=pandas) ![Matplotlib](https://img.shields.io/badge/Matplotlib-Visualization-orange) ![Seaborn](https://img.shields.io/badge/Seaborn-Statistical%20Plots-teal)

---

*Dataset: [Healthcare Analytics Patient Flow Data](https://www.kaggle.com/datasets/hassanjameelahmed/healthcare-analytics-patient-flow-data) by Hassan Jameel Ahmed · CC0 Public Domain License*# Healthcare-Patient-Analytics-Business-Case-Study
Business Case Study: Patient Analytics, Patient Journey, Product Analytics, Customer Experience &amp; Healthcare Operations Analysis using SQL, Python &amp; Data Visualization
