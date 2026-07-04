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
```
### 2. Monthly Recurring Revenue (MRR) & Subscription Split
**Business Value:** Evaluates which plan tier generates the highest financial stability and its overall percentage contribution to the business turnover.
```sql
SELECT 
    mp.plan_name,
    COUNT(s.subscription_id) AS active_subscriptions,
    SUM(mp.monthly_price) AS total_monthly_revenue,
    ROUND(
        (SUM(mp.monthly_price) / SUM(SUM(mp.monthly_price)) OVER ()) * 100, 2
    ) AS revenue_contribution_pct
FROM subscriptions s
JOIN membership_plans mp ON s.plan_id = mp.plan_id
JOIN members m ON s.member_id = m.member_id
WHERE m.status = 'Active' AND s.payment_status = 'Paid'
GROUP BY mp.plan_name
ORDER BY total_monthly_revenue DESC;
```
### 3. Peak Hours & Zone Crowding
Business Value: Ranks the top 3 busiest hours for each zone to help management optimize trainer shifts and plan group class schedules effectively.
```sql
SELECT 
    facility_area,
    EXTRACT(HOUR FROM check_in_time) AS check_in_hour,
    COUNT(*) AS total_visits,
    RANK() OVER (PARTITION BY facility_area ORDER BY COUNT(*) DESC) AS peak_hour_rank
FROM check_ins
GROUP BY facility_area, EXTRACT(HOUR FROM check_in_time)
ORDER BY facility_area, total_visits DESC;
```
📈 Key Insights & Future BI Roadmap
Operational Optimization: Gate data shows massive traffic spikes between 17:00 and 20:00 in the Gym and Cardio zones, allowing management to relocate staff proactively.

Revenue Growth: Highlighting the "Upgrade Target" segment opens a direct path for automated upsell emails.

Next Steps: Connecting this PostgreSQL engine to a data visualization platform to build an interactive operations dashboard.
