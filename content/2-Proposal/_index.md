---
title: "Proposal"
date: 2026-06-08
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

In this section, the proposed project and implementation plan are presented.

# E-Commerce Data Pipeline
## A Modern ELT Pipeline on AWS for Analytics and Business Intelligence

### 1. Executive Summary

The **E-Commerce Data Pipeline** is designed to automate the ingestion, transformation, and analysis of e-commerce data using a modern cloud-based ELT architecture. The project utilizes the Brazilian Olist public dataset as the data source and leverages AWS cloud services together with Apache Airflow, dbt, Amazon Redshift, and Terraform.

The platform automatically ingests raw datasets into Amazon S3, loads them into Amazon Redshift, performs data transformations using dbt, and produces analytical data marts for business reporting. The entire infrastructure is provisioned using Infrastructure as Code (Terraform), while CI/CD is automated through GitHub Actions.

The proposed solution demonstrates modern Data Engineering practices including workflow orchestration, cloud data warehousing, automated transformations, infrastructure automation, and reproducible deployments.

---

## 2. Problem Statement

### What's the Problem?

Many organizations collect large volumes of transactional data every day. However, raw operational databases are not optimized for business analytics.

Typical challenges include:

- Manual ETL processes
- Inconsistent data quality
- Difficulties maintaining SQL transformation logic
- Lack of workflow scheduling
- Poor scalability for analytical workloads

Without a proper data pipeline, generating reports becomes time-consuming and error-prone.

### The Solution

This project implements an automated ELT pipeline using AWS cloud services.

The pipeline performs the following steps:

1. Ingest raw CSV datasets into Amazon S3.
2. Load raw data into Amazon Redshift.
3. Schedule workflows using Apache Airflow.
4. Transform raw tables into analytics-ready models using dbt.
5. Generate business-oriented data marts for reporting.
6. Provision cloud infrastructure automatically using Terraform.
7. Validate code quality through GitHub Actions CI.

The solution follows modern Data Engineering best practices by separating raw, staging, and mart layers.

### Benefits and Return on Investment

The proposed platform provides several advantages:

- Fully automated data pipeline
- Reproducible infrastructure deployment
- Modular SQL transformations using dbt
- Easier maintenance and scalability
- Faster business reporting
- Demonstration of industry-standard Data Engineering architecture

Although this project is educational, the architecture can be extended for production-scale workloads.

---

## 3. Solution Architecture

The solution follows a modern ELT architecture deployed on AWS.

Pipeline flow:

```
CSV Dataset
      │
      ▼
 Amazon S3
      │
      ▼
Amazon Redshift
      │
      ▼
 Apache Airflow
      │
      ▼
      dbt
      │
      ▼
 Analytics Data Marts
```

*(Insert architecture diagram here)*

### AWS Services Used

- **Amazon S3** – Stores raw datasets.
- **Amazon Redshift** – Cloud data warehouse.
- **AWS IAM** – Access control.
- **Terraform** – Infrastructure provisioning.
- **GitHub Actions** – Continuous Integration.

### Open-source Technologies

- Apache Airflow
- dbt
- Docker Compose
- Python
- SQL
- GitHub

### Component Design

#### Data Ingestion

Python scripts upload raw Olist datasets into Amazon S3 before loading them into Amazon Redshift.

#### Workflow Orchestration

Apache Airflow schedules and manages the complete pipeline, including ingestion and transformation tasks.

#### Data Warehouse

Amazon Redshift stores raw and transformed datasets for analytical queries.

#### Data Transformation

dbt transforms raw tables into clean staging models and business-oriented marts.

Current models include:

**Staging Layer**

- Customers
- Orders
- Order Items
- Order Payments
- Order Reviews
- Products
- Sellers

**Mart Layer**

Finance

- Daily Revenue
- Monthly Category Revenue

Operations

- Seller Performance

Customers

- Customer Cohort Analysis

#### Infrastructure

Terraform provisions cloud infrastructure, ensuring reproducible deployments.

#### CI/CD

GitHub Actions automatically validate the project whenever code is pushed.

---

## 4. Technical Implementation

### Implementation Phases

The project follows four major phases.

#### Phase 1 — Research and Design

- Study AWS Data Engineering services
- Design ELT architecture
- Define warehouse schema

#### Phase 2 — Infrastructure Deployment

- Configure Terraform
- Create S3 bucket
- Provision Redshift
- Configure IAM

#### Phase 3 — Pipeline Development

- Develop ingestion scripts
- Build Airflow DAG
- Load data into Redshift
- Develop dbt models

#### Phase 4 — Testing and Deployment

- Validate transformations
- Test Airflow DAG
- Configure GitHub Actions
- Final documentation

### Technical Requirements

Development environment:

- Python
- Docker
- Apache Airflow
- dbt Core
- Terraform
- AWS CLI

AWS Services:

- Amazon S3
- Amazon Redshift
- IAM

Programming Languages:

- Python
- SQL

---

## 5. Timeline & Milestones

### Project Timeline

**Month 1**

- Research architecture
- Learn AWS services
- Design pipeline

**Month 2**

- Deploy infrastructure
- Develop ingestion pipeline
- Configure Airflow

**Month 3**

- Develop dbt models
- Testing
- Documentation
- Final deployment

---

## 6. Budget Estimation

Since this project is intended for learning purposes, AWS Free Tier resources are used whenever possible.

### Estimated Infrastructure Cost

- Amazon S3
- Amazon Redshift (small instance)
- Data Transfer
- IAM

Estimated monthly cost:

**Approximately 5–10 USD/month**, depending on Redshift usage and storage.

---

## 7. Risk Assessment

### Risk Matrix

- AWS service misconfiguration — Medium
- Redshift storage limitations — Low
- Airflow scheduling failures — Medium
- SQL transformation errors — Medium

### Mitigation Strategies

- Use Terraform for reproducible deployments.
- Apply GitHub Actions for automated validation.
- Use dbt testing features.
- Monitor Airflow DAG execution.

### Contingency Plans

- Re-run failed DAGs.
- Restore infrastructure using Terraform.
- Reload datasets from Amazon S3.

---

## 8. Expected Outcomes

### Technical Improvements

The project demonstrates an end-to-end modern Data Engineering workflow, including:

- Automated data ingestion
- Cloud data warehousing
- Workflow orchestration
- SQL transformation with dbt
- Infrastructure as Code
- CI/CD automation

### Long-term Value

The project serves as a reusable Data Engineering template for future analytics systems.

It can be extended with:

- Incremental loading
- Data quality monitoring
- Real-time streaming
- Business Intelligence dashboards using Power BI or Amazon QuickSight
- Machine Learning pipelines