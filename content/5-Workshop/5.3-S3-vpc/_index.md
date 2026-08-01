---
title: "Deploying the Storage Layer and Data Warehouse on AWS"
date: 2026-07-22
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

This section describes the deployment of the two core infrastructure components of the system on AWS: the raw data storage layer using **Amazon S3** and the cloud data warehouse using **Amazon Redshift Serverless**. It also covers the configuration of private connectivity between these services through a **Gateway VPC Endpoint**.

Before discussing the implementation details, it is important to clarify one key architectural decision. According to the Terraform configuration (`infrastructure/terraform/main.tf`), the project's **Amazon Redshift Serverless Workgroup** is created with the attribute `publicly_accessible = false`. This means the workgroup is assigned only private network interfaces within the VPC's private subnets, has no public IP address, and cannot be accessed directly from the public Internet.

From a security perspective, this is the recommended design. Since the data warehouse stores transactional information related to customers and sellers, it should never be directly exposed to the Internet.

However, this design introduces an important networking requirement. When Redshift executes the `COPY` command to load data from **Amazon S3**, it must communicate with an AWS service that exists outside the VPC by default. Without additional network configuration, the `COPY` operation would either fail or require routing traffic through a **NAT Gateway**, resulting in unnecessary operational costs. Since Amazon S3 is itself an AWS-managed service, routing traffic through the public Internet is neither required nor desirable.

The appropriate solution is to configure a **Gateway VPC Endpoint** for Amazon S3. Unlike interface endpoints, a Gateway Endpoint operates at the **Route Table** level, allowing resources inside the VPC—including Amazon Redshift Serverless—to access Amazon S3 entirely through the AWS private backbone network. As a result, traffic never traverses the public Internet, and no NAT Gateway data processing charges are incurred.

Section 5.3 is divided into two subsections that follow the actual deployment sequence:

- **5.3.1 – Provisioning the Storage Infrastructure and Private Connectivity to Amazon S3** (`5.3.1-create-gwe/`)  
  Covers the creation of the Amazon S3 buckets, Amazon Redshift Serverless resources, and the configuration of the Gateway VPC Endpoint.

- **5.3.2 – Verifying Connectivity and Configuring the dbt Profile** (`5.3.2-test-gwe/`)  
  Demonstrates how to verify the private connection by configuring the dbt profile and testing connectivity between dbt and Amazon Redshift.
