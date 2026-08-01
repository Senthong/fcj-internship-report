---
title : "5.4.1 - Chuẩn bị Tầng Staging (Ingestion & Staging Models)"
date : 2026-07-23
weight : 1
chapter : false
pre : " <b> 5.4.1 </b> "
---
### 1. Nạp dữ liệu vào Redshift Staging
Dữ liệu từ Postgres được nạp vào các bảng tạm (Staging) trên Redshift thông qua Airflow task.

### 2. Định nghĩa dbt Staging Models
Tạo các model làm sạch dữ liệu ban đầu trong thư mục `models/staging/`:
* `stg_orders.sql`
* `stg_order_items.sql`
* `stg_sellers.sql`
* `stg_order_reviews.sql`

![Staging Models Tree](image.png)
