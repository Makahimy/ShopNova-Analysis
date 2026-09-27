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
