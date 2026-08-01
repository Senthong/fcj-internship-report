---
title: "5.4.3 - Workflow Orchestration with Airflow & Data Quality Testing"
date: 2026-07-24
weight: 3
chapter: false
pre: " <b> 5.4.3 </b> "
---

### 1. Configure the Airflow DAG (`ecom_full_pipeline_dag`)

Apache Airflow orchestrates the end-to-end data pipeline by executing the following tasks in sequence:

1. `extract_postgres_to_s3` – Extract data from PostgreSQL and upload it to Amazon S3.
2. `s3_to_redshift_staging` – Load the extracted data from Amazon S3 into the Amazon Redshift staging layer.
3. `dbt_run_mart` – Execute dbt models to transform staging data into analytics-ready data marts.
4. `dbt_test_mart` – Run dbt data quality tests to validate the transformed models.

This workflow ensures that data is extracted, loaded, transformed, and validated in a fully automated and repeatable pipeline.

### 2. Execute Data Quality Tests

Run the `dbt test` command to validate data quality and enforce model constraints.

For example, verify that the `seller_score` column always falls within the valid range of **0 to 100**. Additional tests may include checking for unique primary keys, non-null values, and referential integrity between models.

![Airflow DAG Grid View](5.4.3-airflow-dag-success.png)

![dbt Test Terminal Log](5.4.3-dbt-test-pass.png)