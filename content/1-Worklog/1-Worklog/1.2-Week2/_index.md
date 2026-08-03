---
title: "Week 2 Worklog"
date: 2026-06-08
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2 Objectives:

* Go deeper into the three AWS services the pipeline depends on most: S3, IAM, and Redshift Serverless.
* Understand the medallion architecture (Bronze–Silver–Gold) and decide how it maps onto S3 + Redshift.
* Survey the Olist Brazilian E-Commerce dataset in detail: 9 related CSV files, schema, relationships, data quality issues.
* Draft the overall system architecture diagram before writing any code.

### Tasks to be carried out this week:
| Day | Task                                                                                                                                                                                                   | Start Date | Completion Date | Reference Material                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | ----------------------------------------- |
| 2   | - Learn Amazon S3: buckets, object storage, prefixes/partitioning, lifecycle policies, public access block                                                                                              | 06/08/2026 | 06/08/2026      | AWS S3 User Guide |
| 3   | - Learn IAM: users, roles, policies, principle of least privilege <br> - **Practice:** create an IAM Role that lets Redshift assume a role to read from S3 (no static access keys)                     | 06/09/2026 | 06/09/2026      | AWS IAM Documentation |
| 4   | - Learn Amazon Redshift Serverless: RPU, workgroups/namespaces, DISTKEY vs SORTKEY, COPY command <br> - **Practice:** provision a small Redshift Serverless workgroup manually via Console             | 06/10/2026 | 06/10/2026      | AWS Redshift Serverless docs |
| 5   | - Study Data Warehouse concepts: OLTP vs OLAP, medallion architecture (Bronze/Silver/Gold), ETL vs ELT <br> - Decide: this project follows ELT (load raw into Redshift first, transform with dbt after) | 06/11/2026 | 06/11/2026      | Kimball, *The Data Warehouse Toolkit* |
| 6   | - Download and explore the Olist Brazilian E-Commerce dataset (9 CSV files) <br> - Map relationships between orders, customers, order_items, sellers, products, payments, reviews <br> - Sketch the overall pipeline architecture diagram | 06/12/2026 | 06/13/2026 | <https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce> |


### Week 2 Achievements:

* Understood S3 as the Bronze storage layer: how date-partitioned prefixes (`year=/month=/day=`) work, and why lifecycle rules matter for cost control.

* Created my first IAM Role for Redshift to assume when reading from S3 — understood why this is preferred over static access keys (reduces the risk of credential leakage).

* Provisioned a Redshift Serverless workgroup manually and ran a first test query, to understand RPU-based scaling before automating it with Terraform later.

* Learned the difference between DISTKEY (row distribution across nodes) and SORTKEY (physical sort order on disk), and why choosing the wrong one hurts JOIN/filter performance at scale.

* Understood the medallion architecture and settled on the design: Bronze = raw CSVs on S3, Silver = raw tables in Redshift `staging` schema, Gold = dbt-modeled `mart` tables.

* Decided on ELT over ETL: since Redshift can process data in parallel (MPP), it's more efficient to load raw data first and transform it with SQL inside the warehouse, rather than transforming outside with a separate compute engine.

* Fully mapped the Olist dataset: 9 CSV files covering the full order lifecycle (customer → order → order_items → payment → review), plus a product category translation table (Portuguese → English) and a geolocation table.

* Produced a first draft of the system architecture diagram (Kaggle → S3 → Redshift staging → dbt staging/mart), which became the basis for Figure 3.1 in the final report.
