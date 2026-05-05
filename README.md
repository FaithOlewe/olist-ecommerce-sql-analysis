# 🛒 E-Commerce Customer Analytics — SQL Portfolio Project

**Author:** Faith Olewe &nbsp;|&nbsp; [![LinkedIn](https://img.shields.io/badge/LinkedIn-Faith%20Olewe-0077B5?style=flat&logo=linkedin)](https://www.linkedin.com/in/faith-olewe/)

---

## 📌 Project Overview

This project uses the **Olist Brazilian E-Commerce Dataset** (100,000+ real orders, 2016–2018) to answer 10 business questions using SQL — progressing from basic aggregations to window functions, cohort analysis, and RFM customer segmentation. The goal was to demonstrate the ability to translate business questions into SQL queries and extract actionable insights from a relational database.

**Tools:** PostgreSQL 18 · pgAdmin 4  
**Dataset:** [Olist Brazilian E-Commerce — Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)  
**Skills demonstrated:** CTEs · Window functions · Cohort analysis · RFM segmentation · Timestamp arithmetic · Multi-table JOINs

---

## 🗂️ Database Schema

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

## 🔍 Data Exploration & Schema Validation

Before answering business questions, the dataset was audited to understand row counts, date range, order status distribution, and null values in key columns.

![Part A Query](images/PART%20A%20QUERY.png)
![Part A Result](images/PART%20A%20RESULT.png)
![Part B — Date Range](images/PART%20B.png)
![Part C — Order Status Breakdown](images/PART%20C.png)
![Part D — Null Audit](images/PART%20D.png)

---

## 📊 Business Questions & SQL Queries

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

![Q1 Query](images/Q1%20QUERY.png)
![Q1 Result](images/Q1%20RESULT.png)
![Q1 Line Chart](images/Q1%20LINE%20CHART.png)

**💡 Insight:** *(Replace with your finding — e.g. "Revenue peaked in November 2017 with 27% MoM growth, consistent with Black Friday activity.")*

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

![Q2 Query](images/Q2%20QUERY.png)
![Q2 Result](images/Q2%20RESULT.png)
![Q2 Bar Chart](images/Q2%20BAR%20CHART.png)

**💡 Insight:** *(Replace with your finding — e.g. "Health & Beauty is the top revenue category but ranks only 8th in review score, suggesting a quality or expectation gap.")*

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

![Q3 Query](images/Q3%20QUERY.png)
![Q3 Result](images/Q3%20RESULT.png)

**💡 Insight:** *(Replace with your finding — e.g. "Sellers in northern states average 21 days to deliver vs 8 days for São Paulo-based sellers, a 2.6x difference.")*

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

![Q4 Query](images/Q4%20QUERY.png)
![Q4 Result](images/Q4%20RESULT.png)

**💡 Insight:** *(Replace with your finding — e.g. "Two sellers appear in both the top 20 by revenue and bottom 10 by review score — high volume but poor customer experience.")*

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

![Q5 Query](images/Q5%20QUERY.png)
![Q5 Result](images/Q5%20RESULT.png)

**💡 Insight:** *(Replace with your finding — e.g. "97% of customers placed only one order, suggesting retention is a critical weakness for Olist's marketplace model.")*

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

![Q6 Query Part A](images/Q6%20a%20QUERY.png)
![Q6 Query Part B](images/Q6%20b%20QUERY.png)
![Q6 Result](images/Q6%20RESULT.png)

**Cohort Retention Heatmap:**

![Q6 Cohort Heatmap](images/Q6%20HEAT%20MAP.png)

**💡 Insight:** *(Replace with your finding — e.g. "Month-1 retention averages below 1% across all cohorts, indicating that repeat purchasing was structurally absent from Olist's marketplace throughout the observed period.")*

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

![Q7 Query](images/Q7%20QUERY.png)
![Q7 Result](images/Q7%20RESULT.png)

**💡 Insight:** *(Replace with your finding — e.g. "The top 10 categories account for 68% of total revenue — a moderate concentration suggesting a relatively diversified product mix.")*

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

![Q8 Query Part A](images/Q8%20a%20QUERY.png)
![Q8 Query Part B](images/Q8%20b%20QUERY.png)
![Q8 Result](images/Q8%20RESULT.png)

**💡 Insight:** *(Replace with your finding — e.g. "Orders delivered 7+ days late average a 2.1 review score vs 4.3 for early deliveries — delivery speed is the single strongest driver of customer satisfaction.")*

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

![Q9 Query](images/Q9%20QUERY.png)
![Q9 Result](images/Q9%20RESULT.png)

**💡 Insight:** *(Replace with your finding — e.g. "Credit card accounts for 74% of revenue with an average of 3.7 instalments, suggesting Brazilian shoppers heavily rely on instalment financing.")*

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

![Q10 Query Part A](images/Q10%20a%20QUERY.png)
![Q10 Query Part B](images/Q10%20b%20QUERY.png)
![Q10 Result](images/Q10%20RESULT.png)

**💡 Insight:** *(Replace with your finding — e.g. "Less than 1% of customers qualify as Champions or Loyal — the vast majority fall into Lost or Occasional, reinforcing the retention opportunity identified throughout this analysis.")*

---

## 🔑 Key Findings Summary

1. **Revenue seasonality:** *(Your Q1 finding)*
2. **Category concentration:** *(Your Q7 finding — how many categories drive 80% of revenue)*
3. **Retention crisis:** *(Your Q5/Q6 finding on repeat purchase rate)*
4. **Delivery drives satisfaction:** *(Your Q8 finding on review score vs delivery speed)*
5. **Payment behaviour:** *(Your Q9 finding on credit card and instalments)*

---

## ⚙️ How to Reproduce

**Requirements:** PostgreSQL 18 · pgAdmin 4

**Steps:**
1. Download the dataset from [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
2. Create a database called `olist` in pgAdmin
3. Run `setup.sql` to create all 8 tables and indexes
4. Import each CSV using pgAdmin's Import/Export tool (Format: CSV, Header: ON, Encoding: UTF8)
5. Run queries from the `queries/` folder in order

**Folder structure:**
```
olist-sql-project/
├── README.md
├── setup.sql
├── images/                        ← all screenshots go here
├── queries/
│   ├── q01_monthly_revenue.sql
│   ├── q02_category_revenue.sql
│   ├── q03_delivery_by_state.sql
│   ├── q04_seller_scorecard.sql
│   ├── q05_repeat_purchases.sql
│   ├── q06_cohort_retention.sql
│   ├── q07_pareto_analysis.sql
│   ├── q08_delivery_vs_reviews.sql
│   ├── q09_payment_methods.sql
│   └── q10_rfm_segmentation.sql
└── outputs/
    └── Q6 HEAT MAP.png
```

---

*Built by [Faith Olewe](https://www.linkedin.com/in/faith-olewe/)*
