---
title: "5.2 - Prerequisites"
date: 2026-07-21
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## 1. Minimum Hardware Requirements

To ensure a smooth development experience while running multiple Docker containers simultaneously (Apache Airflow, PostgreSQL) and connecting to AWS cloud services, your local machine should meet the following minimum hardware requirements:

| Component | Minimum Specification | Recommended Specification |
| :--- | :--- | :--- |
| **Operating System** | Windows 10/11 Pro/Enterprise (WSL2), macOS 12+, Ubuntu 20.04 LTS | Windows 11 Pro (WSL2), macOS Apple Silicon (M1/M2/M3), Ubuntu 22.04 LTS |
| **CPU** | 4 Cores / 8 Threads | 8 Cores / 16 Threads |
| **Memory (RAM)** | 8 GB | 16 GB or higher (strongly recommended for Apache Airflow) |
| **Available Disk Space** | 20 GB SSD | 50 GB NVMe SSD |

---

## 2. AWS Account and IAM Configuration

This project relies on two core AWS cloud services: **Amazon S3** and **Amazon Redshift**. Before getting started, prepare your AWS account and configure the required permissions.

### 2.1. Active AWS Account

- An active AWS account with billing verification completed.
- Access to the **AWS Management Console**.

### 2.2. Create an IAM User and Access Keys

- Create a dedicated IAM user for this workshop (e.g., `data-pipeline-admin`).
- Grant the IAM user the following minimum policies:
  - `AmazonS3FullAccess`: Permissions to create, read, write, and delete data in Amazon S3 buckets.
  - `AmazonRedshiftFullAccess`: Permissions to create and manage Redshift clusters/serverless workgroups, and execute SQL queries.
- Generate an **Access Key ID** and **Secret Access Key** for **Programmatic Access**. These credentials will be used to authenticate Apache Airflow and the AWS CLI.

![IAM User](5.2-aws-iam-keys.png)

### 2.3. Configure Cloud Resources

1. **Amazon S3 Bucket**
   - Create a new S3 bucket (e.g., `ecom-data-lake-raw-uniquename`).
   - Select the same AWS Region as your Redshift deployment (recommended: `ap-southeast-1` - Singapore).
   - Keep **Block all public access** enabled.

2. **Amazon Redshift Cluster / Serverless**
   - Create either a Redshift Serverless Workgroup or a Provisioned Cluster (recommended node types: `dc2.large` or `ra3.xlplus`).
   - Configure the **Inbound Rules** of the associated **VPC Security Group** to allow inbound traffic on port `5439` from your public IP address or the environment where Apache Airflow is running.

---

## 3. Containerization Environment (Docker)

The entire local data pipeline is packaged and managed using Docker to ensure a consistent runtime environment across different machines.

### 3.1. Install Docker Engine & Docker Compose

- Install **Docker Desktop** (Windows/macOS) or **Docker Engine** with the `docker-compose-plugin` (Linux).
- Ensure Docker Compose version `v2.20.0` or later is installed.

### 3.2. Docker Compose Services

The project is orchestrated through the `docker-compose.yml` file, which includes the following core containers:

1. **Airflow Webserver** – Provides the web-based administration interface (UI) on port `8080`.
2. **Airflow Scheduler** – Scans DAGs, schedules workflows, and dispatches tasks.
3. **Airflow Metadata Database** – Uses PostgreSQL to store Airflow metadata and pipeline state.
4. **Source Database (OLTP)** – A standalone PostgreSQL instance acting as the transactional source database containing the raw Olist E-Commerce dataset.

![Docker Containers Running](5.2-docker-ps.png)

---

## 4. Development Tools and Database Management

### 4.1. Local Python Environment

- Install **Python 3.11** or later on your local machine for developing Python scripts and running the dbt CLI during testing.
- Install the dbt adapter for Amazon Redshift:

```bash
pip install dbt-core dbt-redshift
```

### 4.2. Visual Studio Code and Required Extensions

Use **Visual Studio Code** as your primary Integrated Development Environment (IDE). Install the following extensions:

- **Python (`ms-python.python`)** – Provides IntelliSense, linting, debugging, and Python language support.
- **dbt Power User (`innoverio.vscode-dbt-power-user`)** – Supports compiled SQL preview, lineage graph visualization, and quick execution of dbt models.
- **Docker (`ms-azuretools.vscode-docker`)** – Manage Docker containers, images, and logs directly within VS Code.
- **YAML (`redhat.vscode-yaml`)** – Validates YAML syntax for configuration files such as `docker-compose.yml`, `profiles.yml`, and `schema.yml`.

---

## 5. Readiness Checklist

Before proceeding to the next section, ensure you have successfully completed all items in the checklist below.

| No. | Checklist Item | Verification Command / Action | Status |
| :-- | :--- | :--- | :--- |
| 1 | Docker Engine & Docker Compose | `docker --version` and `docker compose version` | ✅ Pass |
| 2 | Project Containers | `docker ps` displays all Airflow and PostgreSQL containers | ✅ Pass |
| 3 | AWS CLI Authentication | `aws sts get-caller-identity` returns your IAM user information | ✅ Pass |
| 4 | Amazon S3 Connection | `aws s3 ls s3://<your-bucket-name>` executes without errors | ✅ Pass |
| 5 | dbt Core CLI | `dbt --version` displays both `dbt-core` and `dbt-redshift` versions | ✅ Pass |
| 6 | Airflow Web UI | Successfully log in at `http://localhost:8080` | ✅ Pass |