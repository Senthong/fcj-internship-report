---
title: "Week 5 Worklog"
date: 2026-06-29
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Week 5 Objectives:

* Build the Gold layer with dbt: 7 staging models (views) that clean raw data, and 4 mart models (tables) that apply business logic.
* Write 46 data quality tests (17 at the source, 29 at the model level) as a quality gate before data reaches the mart layer.
* Design the `generate_schema_name` macro for dev/prod schema isolation.
* Understand dbt's `ref()`/`source()` lineage graph and the DRY principle it enables.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                                   | Start Date | Completion Date | Reference Material                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | ----------------------------------------- |
| 2   | - Install dbt-core + dbt-redshift, set up `profiles.yml` (dev/prod targets) <br> - Install `dbt_utils` and `audit_helper` packages <br> - Declare `sources.yml` for the 8 raw Redshift tables, with freshness rules (warn 25h, error 49h) | 06/29/2026 | 06/29/2026      | docs.getdbt.com |
| 3   | - Write 7 staging models (views): `stg_customers`, `stg_orders`, `stg_order_items`, `stg_order_payments`, `stg_order_reviews`, `stg_products`, `stg_sellers` — type casting, normalization, derived flags (`is_delivered`, `is_late_delivery`) | 06/30/2026 | 07/01/2026      | |
| 4   | - Write `revenue_daily` and `category_revenue_monthly` mart models (finance) using `ref()` to join staging models                                                                                       | 07/02/2026 | 07/02/2026      | |
| 5   | - Write `seller_performance` (operations, seller_score formula: 50% review + 50% on-time delivery) and `fct_customer_cohorts` (customers, retention by cohort month) mart models                       | 07/03/2026 | 07/03/2026      | |
| 6   | - Write 46 dbt tests across `schema.yml` files (unique, not_null, accepted_values, `dbt_utils.expression_is_true`) <br> - Write the `generate_schema_name` macro for dev/prod schema separation <br> - **Test:** run `dbt run` + `dbt test`, confirm all 11 models build and all tests pass | 07/04/2026 | 07/05/2026 | |


### Week 5 Achievements:

* Completed all 7 staging models as views — no storage cost, always reflect the latest raw data. `stg_orders` turned out to be the most complex: it computes `is_delivered`, `is_late_delivery`, and `days_late` once, so downstream mart models never repeat that `CASE WHEN` logic.

* Completed all 4 mart models as tables: `revenue_daily`, `category_revenue_monthly`, `seller_performance`, `fct_customer_cohorts` — each with a clearly defined grain (1 row per day, per month+category, per seller, per cohort×month-n respectively).

* Designed the `seller_score` formula in `seller_performance` (50% average review score + 50% on-time delivery rate) — a concrete example of turning several raw signals into one operational scorecard metric.

* Wrote and ran all 46 data quality tests: 17 at the source level (freshness, not_null on raw tables) and 29 at the model level (uniqueness of primary keys, `accepted_values` for enums like `order_status`, range checks like `seller_score BETWEEN 0 AND 100`).

* Verified the dbt lineage graph (`dbt docs generate`) matches the design from Week 2: `stg_orders` and `stg_order_items` are referenced by 3–4 mart models each — a good hands-on demonstration of the DRY principle in dbt.

* Built the `generate_schema_name` macro so that `prod` target creates clean schemas (`staging`, `mart`) while `dev` target automatically prefixes them (`analytics_dev__staging`), preventing schema collisions between developers on the same Redshift cluster.

* Caught and fixed a first real data quality issue: `dbt test` flagged negative values in `item_price` for a handful of rows, which led to adding the `expression_is_true: ">= 0"` test in `stg_order_items` — a direct, practical lesson in why the quality gate exists.
