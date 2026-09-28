# ShopNova RFM Customer Segmentation & PostgreSQL Data Engineering

## 🚀 Project Overview
The **ShopNova RFM Customer Segmentation** project bridges the gap between raw e-commerce data and actionable business strategy. Moving beyond standard visualization tools, this project focuses on backend database engineering, data modeling, and advanced SQL transformations in **PostgreSQL** to segment customers based on Recency, Frequency, and Monetary (RFM) behavior.

---

## 🛠️ Tech Stack & Skills Demonstrated
* **Database Management System:** PostgreSQL (via pgAdmin 4)
* **Data Modeling:** Relational schema design with normalized `customers` and `transactions` tables linked by Primary/Foreign Keys.
* **Advanced SQL Techniques:**
  * Common Table Expressions (CTEs) for multi-step querying.
  * Window Functions (`NTILE(5)`) for behavioral scoring.
  * Conditional Logic (`CASE WHEN`) for dynamic customer segmentation.
  * Database Views (`CREATE VIEW`) for reusable BI reporting layers.

---

## 📂 Database Architecture & Schema
The database consists of two core relational tables:
1. **`customers`**: Stores individual customer metadata (`customer_id` [PK], `customer_name`, `country`, `gender`).
2. **`transactions`**: Stores order-level activity linked to customers (`transaction_id` [PK], `customer_id` [FK], `order_date`, `order_amount`, `product_id`).

---

## 💻 Core SQL Transformation Logic
To compute customer value, raw transactional data is aggregated and scored using a 5-tier quantile distribution (`NTILE`). Below is a snippet of the core transformation query:

```sql
WITH RawRFM AS (
    SELECT 
        c.customer_id,
        CURRENT_DATE - MAX(t.order_date) AS Recency,
        COUNT(DISTINCT t.transaction_id) AS Frequency,
        SUM(t.order_amount) AS Monetary
    FROM customers c
    JOIN transactions t ON c.customer_id = t.customer_id
    GROUP BY c.customer_id
),
RFM_Scores AS (
    SELECT 
        customer_id,
        Recency,
        Frequency,
        Monetary,
        NTILE(5) OVER (ORDER BY Recency ASC) AS R_Score,
        NTILE(5) OVER (ORDER BY Frequency DESC) AS F_Score,
        NTILE(5) OVER (ORDER BY Monetary DESC) AS M_Score
    FROM RawRFM
),
RFM_Segments AS (
    SELECT 
        customer_id,
        Recency,
        Frequency,
        Monetary,
        CASE 
            WHEN R_Score >= 4 AND F_Score >= 4 AND M_Score >= 4 THEN 'Champions'
            WHEN R_Score >= 3 AND F_Score >= 3 AND M_Score >= 3 THEN 'Loyal Customers'
            WHEN R_Score >= 4 AND F_Score <= 2 THEN 'New Customers'
            WHEN R_Score <= 2 AND F_Score >= 4 THEN 'At-Risk'
            WHEN R_Score <= 2 AND F_Score <= 2 THEN 'Lost Customers'
            ELSE 'Potential Loyalist'
        END AS customer_segment
    FROM RFM_Scores
)
SELECT 
    customer_segment,
    COUNT(customer_id) AS total_customers,
    ROUND(SUM(Monetary), 2) AS total_revenue,
    ROUND(AVG(Monetary), 2) AS avg_customer_spend,
    ROUND(AVG(Recency), 1) AS avg_recency_days
FROM RFM_Segments
GROUP BY customer_segment
ORDER BY total_revenue DESC;
