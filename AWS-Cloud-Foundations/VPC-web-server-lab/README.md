# Lab 2 – Build Your VPC and Launch a Web Server

## 1. Overview

In this lab I built a custom Amazon Virtual Private Cloud (VPC) from scratch, replacing the default VPC that AWS provides. I created public and private subnets across two Availability Zones, configured route tables with an Internet Gateway and a NAT Gateway, and set up a security group to control traffic. Finally, I launched an EC2 instance into a public subnet with a User Data script that automatically installed Apache and PHP, turning the instance into a functioning web server.

---

## 2. Objectives

After completing this lab, I was able to:

- Create a custom VPC with public and private subnets
- Create subnets in multiple Availability Zones for High Availability
- Configure a security group (virtual firewall)
- Launch an EC2 instance into a VPC
- Automate web server deployment using User Data

---

## 3. Architecture & Scenario

![lab environment](images/architecture.png)

### Task 1: Create Your VPC

I used the **VPC and more** option in the VPC console to create multiple resources at once, including a VPC, an Internet Gateway, a public subnet and a private subnet in a single Availability Zone, two route tables, and a NAT Gateway.

**Configuration used:**

| Parameter | Value |
|-----------|-------|
| Name tag auto-generation | Auto-generate (changed "project" to "lab") |
| IPv4 CIDR block | 10.0.0.0/16 |
| Number of Availability Zones | 1 |
| Number of public subnets | 1 |
| Number of private subnets | 1 |
| Public subnet CIDR block | 10.0.0.0/24 |
| Private subnet CIDR block | 10.0.1.0/24 |
| NAT gateways | In 1 AZ |
| VPC endpoints | None |
| DNS hostnames | Enabled |

**Resources created by the wizard:**

- VPC: lab-vpc
- Public subnet: lab-subnet-public1-us-east-1a
- Private subnet: lab-subnet-private1-us-east-1a
- Public route table: lab-rtb-public
- Private route table: lab-rtb-private1-us-east-1a
- Internet Gateway: lab-igw
- NAT Gateway: lab-nat-public1-us-east-1a

> **[SCREENSHOT HERE – Insert the Task 1 result diagram]**

The NAT Gateway took a few minutes to activate. I waited until all resources were created before proceeding.

---

