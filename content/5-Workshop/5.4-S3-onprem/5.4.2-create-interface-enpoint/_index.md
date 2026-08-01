---
title : "create-interface-enpoint"
date : 2026-07-23
weight : 2
chapter : false
pre : " <b> 5.4.2 </b> "
---

### Build the `seller_performance.sql` Model

This model aggregates performance metrics for each seller, including revenue, customer review ratings, late delivery rate, and a composite **seller_score** ranging from **0 to 100**.

```sql
-- models/mart/operations/seller_performance.sql
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

-- Prevent fanout joins by counting distinct order IDs
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
        s.city as seller_city,

        -- Calculate the Seller Score (0–100 scale)
        round(
            (coalesce(rv.avg_review_score, 3) / 5.0 * 50)
            + (
                1.0
                - coalesce(d.late_deliveries, 0)::float
                / nullif(coalesce(d.delivered_orders, 1), 0)
            ) * 50,
            1
        ) as seller_score,

        current_timestamp as dbt_updated_at

    from sellers s
    left join seller_delivery d
        on s.seller_id = d.seller_id
    left join seller_reviews rv
        on s.seller_id = rv.seller_id
)

select * from final
```

The **seller_score** is calculated using two equally weighted components:

- **Customer Review Score (50%)** – Based on the seller's average review rating, normalized from a 5-point scale to 50 points.
- **On-Time Delivery Score (50%)** – Calculated from the percentage of delivered orders that were not delivered late.

The final score is computed using the following formula:

\[
\text{Seller Score} =
\left(\frac{\text{Average Review Score}}{5} \times 50\right)
+
\left(1 - \frac{\text{Late Deliveries}}{\text{Delivered Orders}}\right) \times 50
\]

As a result:

- A seller with excellent customer ratings and no late deliveries can achieve a maximum score of **100**.
- Sellers with lower review scores or higher late delivery rates will receive proportionally lower scores.

![Seller Score Calculation](image.png)