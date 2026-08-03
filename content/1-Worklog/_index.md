---
title: "Worklog"
date: 2026-06-01
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

**On this page**, I present my worklog for the graduation internship, covering the process of designing and building the **E-Commerce Data Pipeline** project — an end-to-end analytics platform on AWS (S3, Redshift Serverless, IAM, Glue), orchestrated with Apache Airflow, transformed with dbt, provisioned with Terraform, and validated with a GitHub Actions CI pipeline.

The internship was carried out over **8 weeks**, following an iterative approach: starting with AWS fundamentals, then building the pipeline layer by layer (Bronze → Silver → Gold), and finishing with infrastructure-as-code and automated testing.

**Week 1:** [Getting familiar with AWS fundamentals: Console, CLI, EC2, S3, IAM](1.1-week1/)

**Week 2:** [Deep dive into S3, IAM, and Amazon Redshift Serverless; surveying the Olist dataset](1.2-week2/)

**Week 3:** [Building the Bronze layer: Python ingestion script (Kaggle → S3)](1.3-week3/)

**Week 4:** [Building the Silver layer: loading data into Redshift with COPY, DISTKEY/SORTKEY design](1.4-week4/)

**Week 5:** [Building the Gold layer: dbt staging & mart models, data quality tests](1.5-week5/)

**Week 6:** [Orchestrating the pipeline with Apache Airflow (7-task DAG, retries, quality gate)](1.6-week6/)

**Week 7:** [Infrastructure as Code with Terraform and local development with Docker Compose](1.7-week7/)

**Week 8:** [CI/CD with GitHub Actions, end-to-end testing, evaluation, and report finalization](1.8-week8/)
