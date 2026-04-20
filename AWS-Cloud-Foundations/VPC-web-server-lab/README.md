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

