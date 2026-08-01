---
title: "System Overview"
date: 2026-07-20
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

## 1. Detailed Objectives

This introductory section outlines the specific technical objectives of the implementation, providing the basis for evaluating the overall success of the system in the concluding chapter. Upon completing the implementation presented in this chapter, the system will be capable of the following:

- Operating a complete **ELT (Extract – Load – Transform)** data pipeline, where data is extracted from the source, loaded directly into the data warehouse, and transformed only after being stored in the warehouse, rather than being transformed in an intermediate layer as in the traditional ETL architecture.

- Automating the entire workflow—from data ingestion and loading to data transformation and data quality testing—using **Apache Airflow**, with support for automatic retries when temporary failures occur and pipeline termination when data quality checks fail.

- Organizing data into clearly defined processing layers using **dbt Core**, including the **Staging** layer for cleaning and standardizing raw data and the **Data Mart** layer for aggregating data to answer specific business questions.

- Applying **Data Quality Testing** at multiple levels, including uniqueness and referential integrity checks, valid domain value validation, and proactive handling of **fanout joins**—a common issue in one-to-many joins that can produce duplicated aggregation results.

- Delivering a collection of **Data Mart** tables that accurately represent key business metrics for an e-commerce platform, including daily revenue, revenue by product category, seller performance, and customer retention by cohort.

---

## 2. Overall System Architecture

The system is designed using the **Medallion Architecture (Bronze – Silver – Gold)** together with dbt's familiar **Staging – Data Mart** organization. This layered architecture is chosen instead of transforming data directly from the source into reporting tables for two primary reasons. First, it clearly separates raw, immutable data—which can always be traced back and reprocessed—from transformed data. Second, it minimizes duplicated transformation logic, as fundamental transformations are implemented once in the Staging layer and then reused by downstream Data Mart models through dbt's `ref()` function.

The overall data processing workflow is summarized below:

```text
Olist Dataset (Kaggle)
        │
        ▼
Python Ingestion Script
(downloads data, validates it, uploads to S3)
        │
        ▼
Amazon S3 - Raw Bucket
(Bronze Layer, partitioned by year/month/day)
        │
        ├──────────────► AWS Glue Crawler ──► AWS Glue Data Catalog
        │                    (supports ad-hoc queries through Amazon Athena)
        ▼
Amazon Redshift Serverless
Staging Schema (Silver Layer, loaded using COPY)
        │
        ▼
dbt Core - Staging Models
(cleaning, standardization, derived columns)
        │
        ▼
dbt Core - Data Mart Models
(Gold Layer, business-oriented aggregations)
        │
        ▼
Amazon Redshift Serverless
Analytics Schema
(reporting-ready Data Mart tables)
```

The entire workflow is orchestrated by **Apache Airflow**, which executes the pipeline on a daily schedule to ensure that the data warehouse remains continuously up to date without manual intervention.

![Diagram](diagram.png)

---

## 3. Roles of the Technology Components

The following table summarizes the role of each technology used in the project. Unlike the initial conceptual design, the final implementation does **not** use a standalone PostgreSQL database as the OLTP source system. Instead, data is ingested directly from the Olist dataset through the Kaggle API. PostgreSQL is used solely as Airflow's internal metadata database.

| Component | Role in the System |
|-----------|--------------------|
| **Olist Dataset (Kaggle)** | Primary data source that simulates the transactional data of a real-world e-commerce platform, including orders, products, sellers, customers, payments, and customer reviews. |
| **Ingestion Script (Python, boto3, pandas)** | Downloads data from Kaggle, performs preliminary validation (row count, column count, missing value ratio), and uploads the files to Amazon S3 using a date-based partitioning structure. |
| **Amazon S3** | Serves as the Data Lake, storing both raw data (Raw Bucket, Bronze layer) and intermediate processed data (Processed Bucket, Silver layer) in CSV format. |
| **AWS Glue Crawler & Glue Data Catalog** | Automatically scans and infers the schema of raw data stored in S3, enabling ad-hoc querying through Amazon Athena without loading data into Redshift. This workflow is independent of the main processing pipeline. |
| **Amazon Redshift Serverless** | Cloud data warehouse that stores both the **Staging** schema (loaded but minimally transformed data) and the **Analytics** schema (Data Mart tables for reporting). |
| **dbt Core** | Implements all transformation logic, manages version-controlled data models, organizes Staging and Data Mart layers, and executes data quality tests. |
| **Apache Airflow** | Orchestrates, schedules, and monitors the entire pipeline. The PostgreSQL instance bundled with Airflow stores only Airflow metadata (DAG history and task states) and does not contain business data. |
| **Terraform** | Manages AWS infrastructure as code (Infrastructure as Code), ensuring consistent and reproducible deployments across development and production environments. |
| **Docker & Docker Compose** | Containerize the Airflow environment and supporting services, ensuring consistency between local development and deployment environments. |

---

## 4. Architectural Considerations

Compared with an architecture that stages data through an intermediate OLTP database before loading it into the data warehouse, directly ingesting data from Kaggle into Amazon S3 significantly simplifies the infrastructure while remaining well suited to the characteristics of the Olist dataset. Since Olist is a static dataset distributed as CSV files rather than a continuously operating transactional system, this approach avoids unnecessary complexity.

In a production e-commerce environment with a live transactional database, however, the ingestion layer would typically be replaced by an **incremental extraction** mechanism from the OLTP database, such as **Change Data Capture (CDC)** or timestamp-based incremental queries. This represents a natural direction for extending the project in future iterations.

Another notable design choice is the addition of **AWS Glue Crawler** alongside the primary data loading pipeline into Redshift. These two data paths operate independently and serve different purposes. **Amazon Redshift** is optimized for structured, high-frequency analytical workloads and powers the Data Mart layer, while **AWS Glue Data Catalog** together with **Amazon Athena** enables exploratory analysis directly on raw data stored in S3 without incurring the cost of loading the data into the warehouse. This capability is particularly valuable for quickly validating or investigating raw datasets before deciding whether they should enter the primary processing pipeline.