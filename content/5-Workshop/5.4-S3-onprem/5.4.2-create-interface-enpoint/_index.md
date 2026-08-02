---
title: "Building the Data Mart Layer (Data Mart Models)"
date: 2026-07-23
weight: 2
chapter: false
pre: " <b> 5.4.2 </b> "
---

If the Staging layer described in Section 5.4.1 is responsible for cleaning and standardizing data at the record level, then the Data Mart (Gold) layer aggregates that cleansed data into tables designed to answer specific business questions. This is the layer directly queried by BI tools and data analysts. Therefore, each model in this layer is materialized as a physical **table** (`+materialized: table`) rather than a view, ensuring consistent query performance and avoiding repeated execution of aggregation logic for every analytical query.

## 1. Organization by Business Domain

The project's four Data Mart models are organized into three subdirectories under `models/mart/`, each corresponding to an independent business domain:

| Business Domain | Directory | Model | Business Question Answered |
| :--- | :--- | :--- | :--- |
| Finance | `mart/finance/` | `revenue_daily` | How do revenue, order volume, and cancellation rate change on a daily basis? |
| Finance | `mart/finance/` | `category_revenue_monthly` | Which product categories contribute the most revenue each month? |
| Customer Analytics | `mart/customers/` | `fct_customer_cohorts` | For customers whose first purchase occurred in month *X*, how many returned to purchase again in subsequent months? |
| Operations | `mart/operations/` | `seller_performance` | Which sellers are performing well, and which require improvement in delivery performance or service quality? |

This organization is more than a directory structure. It directly maps to the schemas and tags defined in `dbt_project.yml` (described in detail in Section 5.5), enabling selective execution such as `dbt run --select mart.finance` when only a specific business domain needs to be rebuilt, rather than rerunning the entire Gold layer.

All four models follow a consistent design pattern. The initial Common Table Expressions (CTEs) retrieve data exclusively through `select * from {{ ref('stg_...') }}`, followed by one or more intermediate aggregation CTEs, and finally a single `final` CTE that produces the model output. This structure makes each transformation step easier to understand, test, and debug while avoiding one of the most common pitfalls in analytical SQL: the **fanout join** problem, where row counts are unintentionally multiplied when joining one-to-many relationships without first aggregating the data.

## 2. `revenue_daily` – Daily Revenue Report

This model has a grain of **one row per order date** (`order_date`). It summarizes Gross Merchandise Value (GMV), total revenue including freight charges, average order value, late delivery rate, and order cancellation rate.

```sql
-- models/mart/finance/revenue_daily.sql
--
-- Daily GMV (Gross Merchandise Value) report.
-- Joins orders + order_items + payments to get true revenue per day.
-- Materialized as TABLE on Redshift with sort/dist keys for fast BI queries.
--
-- Key metrics:
--   gmv         — sum of item prices (pre-freight)
--   total_revenue — gmv + freight
--   avg_order_value — revenue / distinct orders
--   cancellation_rate — % orders canceled that day

{{
    config(
        materialized='table',
        sort='order_date',
        dist='order_date',
        tags=['mart', 'finance', 'daily']
    )
}}

with orders as (
    select * from {{ ref('stg_orders') }}
    where order_date >= '{{ var("start_date") }}'
),

items as (
    select * from {{ ref('stg_order_items') }}
),

payments as (
    select
        order_id,
        sum(payment_value)    as total_payment_value,
        max(payment_type)     as primary_payment_type
    from {{ ref('stg_order_payments') }}
    group by 1
),

order_revenue as (
    select
        o.order_id,
        o.order_date,
        o.order_status,
        o.is_delivered,
        o.is_late_delivery,
        sum(i.item_price)            as gmv,
        sum(i.freight_value)         as freight_revenue,
        sum(i.total_item_revenue)    as gross_revenue,
        count(distinct i.product_id) as distinct_products,
        count(i.order_item_id)       as item_count,
        p.total_payment_value,
        p.primary_payment_type
    from orders o
    left join items i
        on o.order_id = i.order_id
    left join payments p
        on o.order_id = p.order_id
    group by 1, 2, 3, 4, 5, p.total_payment_value, p.primary_payment_type
),

daily_aggregated as (
    select
        order_date,

        -- Volume
        count(distinct order_id)                                 as total_orders,
        count(distinct case when is_delivered then order_id end) as delivered_orders,
        count(distinct case when order_status = 'canceled'
                            then order_id end)                   as canceled_orders,

        -- Revenue
        round(sum(gmv), 2)                                       as gmv,
        round(sum(freight_revenue), 2)                           as freight_revenue,
        round(sum(gross_revenue), 2)                             as total_revenue,
        round(avg(gross_revenue), 2)                             as avg_order_value,

        -- Delivery quality
        round(
            100.0 * count(distinct case when is_late_delivery then order_id end)
            / nullif(count(distinct case when is_delivered then order_id end), 0),
            2
        )                                                        as late_delivery_pct,

        -- Cancellation rate
        round(
            100.0 * count(distinct case when order_status = 'canceled'
                                        then order_id end)
            / nullif(count(distinct order_id), 0),
            2
        )                                                        as cancellation_rate_pct,

        -- Items
        sum(item_count)                                          as total_items_sold,
        round(avg(item_count), 2)                                as avg_items_per_order,

        -- Payment mix
        sum(case when primary_payment_type = 'credit_card'
                 then 1 else 0 end)                              as orders_credit_card,
        sum(case when primary_payment_type = 'boleto'
                 then 1 else 0 end)                              as orders_boleto,

        current_timestamp                                        as dbt_updated_at

    from order_revenue
    group by 1
)

select *
from daily_aggregated
order by order_date
```

In addition to standard business metrics such as GMV, total revenue, and average order value, the model separately reports freight revenue (`freight_revenue`), the average number of items per order (`avg_items_per_order`), and the distribution of payment methods for the two most common payment options in the Brazilian market—credit card and **boleto** (a bank payment slip)—through the `orders_credit_card` and `orders_boleto` count columns.

The most important technical aspect of this model is its handling of the one-to-many relationship between orders and payments. In the Olist dataset, a single order may contain multiple payment records (for example, installment payments). If `stg_order_payments` were joined directly to the row-level data in `stg_order_items` without prior aggregation, every order item would be duplicated according to the number of associated payment records, producing the **fanout join** issue introduced in Section 5.1.

To prevent this, the model first aggregates the `payments` CTE to exactly one row per `order_id` **before** performing any joins. It then aggregates the `items` data to the order level within the `order_revenue` CTE. As a result, each order contributes exactly one record when revenue is aggregated in the `daily_aggregated` CTE, regardless of how many items or payment transactions are associated with that order.

````markdown
## 3. `category_revenue_monthly` – Monthly Revenue by Product Category

This model has a grain of **one row per (month, English product category)** and is designed to answer the business question: *Which product categories generate the highest revenue each month?*

```sql
-- models/mart/finance/category_revenue_monthly.sql
-- Monthly revenue breakdown by product category (English name)

{{
    config(
        materialized='table',
        sort=['order_month', 'total_gmv'],
        tags=['mart', 'finance']
    )
}}

with items as (
    select * from {{ ref('stg_order_items') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
    where order_date >= '{{ var("start_date") }}'
      and order_status not in ('canceled', 'unavailable')
),

products as (
    select * from {{ ref('stg_products') }}
),

joined as (
    select
        date_trunc('month', o.order_date)::date     as order_month,
        p.category_en,
        count(distinct o.order_id)                  as order_count,
        count(i.order_item_id)                      as items_sold,
        round(sum(i.item_price), 2)                 as total_gmv,
        round(avg(i.item_price), 2)                 as avg_item_price,
        round(sum(i.freight_value), 2)              as total_freight
    from items i
    join orders o   on i.order_id = o.order_id
    join products p on i.product_id = p.product_id
    group by 1, 2
),

ranked as (
    select
        *,
        rank() over (
            partition by order_month
            order by total_gmv desc
        )                       as revenue_rank_in_month,
        current_timestamp       as dbt_updated_at
    from joined
)

select * from ranked
```

Two design decisions are particularly noteworthy in this model.

First, orders with the status `canceled` or `unavailable` are excluded from the `orders` CTE at the beginning of the pipeline. The objective of this report is to measure **actual realized revenue by product category**, rather than expected revenue from incomplete or canceled orders. This differs from the `revenue_daily` model, where canceled orders are intentionally retained to calculate the daily cancellation rate.

Second, the window function `rank() over (partition by order_month order by total_gmv desc)` precomputes the revenue ranking of each product category within every month. As a result, analytical queries such as "Top 5 best-selling categories this month" only need to filter `where revenue_rank_in_month <= 5`, without recalculating rankings within the BI layer.

## 4. `fct_customer_cohorts` – Customer Cohort and Retention Analysis

This model contains the most sophisticated business logic in the project. It implements **cohort analysis**, a common analytical technique in e-commerce for measuring customer retention over time based on the month in which a customer placed their first order (`cohort_month`).

A key characteristic of the Olist dataset must be considered. Each order is associated with a `customer_id`, which uniquely identifies a customer **only within a single order**, whereas `customer_unique_id` identifies the actual individual across multiple purchases. If cohort analysis were performed using `customer_id`, every repeat purchase would incorrectly appear as a new customer acquisition because each order is assigned a different `customer_id`. Consequently, the model must convert every order to its corresponding `customer_unique_id` from the very beginning.

```sql
-- models/mart/customers/fct_customer_cohorts.sql
--
-- Monthly cohort retention analysis.
-- For each cohort (month of first purchase), tracks how many customers
-- purchased again in subsequent months (M+1, M+2, ...).
-- Classic retention table used in e-commerce analytics.

{{
    config(
        materialized='table',
        sort=['cohort_month', 'months_since_first_order'],
        dist='cohort_month',
        tags=['mart', 'customers', 'cohort']
    )
}}

with customers as (
    select * from {{ ref('stg_customers') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
    where order_date >= '{{ var("start_date") }}'
      and order_status not in ('canceled', 'unavailable')
),

-- Map each order to the unique customer (not per-order customer_id)
customer_orders as (
    select
        c.customer_unique_id,
        o.order_date,
        date_trunc('month', o.order_date)::date     as order_month,
        o.order_id
    from orders o
    join customers c on o.customer_id = c.customer_id
),

-- First order date per unique customer
first_orders as (
    select
        customer_unique_id,
        min(order_month)    as cohort_month,
        min(order_date)     as first_order_date
    from customer_orders
    group by 1
),

-- Join back to get months since first order for each subsequent purchase
customer_activity as (
    select
        co.customer_unique_id,
        fo.cohort_month,
        co.order_month,
        datediff(
            'month',
            fo.cohort_month,
            co.order_month
        )                   as months_since_first_order
    from customer_orders co
    join first_orders fo on co.customer_unique_id = fo.customer_unique_id
),

-- Cohort size (how many unique customers acquired per month)
cohort_sizes as (
    select
        cohort_month,
        count(distinct customer_unique_id)  as cohort_size
    from first_orders
    group by 1
),

-- Retained customers per cohort-month combination
retention_counts as (
    select
        cohort_month,
        months_since_first_order,
        count(distinct customer_unique_id)  as retained_customers
    from customer_activity
    group by 1, 2
),

final as (
    select
        rc.cohort_month,
        cs.cohort_size,
        rc.months_since_first_order,
        rc.retained_customers,
        round(
            100.0 * rc.retained_customers / cs.cohort_size,
            2
        )                                   as retention_rate_pct,
        current_timestamp                   as dbt_updated_at
    from retention_counts rc
    join cohort_sizes cs on rc.cohort_month = cs.cohort_month
    where rc.months_since_first_order <= 12
)

select * from final
order by cohort_month, months_since_first_order
```

The transformation pipeline consists of five consecutive aggregation stages. It first identifies the real customer associated with each order (`customer_orders`), determines the cohort month for every customer (`first_orders`), computes the number of months between each purchase and the customer's cohort month (`customer_activity`), and then separately calculates the total number of customers in each cohort (`cohort_sizes`) and the number of retained customers at each monthly interval (`retention_counts`). These results are finally combined in the `final` CTE to calculate the customer retention rate (`retention_rate_pct`).

The analysis is limited to the first 12 months following the initial purchase (`months_since_first_order <= 12`), which is sufficient to generate the standard cohort-by-month retention matrix commonly used in e-commerce analytics.

## 5. `seller_performance` – Seller Performance Scorecard

This model evaluates seller performance across three dimensions—revenue generation, customer satisfaction, and delivery quality—and combines them into a single composite metric, `seller_score`, ranging from 0 to 100.

```sql
-- models/mart/operations/seller_performance.sql
--
-- Seller-level performance scorecard.
-- Aggregates revenue, order count, avg review score, late delivery rate.

{{
    config(
        materialized='table',
        dist='seller_id',
        tags=['mart', 'operations']
    )
}}

with sellers as (
    select * from {{ ref('stg_sellers') }}
),

items as (
    select * from {{ ref('stg_order_items') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
    where order_date >= '{{ var("start_date") }}'
),

reviews as (
    select * from {{ ref('stg_order_reviews') }}
),

-- revenue per seller
seller_revenue as (
    select
        i.seller_id,
        count(distinct i.order_id)          as total_orders,
        count(i.order_item_id)              as total_items_sold,
        round(sum(i.item_price), 2)         as total_gmv,
        round(avg(i.item_price), 2)         as avg_item_price,
        round(sum(i.freight_value), 2)      as total_freight,
        min(o.order_date)                   as first_order_date,
        max(o.order_date)                   as last_order_date
    from items i
    join orders o on i.order_id = o.order_id
    group by 1
),

-- review scores per seller
seller_reviews as (
    select
        i.seller_id,
        round(avg(r.review_score), 2)       as avg_review_score,
        count(distinct r.review_id)         as total_reviews,
        count(distinct case when r.sentiment = 'negative'
                            then r.review_id end) as negative_reviews
    from items i
    join reviews r on i.order_id = r.order_id
    group by 1
),

-- delivery performance per seller
seller_delivery as (
    select
        i.seller_id,
        count(distinct o.order_id) as delivered_orders,
        count(distinct case when o.is_late_delivery then o.order_id end) as late_deliveries
    from items i
    join orders o on i.order_id = o.order_id
    where o.is_delivered = true
    group by 1
),

final as (
    select
        s.seller_id,
        s.city                                              as seller_city,
        s.state_code                                        as seller_state,

        -- Revenue
        coalesce(r.total_orders, 0)                         as total_orders,
        coalesce(r.total_items_sold, 0)                     as total_items_sold,
        coalesce(r.total_gmv, 0)                            as total_gmv,
        coalesce(r.avg_item_price, 0)                       as avg_item_price,
        r.first_order_date,
        r.last_order_date,

        -- Reviews
        coalesce(rv.avg_review_score, 0)                    as avg_review_score,
        coalesce(rv.total_reviews, 0)                       as total_reviews,
        round(
            100.0 * coalesce(rv.negative_reviews, 0)
            / nullif(coalesce(rv.total_reviews, 0), 0),
            2
        )                                                   as negative_review_pct,

        -- Delivery
        round(
            100.0 * coalesce(d.late_deliveries, 0)
            / nullif(coalesce(d.delivered_orders, 0), 0),
            2
        )                                                   as late_delivery_pct,

        -- Composite score: simple weighted score (0–100)
        round(
            (coalesce(rv.avg_review_score, 3) / 5.0 * 50)
            + (
                1.0 - coalesce(d.late_deliveries, 0)::float
                / nullif(coalesce(d.delivered_orders, 1), 0)
            ) * 50,
            1
        )                                                   as seller_score,

        current_timestamp                                   as dbt_updated_at

    from sellers s
    left join seller_revenue  r  on s.seller_id = r.seller_id
    left join seller_reviews  rv on s.seller_id = rv.seller_id
    left join seller_delivery d  on s.seller_id = d.seller_id
)

select * from final
```

This model follows the common design pattern introduced earlier. Three independent aggregation CTEs—`seller_revenue`, `seller_reviews`, and `seller_delivery`—each summarize data at the `seller_id` level before being combined through `LEFT JOIN`s with the `sellers` dimension table in the `final` CTE. Because each branch has already been aggregated to one row per seller, the final joins do not introduce fanout, despite the one-to-many relationships between sellers and their orders, reviews, and deliveries.

The `seller_score` metric consists of two equally weighted components, each contributing up to 50 points:

- **Customer Review Score (50%)**: The average customer review score (on a five-point scale), linearly normalized to a maximum of 50 points using `avg_review_score / 5.0 * 50`.
- **On-Time Delivery Score (50%)**: The proportion of orders delivered on time, converted to a maximum of 50 points using `(1 - late_deliveries / delivered_orders) * 50`.

A seller with perfect customer ratings and no late deliveries receives a `seller_score` of **100**, while the score decreases proportionally as review ratings decline or the late delivery rate increases.

For newly onboarded sellers who have not yet received customer reviews or completed deliveries, the calculation uses `coalesce(rv.avg_review_score, 3)` to assign a neutral default rating (3 out of 5) instead of zero. Likewise, `nullif(..., 1)` prevents division-by-zero errors when `delivered_orders` equals zero. This approach provides new sellers with a reasonable baseline score instead of unfairly assigning a score of zero or causing runtime exceptions. The constraint requiring `seller_score` to remain within the range of **0–100** is automatically validated through dbt tests, as described in Section 5.4.3.

![Query Result of `select seller_id, seller_city, seller_score, late_delivery_pct from mart.seller_performance`](image-1.png)

![Query Result of `select order_date, total_orders, gmv, total_revenue, cancellation_rate_pct from mart.revenue_daily`](image-2.png)
````
