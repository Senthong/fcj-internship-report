---
title: "Week 4 Worklog"
date: 2026-06-22
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Week 4 Objectives:

* Build the Silver layer: load the 9 raw CSVs from S3 into 8 raw tables in Redshift's `staging` schema.
* Design DDL for each raw table, choosing DISTKEY/SORTKEY per table based on expected query patterns.
* Implement the Truncate-and-Load strategy, wrapped in a single atomic transaction across all 8 tables.
* Prefer IAM Role authentication for the `COPY` command over static access keys.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                                   | Start Date | Completion Date | Reference Material                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | ----------------------------------------- |
| 2   | - Design the `TableSpec` structure (csv_file, table_name, dist_key, sort_key, ddl) to declare schema + physical optimization in one place                                                              | 06/22/2026 | 06/22/2026      | |
| 3   | - Write DDL for all 8 raw tables (`raw_orders`, `raw_order_items`, `raw_customers`, `raw_sellers`, `raw_products`, `raw_order_payments`, `raw_order_reviews`, `raw_geolocation`)                       | 06/23/2026 | 06/23/2026      | |
| 4   | - Choose DISTKEY/SORTKEY per table: `raw_orders` → DISTKEY(order_id)/SORTKEY(order_purchase_timestamp) since it's the most-JOINed, most time-filtered table; small dimension tables → DISTSTYLE ALL     | 06/24/2026 | 06/24/2026      | AWS Redshift best practices |
| 5   | - Implement `truncate_and_load()`: CREATE TABLE IF NOT EXISTS → TRUNCATE → COPY FROM S3 <br> - Implement `_auth_clause()` to prefer `REDSHIFT_IAM_ROLE_ARN` over access key/secret key                 | 06/25/2026 | 06/25/2026      | |
| 6   | - Wrap all 8 table loads in a single `autocommit=False` transaction with commit/rollback <br> - **Test:** intentionally corrupt one CSV row to trigger `MAXERROR`, confirm all 8 tables roll back together | 06/26/2026 | 06/27/2026 | |


### Week 4 Achievements:

* Completed `ingestion/load_to_redshift.py`, with 8 `TableSpec` definitions that keep schema, DISTKEY, and SORTKEY together — so the "why" of each optimization choice lives right next to the DDL instead of being scattered.

* Understood the practical difference between DISTKEY and SORTKEY hands-on: after loading `raw_orders` with `DISTKEY(order_id)`, JOINs against `raw_order_items` (also keyed on `order_id`) avoided a broadcast/redistribute step, since Redshift's `EXPLAIN` plan showed no `DS_BCAST_INNER` step.

* Implemented the Truncate-and-Load strategy: since `staging` is a pass-through layer (not a historical store), each run safely replaces the previous day's data rather than accumulating duplicates.

* Verified transactional atomicity: deliberately injected a malformed row into one CSV to trip Redshift's `MAXERROR` threshold on `COPY`, confirmed via `conn.rollback()` that none of the 8 tables retained partial data — the staging schema stayed at its previous consistent state.

* Set up IAM Role authentication (`REDSHIFT_IAM_ROLE_ARN`) as the primary path for `COPY`, with access key/secret key kept only as a fallback — matches the least-privilege principle learned in Week 2.

* First successful end-to-end run of Bronze → Silver: S3 → 8 tables in Redshift `staging` schema, verified row counts against the source CSVs to confirm no data loss.
