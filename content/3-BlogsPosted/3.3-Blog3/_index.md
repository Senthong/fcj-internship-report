---
title: "Blog 3: Optimizing AWS EC2 Costs with AWS Graviton"
date: 2026-07-06
weight: 6
chapter: false
---

# [FCAJ2026] Optimizing AWS EC2 Costs by Migrating to AWS Graviton (ARM)

## Introduction

When building cloud infrastructure, optimizing operational costs is just as important as improving application performance.

While evaluating infrastructure for a small production and testing environment, our team explored whether migrating workloads from traditional **x86-based Amazon EC2 instances** to **AWS Graviton (ARM)** could provide measurable cost savings without sacrificing performance.

After benchmarking several containerized microservices built with **Node.js** and **Go**, the results showed that AWS Graviton delivered both lower infrastructure costs and improved resource utilization.

---

## Benchmark Results

The experiment was conducted using two EC2 instances hosting multiple containerized microservices.

### Migration Scenario

- Original architecture: **t3.medium (x86)**
- New architecture: **t4g.medium (AWS Graviton2 ARM)**

The comparison produced the following results.

| Metric | t3.medium | t4g.medium |
|--------|----------:|-----------:|
| On-Demand Price | $0.0416/hour | $0.0336/hour |
| Monthly EC2 Cost | ~$60 | ~$48 |
| Average CPU Utilization | ~45% | ~30% |

The migration achieved:

- **19% lower hourly infrastructure cost**
- **Approximately 33% lower CPU utilization**
- Better overall resource efficiency for the same workloads

---

## Right-Sizing Opportunities

Lower CPU utilization created an opportunity to further optimize infrastructure.

Instead of running:

- 2 × t3.medium

the workload could comfortably operate on:

- 1 × t4g.medium
- 1 × t4g.small

This additional right-sizing reduced monthly infrastructure costs even further.

| Architecture | Monthly Cost |
|--------------|-------------:|
| Original | ~$60 |
| Optimized | ~$42 |

Overall, the migration reduced EC2 costs by approximately **30%**, saving around **$18 per month** for this small environment.

Although the savings may appear modest, the impact becomes substantial when applied across dozens or hundreds of EC2 instances.

---

## Migration Process

### Build Multi-Architecture Docker Images

The CI/CD pipeline was updated using **GitHub Actions** together with Docker Buildx.

Container images are now built for both:

- `linux/amd64`
- `linux/arm64`

This allows the same deployment pipeline to support both Intel/AMD and AWS Graviton instances.

---

### Verify Application Compatibility

Before migrating production workloads, every application dependency was tested on ARM64.

Fortunately, most modern runtimes already provide excellent native ARM support, including:

- Node.js
- Go
- Python
- Java

In our case, no application code changes were required.

---

### Deploy the New Infrastructure

Infrastructure definitions managed with Terraform or CloudFormation were updated to launch AWS Graviton instances.

Traffic was gradually shifted from the existing x86 instances to the new ARM-based instances before the original infrastructure was safely decommissioned.

---

## Why AWS Graviton Performs Better

AWS Graviton processors are built specifically for cloud-native workloads.

Compared with equivalent x86 instances, they typically provide:

- Lower hourly pricing
- Better price-to-performance ratio
- Improved energy efficiency
- Excellent performance for web services and containerized applications

For many API services and backend workloads, Graviton offers sufficient computing power while consuming fewer infrastructure resources.

---

## Lessons Learned

This migration highlighted two important optimization strategies.

### Lower Infrastructure Cost

Simply migrating from x86 instances to AWS Graviton reduced hourly EC2 pricing by nearly 20% without changing application functionality.

---

### Better Resource Utilization

Because CPU utilization decreased significantly after migration, the infrastructure could be right-sized to smaller instance types, producing additional long-term savings.

Instead of treating instance selection as a one-time decision, regularly reviewing workload utilization can uncover meaningful cost optimization opportunities.

---

## Conclusion

Migrating workloads to AWS Graviton is one of the simplest ways to reduce AWS infrastructure costs while maintaining—or even improving—application performance.

For containerized applications built with modern runtimes such as Node.js, Go, Python, or Java, the migration effort is often minimal thanks to widespread ARM64 support.

Combined with proper right-sizing and multi-architecture Docker images, AWS Graviton enables organizations to build more efficient, cost-effective cloud-native infrastructure.

---

## References
Read the full article on **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2233445714087055/)**.

- AWS. **AWS Graviton Processors**  
  https://aws.amazon.com/ec2/graviton/

- AWS. **Amazon EC2 On-Demand Pricing**  
  https://aws.amazon.com/ec2/pricing/on-demand/

- AWS. **AWS Graviton Getting Started Guide**  
  https://github.com/aws/aws-graviton-getting-started

- Docker. **Multi-platform Builds**  
  https://docs.docker.com/build/building/multi-platform/