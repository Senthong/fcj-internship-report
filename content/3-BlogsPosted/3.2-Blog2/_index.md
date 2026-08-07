---
title: "02. Hunting Down and Eliminating Zombie Resources on AWS"
date: 2026-07-07
weight: 4
chapter: false
---

# [FCAJ2026] Hunting Down and Eliminating "Zombie" Resources on AWS – Practical Cost Optimization Lessons

## Introduction

During application deployment and infrastructure management on AWS, there is a very familiar scenario that almost every Cloud Engineer or DevOps engineer has experienced:

> **The AWS bill at the end of the month suddenly skyrockets, even though you are certain that the test servers (EC2) were shut down last week.**

When checking the details in **AWS Cost Explorer**, you are shocked to discover a series of small charges coming from components that "nobody uses anymore" but are still silently running and incurring costs 24/7. In the Cloud & FinOps community, these resources are known by a very intuitive name: **"Zombie" Resources**.

This article will help you clearly understand the nature of Zombie resources, identify the most common "culprits" causing budget leakage, and follow a step-by-step process to hunt them down, eliminate them, and prevent them from reappearing in your system.

---

## Why Do Zombie Resources Silently Consume Money?

One of the biggest advantages of AWS is the ability to provision resources flexibly with just a few clicks or a single CLI command. However, this flexibility is also a double-edged sword.

When developing or testing applications (POC/Staging), engineers often quickly create EC2 Instances along with EBS volumes, Elastic IP addresses, Load Balancers, NAT Gateways, or backup Snapshots. Once testing is complete, we usually have the habit of clicking **Stop** or **Terminate** on the EC2 server.

**However, AWS's billing mechanism separates charges by individual service:**

1. **Compute:** Charges stop when the EC2 instance is in the `Stopped` state.
2. **Storage & Networking:** **Charges continue** as long as those resources still exist in your account, regardless of whether they are attached to an EC2 instance or not!

A common misconception is *"Stopping EC2 means stopping all costs."* In reality, orphaned resources left behind continue to exist like "ghosts", silently draining money from your credit card every day.

---

## Top 4 Most Common "Zombie" Culprits on AWS

### 1. Unattached EBS Volumes

When creating an EC2 Instance, AWS will provision **Amazon EBS** (Elastic Block Store) volumes along with it. If, when deleting the EC2 instance, you do not select the `Delete on Termination` option for additional volumes, these EBS volumes will transition to the `available` (unattached) state.

- **Waste level:** EBS is charged based on the provisioned storage capacity per GB/month, regardless of whether you actually write data to it.
- **Example:** A forgotten `gp3` volume with a capacity of 500 GB can consume approximately ~$40 - $50 USD/month without providing any value.

---

### 2. Unassociated Elastic IP Addresses

AWS provides public static IP addresses (**Elastic IP - EIP**) for free when you associate them with a running EC2 Instance. However, if you:

- Stop the EC2 Instance (`Stopped`).
- Or disassociate the IP from the EC2 instance without releasing it back to AWS's pool.

AWS will start charging approximately **$0.005 USD/hour** (~$3.60 USD/month) for each unused EIP. The reason is that AWS wants to encourage users to return IPv4 resources, which are becoming increasingly scarce worldwide.

---

### 3. Old & Orphaned Snapshots / AMIs

During maintenance or upgrades, we often create **EBS Snapshots** or **Amazon Machine Images (AMI)** as backups.

Over time, the application has been upgraded through many new versions, but dozens of Snapshot backups, hundreds of GB in size and months or years old, remain untouched. Since Snapshots are stored on S3 in the form of deltas, the monthly accumulated cost of these old backups can reach hundreds of dollars without a cleanup policy.

---

### 4. Idle Load Balancers & Unused NAT Gateways

- **NAT Gateway:** Each NAT Gateway created in a VPC costs approximately **$0.045 USD/hour** (~$32 USD/month) just for the base hourly availability charge, excluding data processing fees. If you set up a NAT Gateway for a Dev/Test environment and forget about it, it will consume at least $32 USD per month.
- **Application / Network Load Balancer (ALB/NLB):** A Load Balancer with no active target groups or no traffic can still incur minimum LCU (Load Balancer Capacity Units) charges.

---

## 4-Step Process to Hunt Down and Eliminate Zombies

### Step 1 – Manually Review Using AWS CLI

You can use the following simple AWS CLI commands to quickly list Zombie resources in the current Region:

**Find orphaned EBS volumes (`status = available`):**

```bash
aws ec2 describe-volumes     --filters Name=status,Values=available     --query "Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}"     --output table
```

**Find Elastic IPs that are not associated with any resource:**

```bash
aws ec2 describe-addresses     --query "Addresses[?AssociationId==null].{IP:PublicIp,AllocationId:AllocId}"     --output table
```

**Find unattached Elastic Network Interfaces (ENIs):**

```bash
aws ec2 describe-network-interfaces     --filters Name=status,Values=available     --query "NetworkInterfaces[*].{ID:NetworkInterfaceId,Vpc:VpcId,Subnet:SubnetId}"     --output table
```

---

### Step 2 – Leverage AWS Trusted Advisor & Cost Explorer

If you want an overview across the entire AWS Account:

1. **AWS Trusted Advisor:** Access the Trusted Advisor dashboard and check the **Cost Optimization** section. The system will automatically scan and evaluate:
   - *Unattached EBS Volumes*
   - *Unassociated Elastic IP Addresses*
   - *Idle Load Balancers*
   - *Low Utilization Amazon EC2 Instances*
2. **AWS Cost Anomaly Detection:** Enable cost anomaly alerts. If a newly created Zombie resource causes a sudden increase in daily costs, AWS will immediately send an alert via Email or Slack/Teams.

---

### Step 3 – Apply Tagging and Embrace IaC Culture

The root cause of Zombies is the lack of accountability and transparency regarding resources.

- **Establish a mandatory Tagging Strategy:** Every resource created must have standard Tags such as:
  - `Environment` (Dev / Staging / Prod)
  - `Owner` (email/name of the creator)
  - `ExpirationDate` (Expected destruction date for Test environments)
- **Implement Infrastructure as Code (IaC):** Use **Terraform** or **AWS CloudFormation** to manage infrastructure. When a test project needs to be terminated, simply run `terraform destroy` and all related resources (EC2, EBS, EIP, SG, NAT) will be cleanly removed, leaving no "Zombie traces" behind.

---

### Step 4 – Automate Cleanup with AWS Lambda & DLM

For large-scale systems, manual reviews are not feasible. Apply automation:

1. **Automated Snapshot Management:** Use **Amazon Data Lifecycle Manager (DLM)** to establish policies that automatically create and delete EBS Snapshots on a schedule (for example: keep snapshots for only 7 days and automatically delete them after 7 days).
2. **Auto-Cleanup Script with Lambda & EventBridge:** Write an **AWS Lambda** function (Python/Boto3) that runs weekly through **Amazon EventBridge**. This function will scan for EBS Volumes in the `available` state for more than 3 days or unused EIPs and send warning notifications via Telegram/Slack before automatically deleting them.

---

## Lessons Learned

Through managing and optimizing cloud costs across multiple projects, I have drawn the following important lessons:

1. **Stopping does not mean billing stops:** Always remember that Cloud services clearly separate Compute costs from Storage/Network costs.
2. **Clean up thoroughly when deleting servers:** When deleting EC2 from the Console, always check whether the attached EBS volumes have `Delete on termination` enabled, and manually delete unused Elastic IPs/Snapshots.
3. **FinOps is everyone's responsibility:** Cost optimization is not only the responsibility of the accounting or management department. Every developer and Cloud engineer needs to develop Cost Awareness from the moment they write code or design infrastructure.
4. **Automation is the key:** Humans can always forget, but automated scripts and Lifecycle policies do not.

---

## Conclusion

AWS costs are like a running water faucet: if you do not tightly control every small valve, the drops lost through **Zombie** resources will accumulate into a huge amount of money by the end of the month.

By understanding the list of common orphaned resources and combining regular reviews with automated cleanup processes, you can not only save **20% to 40% of your monthly AWS costs**, but also make your infrastructure much cleaner, safer, and easier to manage.

Open your **AWS Management Console** today, run the inspection commands above, and start "eliminating" your first Zombies!

---

## References

Read the full article on **[AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2237584380339855)**.

- AWS. **Optimizing AWS Cost with AWS Trusted Advisor**
  [https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor.html](https://docs.aws.amazon.com/awssupport/latest/user/trusted-advisor.html)

- AWS. **Amazon EC2 Pricing & Elastic IP Costs**
  [https://aws.amazon.com/ec2/pricing/](https://aws.amazon.com/ec2/pricing/)

- AWS. **Automating EBS Snapshot Lifecycle Management**
  [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snapshot-lifecycle.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snapshot-lifecycle.html)
