---
title: "Blog 1: AWS Glue vs AWS Lambda + Polars"
date: 2026-06-29
weight: 5
chapter: false
---

# [FCAJ2026] AWS Glue (PySpark) vs. AWS Lambda + Polars: When a Simple Serverless Architecture Outperforms Spark

## Introduction

Within the Data Engineering community, there is a common assumption that processing tens or hundreds of gigabytes of data on AWS automatically requires **AWS Glue**, **Apache Spark**, or **Amazon EMR**.

However, while redesigning a production data pipeline that processes approximately **100 GB of transaction data per day** stored as compressed Parquet and CSV files on Amazon S3, our team experimented with a different architecture.

Instead of relying on AWS Glue, we built a fully serverless ETL pipeline using:

- AWS Lambda
- Polars
- AWS Step Functions (Distributed Map)

The benchmark results surprised our entire infrastructure team.

---

## Benchmark Results

Both solutions processed exactly the same production dataset.

| Metric | AWS Glue (8 DPUs) | AWS Lambda + Polars |
|---------|------------------:|--------------------:|
| Processing Time | ~11 min 40 sec | ~2 min 15 sec |
| Startup Time | 2–3 minutes | < 800 ms |
| Cost per Run | ~$1.68 | ~$0.11 |
| Code Complexity | High | Low |

Compared with AWS Glue, the serverless solution achieved:

- **5.2× faster execution**
- **Over 93% lower processing cost**
- **Virtually zero infrastructure management**

As a result, AWS Lambda + Polars became our production ETL solution, while AWS Glue remains available for future petabyte-scale workloads.

---

## Why Was Lambda Faster?

The biggest performance difference comes from the underlying execution engine.

AWS Glue is built on Apache Spark, which requires:

- Driver initialization
- Executor provisioning
- JVM startup
- Python-to-JVM communication through Py4J
- Serialization and deserialization

Although Spark excels at massive distributed workloads, these overheads become significant for medium-sized datasets.

Polars, on the other hand, is written entirely in **Rust** and built on **Apache Arrow**.

Its execution engine performs:

- Multi-threaded processing
- Vectorized execution
- Zero-copy memory operations

without any JVM overhead, allowing it to utilize hardware resources much more efficiently.

---

## Divide and Conquer with AWS Step Functions

Instead of processing the entire 100 GB dataset inside one large ETL job, the pipeline adopts a divide-and-conquer strategy.

Data stored in Amazon S3 is partitioned by:

- Year
- Month
- Day
- Hour

AWS Step Functions Distributed Map launches multiple Lambda functions simultaneously.

Each Lambda processes only one partition.

For example:

- 50 Lambda functions
- 10 GB memory each
- Up to 6 vCPUs
- Processing partitions independently

This architecture fully utilizes AWS Lambda concurrency while keeping each execution lightweight.

---

## Eliminating Spark Shuffle

One of Spark's largest performance bottlenecks is **shuffle operations**.

Whenever joins or aggregations require data movement between worker nodes, Spark spends considerable time transferring intermediate data across the network.

Instead, our pipeline minimizes network communication by designing the S3 data layout carefully.

Using partition pruning during ingestion allows every Lambda function to read only the partitions it actually needs.

No expensive shuffle operations are required.

---

## Benefits Beyond Performance

### Lower Cloud Costs

AWS Lambda follows a true pay-as-you-go pricing model.

For organizations running hundreds of ETL pipelines every day, reducing execution costs by more than 90% can translate into thousands of dollars saved each month.

---

### Zero Cluster Management

Unlike Spark clusters, the Lambda architecture requires no:

- Cluster provisioning
- JVM tuning
- Executor configuration
- Memory optimization
- Spark version management

The deployment simply consists of lightweight Docker container images running Python and Polars.

---

### Easier Development Experience

Running PySpark locally often requires additional configuration and substantial system resources.

Polars runs directly as standard Python code.

Developers can execute, debug, and test locally with minimal setup before deploying the same container image to AWS Lambda.

---

## Is AWS Glue Still Relevant?

Absolutely.

This benchmark does **not** suggest that Apache Spark or AWS Glue are obsolete.

AWS Glue remains the preferred solution for:

- Petabyte-scale processing
- Large distributed joins
- Memory-intensive workloads
- Complex enterprise data lakes

Spark is designed for problems that cannot fit into the resources available to individual compute instances.

---

## Conclusion

For many enterprise ETL workloads ranging from gigabytes to several terabytes, a serverless architecture built with **AWS Lambda**, **AWS Step Functions**, and modern data processing engines such as **Polars** can provide significantly better performance-to-cost efficiency than traditional Spark-based solutions.

Rather than defaulting to a large distributed cluster, engineers should first ask a simple question:

> **Can this problem be solved more efficiently with a lightweight serverless architecture?**

In many real-world scenarios, the answer may be yes.

---

## References
Read the full article on **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233435944088032/)**.

- AWS. **AWS Lambda Developer Guide**  
  https://docs.aws.amazon.com/lambda/

- Polars. **Blazingly Fast DataFrames Library**  
  https://pola.rs/

- AWS. **AWS Step Functions Distributed Map**  
  https://aws.amazon.com/step-functions/

- Apache Arrow. **Architecture Documentation**  
  https://arrow.apache.org/