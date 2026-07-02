# AWS Cloud Security Foundations

A collection of hands-on guides covering core AWS cloud security practices — encryption, monitoring, access control, network security, and incident remediation. Each guide walks through a specific security control, why it matters, and how to implement it using native AWS services.

## 📚 Contents

| Guide | Description |
|---|---|
| [Securing Access to Cloud Resources](./Securing-Access-To-Cloud-Ressources.md) | Foundational IAM practices — least-privilege policies, roles vs. users, MFA, and access reviews for controlling who can do what in your AWS account. |
| [Securing VPC Resources by Using Security Groups](./Securing-VPC-Resources-by-Using-Security-Groups.md) | Network-level security using Security Groups and NACLs to control inbound/outbound traffic to EC2 instances and other VPC resources. |
| [Encrypting Data at Rest by Using AWS KMS](./Encrypting-Data-at-Rest-by-Using-AWS-KMS.md) | Protecting stored data with AWS Key Management Service — creating and managing customer-managed keys, and enabling encryption for S3, EBS, and RDS. |
| [Monitoring and Alerting with CloudTrail and CloudWatch](./Monitoring-and-Alerting-with-CloudTrail-and-CloudWatch.md) | Setting up account activity logging with CloudTrail and configuring CloudWatch alarms/metrics to detect suspicious or unauthorized activity. |
| [Remediating an Incident by Using AWS Config and Lambda](./Remediating-an-Incident-by-Using-AWS-Config-and-Lambda.md) | Automated incident response — using AWS Config rules to detect non-compliant resources and Lambda functions to automatically remediate them. |

## 🎯 Purpose

This repository serves as a practical reference for implementing a defense-in-depth security posture on AWS, touching the key pillars of cloud security:

- **Identity & Access Management** — who can access what
- **Network Security** — how traffic is controlled and segmented
- **Data Protection** — how data is encrypted at rest
- **Detection & Monitoring** — how activity is logged and alerted on
- **Response & Remediation** — how issues are automatically fixed

## 🚀 Getting Started

Each `.md` file is self-contained and can be followed independently, though it's recommended to start with **Securing Access to Cloud Resources** since IAM underpins every other guide in this repository.

1. Clone this repository
2. Open the guide relevant to the control you want to implement
3. Follow the step-by-step instructions in your own AWS environment (a sandbox or non-production account is recommended)

## 📄 License

Feel free to use and adapt these guides for learning or internal documentation purposes.
