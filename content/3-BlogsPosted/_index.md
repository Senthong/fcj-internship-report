---
title: "Blogs Posted"
date: 2026-06-01
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

# AWS Study Group Blogs

This repository contains a collection of blog posts that I have published on the [AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj)., covering cloud-native architecture, serverless solutions, security, and AWS services applied to real-world healthcare systems.

### [Blog 1 - AWS Glue (PySpark) vs. AWS Lambda + Polars: When Serverless Outperforms Spark](3-BlogsPosted/3.1-Blog1)

This blog compares two modern AWS ETL architectures through a real-world benchmark: **AWS Glue (PySpark)** versus **AWS Lambda + Polars orchestrated by AWS Step Functions**. It analyzes performance, cost, scalability, and architectural trade-offs, demonstrating how a lightweight serverless design can significantly outperform traditional Spark clusters for gigabyte-to-terabyte scale data pipelines.

Read the full article on **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233435944088032/)**.

![alt text](image-1.png)
### [Blog 2 - Hunting Down and Eliminating "Zombie" Resources on AWS](3-BlogsPosted/3.2-Blog2)

This blog shares a practical experience of identifying and eliminating **"Zombie" resources** on AWS — orphaned resources that continue to incur costs even after their associated EC2 instances have been stopped or terminated. It covers common cost culprits such as unattached EBS volumes, unassociated Elastic IP addresses, old Snapshots/AMIs, idle Load Balancers, and unused NAT Gateways, along with practical AWS CLI commands, cost monitoring, tagging strategies, Infrastructure as Code (IaC), and automated cleanup solutions.

Read the full article on **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2237584380339855/)**.

![alt text](<Screenshot 2026-08-07 193410.png>)
### [Blog 3 - Optimizing AWS EC2 Costs with AWS Graviton (ARM)](3-BlogsPosted/3.3-Blog3)
This blog demonstrates how migrating Amazon EC2 workloads from traditional **x86 instances** to **AWS Graviton (ARM)** can significantly reduce infrastructure costs while improving performance. It covers real-world benchmarking results, migration steps, multi-architecture Docker builds, and best practices for adopting ARM-based cloud-native workloads.
Read the full article on **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233445714087055/)**.
![alt text](image.png)