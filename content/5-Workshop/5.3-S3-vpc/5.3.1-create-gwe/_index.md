---
title: "5.3.1 - Creating the S3 Bucket & Amazon Redshift Cluster"
weight: 1
---

### Step 1: Create an Amazon S3 Bucket for Raw Data

1. Open the **Amazon S3 Console** and choose **Create bucket**.
2. Enter the bucket name: `ecom-raw-data-lake-prod`.
3. Keep the default settings and click **Create bucket**.

![Create S3 Bucket](images/5.3.1-s3-bucket.png)

> 📸 **Screenshot Suggestion 5.3.1a:** Capture the Amazon S3 Buckets page showing that the `ecom-raw-data-lake-prod` bucket has been created successfully.

---

### Step 2: Create an Amazon Redshift Cluster

1. Open the **Amazon Redshift Console** and choose **Create cluster**.
2. Configure the following settings:
   - **Cluster identifier:** `redshift-ecom-dw`
   - **Database name:** `dev`
   - **Admin user:** `awsuser`
3. After the cluster has been provisioned, create the required schemas in Redshift:

```sql
CREATE SCHEMA staging;
CREATE SCHEMA mart;
```