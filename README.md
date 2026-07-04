# 🏋️‍♂️ Fitness Center Subscription Analytics (SQL & Business Intelligence)

## 📌 Project Overview
This project focuses on providing actionable business intelligence for a modern subscription-based fitness center. By simulating real-world relational data, the project answers critical business questions regarding **customer retention (Churn Risk)**, **Monthly Recurring Revenue (MRR)**, and **facility utilization metrics**.

The primary goal is to help gym management optimize staffing schedules, target marketing campaigns at high-risk users, and maximize revenue growth through data-driven decisions.

## 🛠️ Tech Stack & Database Architecture
*   **Database Engine:** PostgreSQL
*   **Database GUI Tool:** DBeaver
*   **Analytics Techniques:** Common Table Expressions (CTEs), Window Functions, Data Aggregation, and Case Segmentations.

### Entity-Relationship (ER) Schema
The relational database consists of 4 core tables:
1.  `membership_plans` - Stores tier details (`Bronze`, `Silver`, `Gold`, `VIP`), monthly prices, and visit limits.
2.  `members` - Customer profiles including registration dates and their operational status (`Active`, `Cancelled`, `Suspended`).
3.  `subscriptions` - Tracks transaction cycles, end dates, and payment success rates.
4.  `check_ins` - Real-time gate logs recording the timestamp and exact zone (`Gym`, `Cardio`, `Pool`, `Group Class`) for every visit.

---

## 📊 Business Metrics & Advanced Queries

### 1. Utilization & Churn Risk Analysis
**Business Value:** Identifies passive paying customers who are at a high risk of canceling their subscription next month, while highlighting users ready for a tier upgrade.

```sql
WITH usage_stats AS (
    SELECT 
        m.member_id,
        CONCAT(m.first_name, ' ', m.last_name) AS member_name,
        mp.plan_name,
        mp.included_visits_per_month,
        COUNT(c.check_in_id) AS visits_june_2026
    FROM members m
    JOIN subscriptions s ON m.member_id = s.member_id
    JOIN membership_plans mp ON s.plan_id = mp.plan_id
    LEFT JOIN check_ins c ON m.member_id = c.member_id 
        AND c.check_in_time >= '2026-06-01' AND c.check_in_time < '2026-07-01'
    WHERE m.status = 'Active'
    GROUP BY m.member_id, m.first_name, m.last_name, mp.plan_name, mp.included_visits_per_month
)
SELECT 
    member_id,
    member_name,
    plan_name,
    visits_june_2026,
    CASE 
        WHEN included_visits_per_month = 999 THEN 'Unlimited'
        ELSE CAST(included_visits_per_month AS VARCHAR)
    END AS monthly_limit,
    CASE 
        WHEN included_visits_per_month <> 999 AND visits_june_2026 >= included_visits_per_month THEN 'Upgrade Target'
        WHEN visits_june_2026 <= 1 THEN 'High Churn Risk'
        WHEN visits_june_2026 > 15 THEN 'Highly Engaged'
        ELSE 'Healthy Retention'
    END AS customer_health_status
FROM usage_stats
ORDER BY visits_june_2026 DESC;

