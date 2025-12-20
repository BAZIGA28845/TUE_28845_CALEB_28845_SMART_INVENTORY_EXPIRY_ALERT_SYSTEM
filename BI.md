

# **PHASE VIII – Business Intelligence (BI) Implementation**

## **Short Summary**

Phase VIII focuses on transforming database data into **meaningful insights** using **Power BI**.
The system analyzes **user activity patterns** and **product value trends** through aggregation and visualization, supporting informed decision-making.

---

## **Objective**

To support management and administrative decisions by:

* Aggregating operational data
* Identifying usage patterns by user role
* Analyzing product value trends over time
* Presenting insights through dashboards

---

## **1️⃣ Defined KPIs (Based on Implemented Power BI Analysis)**

### **KPI 1: User Activity Volume**

* **Metric:** Sum of PHONE
* **Grouped By:** USER_ID, ROLE
* **Purpose:** Measure operational activity per user role
<img width="960" height="501" alt="KPI" src="https://github.com/user-attachments/assets/a5554e07-4625-4a51-b15d-bdac1bc40e66" />

---

### **KPI 2: Product Value Contribution**

* **Metric:** Sum of UNIT_PRICE
* **Grouped By:** CATEGORY, CREATED_AT
* **Purpose:** Identify valuable product categories over time
<img width="957" height="504" alt="Sum of UNIT_PRICE by category created by" src="https://github.com/user-attachments/assets/7956fdfa-495f-4ff6-b400-501610d5bdf3" />


---

## **2️⃣ Decision Support Needs**

| Decision Area    | BI Insight Provided                    |
| ---------------- | -------------------------------------- |
| User Management  | Identify highly active user roles      |
| Role Performance | Compare workload by role               |
| Product Strategy | Identify high-value product categories |
| Trend Analysis   | Track product value growth over time   |

---

## **3️⃣ Stakeholders**

| Stakeholder          | BI Interest                        |
| -------------------- | ---------------------------------- |
| System Administrator | User activity by role              |
| Management           | Product value trends               |
| Inventory Manager    | Category-based product performance |
| Auditors             | User-level traceability            |

---

## **4️⃣ Reporting Frequency**

| Report Type             | Frequency |
| ----------------------- | --------- |
| User Activity Dashboard | Monthly   |
| Product Value Trends    | Monthly   |
| Executive BI Summary    | Quarterly |

---

## **5️⃣ Dashboards Implemented in Power BI**

---

### **A. Executive Summary Dashboard**

**Purpose:** High-level overview for management.

**Visuals Included:**

* KPI Card: Total Product Value (Sum of UNIT_PRICE)
* Line Chart: Sum of UNIT_PRICE by CREATED_AT
* Bar Chart: Sum of UNIT_PRICE by CATEGORY

```
[ Total Product Value ]
Product Value Trend (by Date)
Category Contribution Chart
```
B. Audit / User Activity Dashboard
<img width="948" height="341" alt="BI" src="https://github.com/user-attachments/assets/9e164696-0104-4a15-8676-958aa608c34d" />


Purpose: Analyze system usage by user roles.

Visuals Included:

Bar Chart: Sum of PHONE by USER_ID

Stacked Chart: Sum of PHONE by ROLE

Table: USER_ID, ROLE, PHONE
---



## **6️⃣ Power BI Implementation Notes**

* Data sourced from Oracle database
* Aggregations performed using **Power BI measures**
* Filters applied by ROLE, CATEGORY, and DATE
* Dashboards updated monthly

---
## **RUNNING CODES IN SQL**
1️⃣ User Activity Analysis

Sum of PHONE by USER_ID and ROLE
<img width="959" height="291" alt="image" src="https://github.com/user-attachments/assets/6d8b9a25-2c4e-4206-90ab-b1bc21c2a021" />


Table contains: USER_ID, ROLE, PHONE

Create View for Power BI
CREATE OR REPLACE VIEW vw_user_activity AS
SELECT
    user_id,
    role,
    SUM(phone) AS total_phone
FROM users
GROUP BY user_id, role;

What this does

Aggregates PHONE values

Groups results by user and role

Used for Audit / Performance Dashboard

2️⃣ Product Value Analysis

Sum of UNIT_PRICE by CATEGORY and CREATED_AT
<img width="959" height="287" alt="image" src="https://github.com/user-attachments/assets/404a1708-7420-47ab-a1b6-1294e415296c" />



Create View for Power BI
CREATE OR REPLACE VIEW vw_product_value_trend AS
SELECT
    category,
    TRUNC(created_at) AS created_date,
    SUM(unit_price) AS total_value
FROM product
GROUP BY category, TRUNC(created_at);

What this does

Aggregates product value

Groups by product category and creation date

Used for Executive Summary Dashboard

3️⃣ Optional: Total Product Value KPI
<img width="960" height="262" alt="Total Product Value KPI" src="https://github.com/user-attachments/assets/56c53cd8-061b-46b9-93f2-911df8f22c36" />

CREATE OR REPLACE VIEW vw_total_product_value AS
SELECT
    SUM(unit_price) AS total_product_value
FROM product;


## **7️⃣ One-Line Examiner Justification**

> *Phase VIII uses Power BI to analyze user activity and product value trends through aggregated metrics.*

---

## **Conclusion**

Phase VIII successfully demonstrates the use of **business intelligence tools** to convert raw database data into **actionable insights**, supporting management decisions 
