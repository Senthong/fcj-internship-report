---
title: "Blog 3: AWS S3 Bucket for Digital Healthcare System"
date: 2026-06-15
weight: 3
chapter: false
---

# [FCAJ2026] AWS S3 Bucket: The Cloud-Native Storage "Missing Piece" for Digital Healthcare Systems

## Introduction

During the development of our **Digital Healthcare System**—an AI-assisted medical consultation and appointment management platform—we successfully implemented appointment scheduling, AI-powered consultation, and user authentication. However, another major challenge quickly emerged: **file storage and management**.

The system continuously generates a wide variety of files, including:

- **Patients:** Profile avatars, medical reports, and prescriptions in PDF format.
- **Doctors & Staff:** Medical licenses, professional certificates, and supporting documents.
- **AI Models:** Model weight files (`.pt`, `.bin`) that can reach hundreds of megabytes, along with AI consultation chat logs.

At the beginning of development, it was convenient to store uploaded files inside the backend's `uploads/` directory and commit AI model weights directly into the Git repository.

However, when preparing the application for production deployment, this approach quickly became unsustainable:

- The Git repository became unnecessarily large.
- The backend server turned into a **stateful** storage server.
- Sensitive medical files faced significant security risks.
- Scaling the application across multiple servers became much more difficult.

To solve these problems, the project adopted **Amazon Simple Storage Service (Amazon S3)** as the centralized cloud storage solution.

---

## Practical Application in the Project

Rather than allowing the backend server to store every uploaded file, all file-based resources were migrated to Amazon S3.

### Securing Doctor Practice Certificates

Medical certificates and professional licenses contain highly sensitive personal information.

These documents are stored inside a **private S3 bucket**.

Whenever administrators need to review a doctor's credentials, the NestJS backend generates a **Presigned URL** with a short expiration time.

Once the URL expires, access is automatically revoked, ensuring that confidential documents remain protected.

---

### Managing AI Model Weights

AI model weight files (`model.pt`) are completely separated from the application source code.

Instead of committing large files into Git, all training checkpoints and model weights are stored in Amazon S3.

When the AI service starts, a Python initialization script automatically downloads the latest model weights from S3 before loading the model into memory.

This architecture simplifies model version management while keeping the source repository clean and lightweight.

---

### Storing Medical Reports and Chat Logs

After every AI consultation session, the system automatically:

- Generates a PDF medical report.
- Saves the AI conversation history.
- Uploads both resources directly to Amazon S3.

These files can later be downloaded by authorized users or retained for long-term storage and auditing purposes.

---

## Benefits of Migrating to Amazon S3

Integrating Amazon S3 provided several significant advantages for the project.

### Stateless Backend Architecture

The NestJS backend now focuses entirely on business logic and API processing.

Since all static files are stored externally, backend servers remain stateless and can be scaled horizontally without worrying about synchronizing uploaded files between instances.

---

### Enhanced Security for Medical Data

Sensitive healthcare documents are never exposed through public file paths.

Instead, access is controlled through:

- AWS IAM permissions
- Private S3 buckets
- Time-limited Presigned URLs

This ensures that only authorized users can access protected medical documents.

---

### High Reliability with Cost Optimization

Amazon S3 offers industry-leading durability of **99.999999999% (11 nines)**.

In addition, AWS provides a generous Free Tier, allowing projects to store up to **5 GB** during the first 12 months.

By combining S3 Lifecycle Rules with storage classes such as **Amazon S3 Glacier**, temporary logs and infrequently accessed files can be archived automatically, keeping storage costs extremely low.

---

## Conclusion

Adopting cloud-native storage services such as Amazon S3 is an essential architectural practice for modern applications.

Instead of transforming backend servers into large file repositories with complex synchronization and security concerns, developers can delegate storage responsibilities to AWS.

This approach results in a more secure, scalable, and maintainable system while allowing engineering teams to focus entirely on delivering valuable business features rather than managing storage infrastructure.

---

## References

- AWS. **Amazon Simple Storage Service (S3) User Guide**  
  https://docs.aws.amazon.com/AmazonS3/latest/userguide/

- AWS. **Amazon S3 Overview**  
  https://aws.amazon.com/s3/