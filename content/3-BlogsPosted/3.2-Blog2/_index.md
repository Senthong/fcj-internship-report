---
title: "02. Troubleshooting AWS S3 Access Denied (403)"
date: 2026-06-22
weight: 4
chapter: false
---

# [FCAJ2026] Troubleshooting AWS S3 Access Denied (403) on EC2 – Lessons Learned

## Introduction

During the deployment of our cloud-native application on AWS, I encountered one of the most common yet frustrating issues when integrating Amazon S3 with Amazon EC2:

> **AccessDenied (403 Forbidden)**

The application worked perfectly in the local development environment but immediately failed after deployment to EC2.

Initially, I assumed the issue was simply caused by missing permissions. However, after several hours of debugging, I realized that Amazon S3 authorization involves multiple layers of security beyond a single IAM policy.

This article summarizes the root causes, troubleshooting process, and lessons learned from resolving this issue.

---

## Why Did It Work Locally but Fail on EC2?

During development, I authenticated using an **IAM User** configured through the AWS CLI.

The IAM user had the **AmazonS3FullAccess** managed policy attached, so every S3 operation succeeded.

For production deployment, I followed AWS best practices and switched to using an **IAM Role (Instance Profile)** attached directly to the EC2 instance.

Immediately after deployment, every request to Amazon S3 failed with:

```
AccessDenied (403 Forbidden)
```

After investigating the issue, I identified two major causes.

---

## Root Cause 1 – Incorrect Resource ARN

One of the most common mistakes is using the wrong resource scope inside the IAM policy.

For example:

- **Bucket-level actions** such as `s3:ListBucket` require:

```
arn:aws:s3:::my-bucket
```

- **Object-level actions** such as:

- GetObject
- PutObject
- DeleteObject

must instead use:

```
arn:aws:s3:::my-bucket/*
```

Without the `/*` suffix, Amazon S3 immediately rejects object requests with an Access Denied error.

---

## Root Cause 2 – SSE-KMS Encryption

Another hidden issue came from enabling **Server-Side Encryption with AWS KMS (SSE-KMS)**.

Although the IAM Role already had permission to access the S3 bucket, the EC2 instance was **not authorized to use the KMS key**.

As a result:

- Amazon S3 successfully located the object.
- But AWS KMS refused to decrypt it.
- The final response was still **403 Access Denied**.

To resolve the issue, the EC2 IAM Role was added to the **KMS Key Policy** with permissions including:

- `kms:Decrypt`
- `kms:GenerateDataKey`

---

## Troubleshooting Steps

### Step 1 – Review the EC2 IAM Role

Separate bucket-level permissions from object-level permissions.

Bucket actions:

- `s3:ListBucket`

Object actions:

- `s3:GetObject`
- `s3:PutObject`
- `s3:DeleteObject`

Verify that each action references the correct ARN.

---

### Step 2 – Review the KMS Key Policy

If the bucket uses SSE-KMS encryption, ensure that the EC2 IAM Role is explicitly allowed to:

- Decrypt objects
- Generate data keys

Without these permissions, S3 cannot return encrypted objects even if the bucket policy is correct.

---

### Step 3 – Check Other Security Layers

Amazon S3 authorization involves multiple policy evaluation layers.

When debugging Access Denied errors, verify:

- Bucket Policy
- IAM Policy
- KMS Key Policy
- Object Ownership configuration
- Block Public Access settings
- VPC Endpoint Policy (if applicable)

Any explicit **Deny** in one layer overrides all Allow statements.

---

### Step 4 – Configure Monitoring

To reduce future troubleshooting time, I configured Amazon CloudWatch to monitor permission failures.

A Metric Filter searches for **AccessDenied** events and forwards alerts through Amazon SNS.

This allows permission issues to be detected immediately without waiting for users to report failures.

---

## Lessons Learned

This debugging experience highlighted several important AWS security concepts.

- A working local environment does not guarantee a successful EC2 deployment because IAM Users and IAM Roles follow different permission models.

- AccessDenied errors are not always caused by IAM policies. Bucket Policies, KMS Key Policies, Block Public Access settings, Object Ownership, and VPC Endpoint Policies can all affect authorization.

- Always validate permissions using **IAM Policy Simulator** or **IAM Access Analyzer** before deploying applications.

- If Amazon S3 uses **SSE-KMS**, always verify the KMS Key Policy in addition to the S3 permissions.

---

## Conclusion

Although the **S3 Access Denied (403)** error appears simple, troubleshooting it often requires understanding how multiple AWS security services interact.

By carefully reviewing IAM Roles, resource ARNs, Bucket Policies, KMS permissions, and monitoring configurations, I was able to completely resolve the issue and strengthen the security of the application's storage architecture.

Understanding these permission layers not only helps solve deployment problems faster but also leads to more secure and reliable cloud-native applications.

---

## References
Read the full article on **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233435944088032/)**.
- AWS. **Troubleshoot Amazon S3 403 Access Denied Errors**  
  https://repost.aws/knowledge-center/s3-troubleshoot-403

- AWS. **IAM Roles for Amazon EC2**  
  https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html

- AWS. **IAM Policy Simulator**  
  https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html