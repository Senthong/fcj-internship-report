---
title: "5.4.1 - Preparing the Staging Layer (Ingestion & Staging Models)"
date: 2026-07-23
weight: 1
chapter: false
pre: " <b> 5.4.1 </b> "
---

### 1. Load Data into the Redshift Staging Layer

Data is extracted from PostgreSQL and loaded into staging tables in Amazon Redshift through an Apache Airflow task. These staging tables serve as the foundation for subsequent transformation processes.

### 2. Define dbt Staging Models

Create the initial data cleaning and standardization models under the `models/staging/` directory:

- `stg_orders.sql`
- `stg_order_items.sql`
- `stg_sellers.sql`
- `stg_order_reviews.sql`

These staging models are responsible for performing lightweight transformations such as renaming columns, standardizing data types, filtering invalid records, and preparing the raw data for downstream data marts.

![Staging Models Tree](image.png)