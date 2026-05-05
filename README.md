#  E-Commerce Customer Analytics — SQL Portfolio Project

**Author:** Faith Olewe &nbsp;|&nbsp; [![LinkedIn](https://img.shields.io/badge/LinkedIn-Faith%20Olewe-0077B5?style=flat&logo=linkedin)](https://www.linkedin.com/in/faith-olewe/)

---

##  Project Overview

This project uses the **Olist Brazilian E-Commerce Dataset** (100,000+ real orders, 2016–2018) to answer 10 business questions using SQL — progressing from basic aggregations to window functions, cohort analysis, and RFM customer segmentation. The goal was to demonstrate my ability to translate business questions into SQL queries and extract actionable insights from a relational database.

**Tools:** PostgreSQL 18 · pgAdmin 4  
**Dataset:** [Olist Brazilian E-Commerce — Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)  
**Skills demonstrated:** CTEs · Window functions · Cohort analysis · RFM segmentation · Timestamp arithmetic · Multi-table JOINs

---

## Database Schema

The dataset consists of 8 tables centred around an `orders` fact table:

```
customers ──┐
             ├──▶ orders ──▶ order_items ──▶ products
sellers ────┘         │                └──▶ sellers
                      ├──▶ order_payments
                      ├──▶ order_reviews
                      └──▶ geolocation (via zip code)
```

> **Key data quality note:** The dataset contains two customer ID fields. `customer_id` is unique per order, while `customer_unique_id` identifies a unique person. All repeat-purchase and retention analyses use `customer_unique_id` to avoid overcounting.

---

##  Data Exploration & Schema Validation

Before answering business questions, the dataset was audited to understand row counts, date range, order status distribution, and null values in key columns.

![Part A Query](PART%20A%20QUERY.png)
![Part A Result](PART%20A%20RESULT.png)
![Part B — Date Range](PART%20B.png)
![Part C — Order Status Breakdown](PART%20C.png)
![Part D — Null Audit](PART%20D.png)

---

##  Business Questions & SQL Queries

---

### Q1 — What is the monthly revenue trend, and which months had the highest growth?

**Business context:** Understand seasonality and identify peak trading periods.

```sql
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', o.order_purchase_timestamp) AS month,
        ROUND(SUM(p.payment_value)::NUMERIC, 2)         AS revenue
    FROM orders o
    JOIN order_payments p ON o.order_id = p.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY 1
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month)                          AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100
    , 1)                                                        AS mom_growth_pct
FROM monthly
ORDER BY month;
```

![Q1 Result](Q1%20RESULT.png)
![Q1 Line Chart](Q1%20LINE%20CHART.png)

** Insight:** *(Revenue grew consistently from R$46,567 in October 2016 to over R$1.1M by mid-2018, with the sharpest single-month jump occurring in November 2017 (54% MoM growth), strongly indicating a Black Friday effect. The platform reached maturity around January 2018 where revenue stabilised above R$1M per month.)*

---

### Q2 — Which product categories generate the most revenue, and what is their average review score?

**Business context:** Identify which categories drive the business and whether high revenue correlates with customer satisfaction.

```sql
SELECT
    pr.product_category_name_english          AS category,
    COUNT(DISTINCT oi.order_id)               AS total_orders,
    ROUND(SUM(oi.price)::NUMERIC, 2)          AS total_revenue,
    ROUND(AVG(oi.price)::NUMERIC, 2)          AS avg_item_price,
    ROUND(AVG(rv.review_score)::NUMERIC, 2)   AS avg_review_score
FROM order_items oi
JOIN products p       ON oi.product_id = p.product_id
JOIN product_category_name_translation pr
                      ON p.product_category_name = pr.product_category_name
JOIN order_reviews rv ON oi.order_id = rv.order_id
JOIN orders o         ON oi.order_id = o.order_id
WHERE o.order_status = 'delivered'
GROUP BY category
HAVING COUNT(DISTINCT oi.order_id) > 100
ORDER BY total_revenue DESC
LIMIT 15;
```

![Q2 Result](Q2%20RESULT.png)
![Q2 Bar Chart](Q2%20BAR%20CHART.png)

** Insight:** *(Health & Beauty leads all categories with R$1.23M in revenue and a strong 4.19 average review score, suggesting both high demand and customer satisfaction. Bed, Bath & Table ranks third by revenue but has the lowest review score among the top 5 at 3.92, pointing to a potential quality or fulfilment issue worth investigating.)*

---

### Q3 — What is the average delivery time, and how does it vary by seller state?

**Business context:** Logistics performance varies by region — identifying laggards helps prioritise fulfilment improvements.

```sql
SELECT
    s.seller_state,
    COUNT(o.order_id)                                               AS deliveries,
    ROUND(AVG(
        EXTRACT(EPOCH FROM (
            o.order_delivered_customer_date - o.order_purchase_timestamp
        )) / 86400
    )::NUMERIC, 1)                                                  AS avg_days_to_deliver,
    ROUND(AVG(
        EXTRACT(EPOCH FROM (
            o.order_estimated_delivery_date - o.order_delivered_customer_date
        )) / 86400
    )::NUMERIC, 1)                                                  AS avg_days_early_late
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN sellers s      ON oi.seller_id = s.seller_id
WHERE o.order_status = 'delivered'
  AND o.order_delivered_customer_date IS NOT NULL
GROUP BY s.seller_state
HAVING COUNT(o.order_id) > 50
ORDER BY avg_days_to_deliver DESC;
```
![Q3 Result](Q3%20RESULT.png)

** Insight:** *(Sellers in CE (Ceará) and MA (Maranhão) have the slowest average delivery times at 17.9 and 17.7 days respectively, and are also among the latest relative to estimates. São Paulo (SP), which handles 78,598 deliveries — the highest volume by far — averages just 12.3 days, highlighting a significant logistics disparity between high-volume and peripheral states.)*

---

### Q4 — Which sellers are top performers on revenue, and which have the worst review scores?

**Business context:** A seller scorecard helps marketplace teams reward top partners and flag underperformers.

```sql
WITH seller_stats AS (
    SELECT
        oi.seller_id,
        s.seller_state,
        COUNT(DISTINCT oi.order_id)              AS total_orders,
        ROUND(SUM(oi.price)::NUMERIC, 2)         AS total_revenue,
        ROUND(AVG(rv.review_score)::NUMERIC, 2)  AS avg_score,
        RANK() OVER (ORDER BY SUM(oi.price) DESC)       AS revenue_rank,
        RANK() OVER (ORDER BY AVG(rv.review_score) ASC) AS worst_score_rank
    FROM order_items oi
    JOIN sellers s        ON oi.seller_id = s.seller_id
    JOIN order_reviews rv ON oi.order_id = rv.order_id
    JOIN orders o         ON oi.order_id = o.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY oi.seller_id, s.seller_state
    HAVING COUNT(DISTINCT oi.order_id) >= 30
)
SELECT *
FROM seller_stats
WHERE revenue_rank <= 20 OR worst_score_rank <= 10
ORDER BY revenue_rank;
```

![Q4 Result](Q4%20RESULT.png)

** Insight:** *(The top-ranked seller by revenue generated R$225,586 across 1,116 orders with a solid 4.14 review score, showing that high volume and quality can coexist. Notably, seller rank 5 generated R$186,664 but has the lowest review score among the top performers at 3.35 — a flag for the marketplace team to investigate fulfilment or product quality issues.)*

---

### Q5 — What percentage of customers make repeat purchases?

**Business context:** Repeat purchase rate is a core retention metric for any e-commerce business.

```sql
WITH customer_orders AS (
    SELECT
        c.customer_unique_id,
        COUNT(o.order_id) AS order_count
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id
)
SELECT
    order_count,
    COUNT(*)                                              AS customers,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2)   AS pct_of_total
FROM customer_orders
GROUP BY order_count
ORDER BY order_count;
```

![Q5 Result](Q5%20RESULT.png)

** Insight:** *(97% of customers placed only one order during the entire observation period, with just 2.76% placing two orders and virtually no one placing three or more. This reveals a structural retention problem — Olist's marketplace was almost entirely dependent on acquiring new customers rather than generating repeat business.)*

---

### Q6 — Monthly customer cohort retention table

**Business context:** Cohort analysis reveals whether customers acquired in a given month return to buy again in subsequent months.

```sql
WITH first_purchase AS (
    SELECT
        c.customer_unique_id,
        DATE_TRUNC('month', MIN(o.order_purchase_timestamp)) AS cohort_month
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id
),
all_purchases AS (
    SELECT
        c.customer_unique_id,
        DATE_TRUNC('month', o.order_purchase_timestamp) AS order_month
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id,
             DATE_TRUNC('month', o.order_purchase_timestamp)
),
cohort_data AS (
    SELECT
        fp.cohort_month,
        DATE_PART('month', AGE(ap.order_month, fp.cohort_month))::INT AS month_number,
        COUNT(DISTINCT fp.customer_unique_id) AS retained
    FROM first_purchase fp
    JOIN all_purchases ap USING (customer_unique_id)
    GROUP BY fp.cohort_month, month_number
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(*) AS cohort_size
    FROM first_purchase
    GROUP BY cohort_month
)
SELECT
    cd.cohort_month,
    cs.cohort_size,
    cd.month_number,
    cd.retained,
    ROUND(cd.retained * 100.0 / cs.cohort_size, 1) AS retention_pct
FROM cohort_data cd
JOIN cohort_sizes cs ON cd.cohort_month = cs.cohort_month
ORDER BY cd.cohort_month, cd.month_number;
```

![Q6 Result](Q6%20RESULT.png)

**Cohort Retention Heatmap:**

![Q6 Cohort Heatmap](Q6%20HEAT%20MAP.png)

** Insight:** *(Across all monthly cohorts from September 2016 to August 2018, retention drops to below 1% by Month 2 for virtually every cohort. The October 2016 cohort — one of the earliest — shows the highest Month 1 retention at 0.4%, but even this is negligible. The heatmap confirms that one-time purchasing was not a seasonal pattern but a persistent structural characteristic of the platform.)*

---

### Q7 — Running total of revenue and cumulative share by category (Pareto analysis)

**Business context:** Identify whether revenue follows the 80/20 rule — a few categories driving most of the business.

```sql
WITH category_revenue AS (
    SELECT
        pr.product_category_name_english  AS category,
        ROUND(SUM(oi.price)::NUMERIC, 2)  AS revenue
    FROM order_items oi
    JOIN products p  ON oi.product_id = p.product_id
    JOIN product_category_name_translation pr
                     ON p.product_category_name = pr.product_category_name
    JOIN orders o    ON oi.order_id = o.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY category
)
SELECT
    category,
    revenue,
    SUM(revenue) OVER (ORDER BY revenue DESC
                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  AS running_total,
    ROUND(
        SUM(revenue) OVER (ORDER BY revenue DESC
                           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        / SUM(revenue) OVER () * 100
    , 1)                                                                  AS cumulative_pct
FROM category_revenue
ORDER BY revenue DESC;
```

![Q7 Result](Q7%20RESULT.png)

** Insight:** *(The top 7 categories account for 50% of total revenue, and the top 14 categories account for 75% — indicating a moderately diversified product mix rather than a pure 80/20 concentration. Health & Beauty alone contributes 9.5% of all revenue, making it the single most critical category to protect and grow.)*

---

### Q8 — How does review score correlate with delivery speed?

**Business context:** Quantify how much delivery performance drives customer satisfaction scores.

```sql
WITH delivery_data AS (
    SELECT
        o.order_id,
        rv.review_score,
        EXTRACT(EPOCH FROM (
            o.order_delivered_customer_date - o.order_estimated_delivery_date
        )) / 86400  AS days_vs_estimate
    FROM orders o
    JOIN order_reviews rv ON o.order_id = rv.order_id
    WHERE o.order_status = 'delivered'
      AND o.order_delivered_customer_date IS NOT NULL
)
SELECT
    CASE
        WHEN days_vs_estimate < -7  THEN '1. More than 7 days early'
        WHEN days_vs_estimate < 0   THEN '2. Early (0–7 days)'
        WHEN days_vs_estimate = 0   THEN '3. On time'
        WHEN days_vs_estimate <= 7  THEN '4. Late (1–7 days)'
        ELSE                             '5. Very late (7+ days)'
    END                                     AS delivery_bucket,
    COUNT(*)                                AS orders,
    ROUND(AVG(review_score)::NUMERIC, 2)    AS avg_review_score,
    ROUND(
        SUM(CASE WHEN review_score >= 4 THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
    , 1)                                    AS pct_positive_reviews
FROM delivery_data
GROUP BY delivery_bucket
ORDER BY delivery_bucket;
```

![Q8 Result](Q8%20RESULT.png)

** Insight:** *(The data shows a stark and direct relationship between delivery speed and customer satisfaction. Orders delivered more than 7 days early average a 4.32 review score with 83.5% positive reviews, while very late orders (7+ days past estimate) average just 1.73 with only 12.6% positive reviews — a 2.5x difference in satisfaction. On-time delivery is clearly the single most controllable driver of customer experience on this platform.)*

---

### Q9 — Which payment methods are most popular, and do they correlate with order value?

**Business context:** Payment mix informs checkout UX decisions and instalment financing strategy.

```sql
SELECT
    payment_type,
    COUNT(DISTINCT order_id)                              AS orders,
    ROUND(AVG(payment_value)::NUMERIC, 2)                 AS avg_order_value,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP
          (ORDER BY payment_value)::NUMERIC, 2)           AS median_order_value,
    ROUND(SUM(payment_value)::NUMERIC, 2)                 AS total_revenue,
    ROUND(SUM(payment_value) * 100.0
          / SUM(SUM(payment_value)) OVER ()
    , 1)                                                  AS revenue_share_pct,
    ROUND(AVG(payment_installments)::NUMERIC, 1)          AS avg_installments
FROM order_payments
GROUP BY payment_type
ORDER BY total_revenue DESC;
```

![Q9 Result](Q9%20RESULT.png)

** Insight:** *(Credit card dominates with 78.3% of total revenue across 76,505 orders, with customers averaging 3.5 instalments per transaction hence reflecting the importance of instalment financing in the Brazilian market. Boleto (bank slip) is a distant second at 17.9% revenue share, while vouchers and debit cards together account for less than 4%.)*

---

### Q10 — Identify high-value customers at risk of churning (RFM scoring)

**Business context:** Segment customers by Recency, Frequency and Monetary value to prioritise retention marketing spend.

```sql
WITH rfm_base AS (
    SELECT
        c.customer_unique_id,
        MAX(o.order_purchase_timestamp)                         AS last_purchase,
        COUNT(DISTINCT o.order_id)                              AS frequency,
        ROUND(SUM(p.payment_value)::NUMERIC, 2)                 AS monetary
    FROM customers c
    JOIN orders o          ON c.customer_id = o.customer_id
    JOIN order_payments p  ON o.order_id = p.order_id
    WHERE o.order_status = 'delivered'
    GROUP BY c.customer_unique_id
),
rfm_scored AS (
    SELECT *,
        DATE_PART('day', NOW() - last_purchase)::INT            AS recency_days,
        NTILE(5) OVER (ORDER BY last_purchase DESC)             AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC)                  AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC)                   AS m_score
    FROM rfm_base
)
SELECT
    customer_unique_id,
    recency_days,
    frequency,
    monetary,
    r_score, f_score, m_score,
    (r_score + f_score + m_score)  AS rfm_total,
    CASE
        WHEN r_score >= 4 AND f_score >= 4 THEN 'Champion'
        WHEN r_score >= 3 AND f_score >= 3 THEN 'Loyal'
        WHEN r_score >= 4 AND f_score <= 2 THEN 'New Customer'
        WHEN r_score <= 2 AND f_score >= 3 THEN 'At Risk'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost'
        ELSE 'Occasional'
    END                            AS rfm_segment
FROM rfm_scored
ORDER BY rfm_total DESC;
```

![Q10 Result](Q10%20RESULT.png)

** Insight:** *(The RFM model scored 93,357 customers across 94 pages of results. The top segment shown — Champions — are customers with high recency, frequency and monetary scores (all 5s, rfm_total of 15), though the dataset's low repeat purchase rate confirmed by Q5 means true Champions are rare. The majority of the customer base falls into lower RFM tiers, reinforcing that Olist's retention challenge is platform-wide rather than isolated to specific segments)*

---

##  Key Findings Summary

1. **Revenue seasonality:** *(Revenue grew 24x from October 2016 to mid-2018, with a clear Black Friday spike in November 2017 (+54% MoM) confirming seasonal demand patterns.)*
2. **Category concentration:** *(The top 7 categories drive 50% of total revenue, led by Health & Beauty — a moderately diversified mix, but one where losing the top 3 categories would eliminate over a quarter of the business.)*
3. **Retention crisis:** *(97% of customers never returned after their first purchase. Month-2 retention across all cohorts sits below 1%, indicating the platform was structurally acquisition-dependent throughout its entire observed history.)*
4. **Delivery drives satisfaction:** *(Very late deliveries score 1.73 on average vs 4.32 for early deliveries — a 2.5x gap. Improving logistics in slow states like CE and MA is the clearest lever available to improve NPS.)*
5. **Payment behaviour:** *(Credit card with instalments accounts for 78% of revenue at an average of 3.5 instalments per order, confirming that instalment financing is not optional but central to how Brazilian consumers shop online.)*

---

##  How to Reproduce

**Requirements:** PostgreSQL 18 · pgAdmin 4

**Steps:**
1. Download the dataset from [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
2. Create a database called `olist` in pgAdmin
3. Run `setup.sql` to create all 8 tables and indexes
4. Import each CSV using pgAdmin's Import/Export tool (Format: CSV, Header: ON, Encoding: UTF8)
5. Run queries from the `queries/` folder in order

---

*Built by [Faith Olewe](https://www.linkedin.com/in/faith-olewe/)*
