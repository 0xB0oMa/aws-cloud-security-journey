# Lab 3: Introduction to Amazon EC2

## Overview

In this lab, I launched, resized, managed, and monitored an Amazon EC2 instance. I deployed a web server with termination protection enabled, monitored the instance using Amazon CloudWatch metrics and system logs, modified the security group to allow HTTP access, resized the instance to scale resources, explored EC2 service quotas, and tested stop protection.

Amazon Elastic Compute Cloud (Amazon EC2) is a web service that provides resizable compute capacity in the cloud. It is designed to make web-scale cloud computing easier for developers. Amazon EC2 provides complete control of computing resources and reduces the time required to obtain and boot new server instances to minutes, allowing rapid scaling based on application requirements.

After completing this lab, I was able to:

- Launch a web server with termination protection enabled
- Monitor an EC2 instance
- Modify a security group to allow HTTP access
- Resize an EC2 instance and enable stop protection
- Explore EC2 service quotas
- Test stop protection
- Stop an EC2 instance

---

## Architecture Diagram

![Architecture Diagram](images/architecture-diagram.png)

---

# Task 1: Launch an Amazon EC2 Instance

In the AWS Management Console, I navigated to:

```text
Services → Compute → EC2
```

I verified that the selected region was:

```text
N. Virginia (us-east-1)
```

From the **Launch instance** menu, I selected **Launch instance**.

---

## Step 1: Name and Tags

| Configuration Setting | Value |
|---|---|
| Name | Web Server |

---

## Step 2: Application and OS Images (AMI)

| Configuration Setting | Value |
|---|---|
| AMI | Amazon Linux 2023 |

---

## Step 3: Instance Type

| Configuration Setting | Value |
|---|---|
| Instance Type | t2.micro |

---

## Step 4: Key Pair

| Configuration Setting | Value |
|---|---|
| Key Pair Name | vockey |

---

## Step 5: Network Settings

| Configuration Setting | Value |
|---|---|
| VPC | Lab VPC |
| Subnet | PublicSubnet1 |
| Auto-assign Public IP | Enable |
| Firewall (Security Groups) | Create new security group |
| Security Group Name | Web Server security group |
| Description | Security group for my web server |

I removed the default inbound rule.

---

## Step 6: Configure Storage

| Configuration Setting | Value |
|---|---|
| Root Volume Size | 8 GiB |

---

## Step 7: Advanced Details

| Configuration Setting | Value |
|---|---|
| Termination Protection | Enable |

I pasted the following script into the **User data** section:

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable httpd
systemctl start httpd
echo '<html><h1>Hello From Your Web Server!</h1></html>' > /var/www/html/index.html
```

---

## Step 8: Launch the Instance

At the bottom of the Summary panel, I selected **Launch instance**.

After the instance launched successfully, I selected **View all instances**.

I waited until the instance displayed:

| Status | Value |
|---|---|
| Instance State | Running |
| Status Checks | 2/2 checks passed |

![EC2 Instance Running](images/ec2-instance-running.png)

---

# Task 2: Monitor the EC2 Instance

I selected the `Web Server` instance and explored its monitoring features.

---

## Status Checks

I selected the **Status checks** tab and verified that both checks passed.

| Check | Status |
|---|---|
| System Reachability | Passed |
| Instance Reachability | Passed |

![Status Checks](images/status-checks.png)

---

## Monitoring Tab

I selected the **Monitoring** tab to view Amazon CloudWatch metrics for the EC2 instance.

![CloudWatch Metrics](images/cloudwatch-metrics.png)

---

## Get System Log

From the **Actions** menu, I selected:

```text
Monitor and troubleshoot → Get system log
```

I reviewed the console output and confirmed that the Apache HTTP package was installed from the user data script.

![System Log](images/system-log.png)

---

## Get Instance Screenshot

From the **Actions** menu, I selected:

```text
Monitor and troubleshoot → Get instance screenshot
```

![Instance Screenshot](images/instance-screenshot.png)

---

# Task 3: Update the Security Group and Access the Web Server

---

## Test Web Server Access Before Updating Security Group

I copied the **Public IPv4 address** from the instance details page and opened it in a web browser.

### Result

The web server was not accessible because the security group did not allow inbound HTTP traffic on port 80.

![Web Server Inaccessible](images/web-server-inaccessible.png)

---

## Update the Security Group

In the left navigation pane, I selected:

```text
Security Groups
```

I selected `Web Server security group` and edited the inbound rules.

I added the following rule:

| Type | Source |
|---|---|
| HTTP | Anywhere-IPv4 (0.0.0.0/0) |

I saved the rule changes.

![Security Group Rule](images/security-group-rule.png)

---

## Test Web Server Access After Updating Security Group

I refreshed the browser page.

### Result

The web server displayed the following message:

```text
Hello From Your Web Server!
```

![Web Server Accessible](images/web-server-accessible.png)

---

# Task 4: Resize the EC2 Instance

---

## Stop the Instance

From the **Instance state** menu, I selected:

```text
Stop instance
```

I waited until the instance state changed to:

```text
Stopped
```

![Instance Stopped](images/instance-stopped.png)

---

## Change the Instance Type

From the **Actions** menu, I selected:

```text
Instance settings → Change instance type
```

I changed the instance type to:

| Configuration Setting | Value |
|---|---|
| Instance Type | t2.small |

I selected **Apply**.

---

## Enable Stop Protection

From the **Actions** menu, I selected:

```text
Instance settings → Change stop protection
```

I enabled stop protection and saved the configuration.

![Change Instance Type](images/change-instance-type.png)

---

## Resize the EBS Volume

From the **Storage** tab, I selected the attached volume.

From the **Actions** menu, I selected:

```text
Modify volume
```

I changed the volume size to:

| Configuration Setting | Value |
|---|---|
| Size | 10 GiB |

I confirmed the modification.

![Modify Volume](images/modify-volume.png)

---

## Start the Resized Instance

From the **Instance state** menu, I selected:

```text
Start instance
```

![Instance Running After Resize](images/instance-running-after-resize.png)

---

# Task 5: Explore EC2 Service Quotas

In the AWS Management Console, I searched for and opened:

```text
Service Quotas
```

I selected:

```text
AWS services → Amazon Elastic Compute Cloud (Amazon EC2)
```

In the **Find quotas** search bar, I searched for:

```text
running on-demand
```

I reviewed the available EC2 service quotas.

![EC2 Service Quotas](images/ec2-service-quotas.png)

---

# Task 6: Test Stop Protection

I returned to the EC2 console and selected the `Web Server` instance.

From the **Instance state** menu, I selected:

```text
Stop instance
```

### Result

AWS displayed an error message indicating that the instance could not be stopped because stop protection was enabled.

![Stop Protection Error](images/stop-protection-error.png)

---

## Disable Stop Protection

From the **Actions** menu, I selected:

```text
Instance settings → Change stop protection
```

I disabled stop protection and saved the configuration.

---

## Stop the Instance

I selected:

```text
Instance state → Stop instance
```

The instance stopped successfully.

![Instance Stopped Successfully](images/instance-stopped-successfully.png)

---

# Complete Architecture

![Complete Architecture](images/complete-architecture-diagram.png)

| Component | Configuration |
|---|---|
| EC2 Instance Name | Web Server |
| AMI | Amazon Linux 2023 |
| Instance Type | t2.micro → t2.small |
| Key Pair | vockey |
| VPC | Lab VPC |
| Subnet | PublicSubnet1 |
| Security Group | HTTP from 0.0.0.0/0 |
| Root Volume | 8 GiB → 10 GiB |
| Termination Protection | Enabled |
| Stop Protection | Enabled then disabled |
| User Data | Apache installation and web page creation |

---

# Lessons Learned

| Concept | Description |
|---|---|
| Amazon EC2 | Provides scalable compute capacity in the cloud |
| AMI | Template used to launch EC2 instances |
| Instance Type | Defines CPU, RAM, storage, and networking capacity |
| Key Pair | Used for secure authentication |
| Security Group | Virtual firewall controlling inbound and outbound traffic |
| User Data | Script automatically executed during first boot |
| Termination Protection | Prevents accidental instance termination |
| Stop Protection | Prevents accidental instance shutdown |
| Status Checks | Automated health checks for EC2 instances |
| CloudWatch Metrics | Monitoring metrics collected from EC2 instances |
| System Log | Console output useful for troubleshooting |
| Instance Screenshot | Captures the EC2 console display |
| Service Quotas | AWS resource usage limits per region |
| EBS Volume | Persistent block storage attached to EC2 instances |

---

# Repository Structure

```text
.
├── README.md
└── images/
    ├── architecture-diagram.png
    ├── ec2-instance-running.png
    ├── status-checks.png
    ├── cloudwatch-metrics.png
    ├── system-log.png
    ├── instance-screenshot.png
    ├── web-server-inaccessible.png
    ├── security-group-rule.png
    ├── web-server-accessible.png
    ├── instance-stopped.png
    ├── change-instance-type.png
    ├── modify-volume.png
    ├── instance-running-after-resize.png
    ├── ec2-service-quotas.png
    ├── stop-protection-error.png
    ├── instance-stopped-successfully.png
    └── complete-architecture-diagram.png
```

---

# References

- [Amazon EC2 Documentation](https://docs.aws.amazon.com/ec2/?utm_source=chatgpt.com)
- [Launch an EC2 Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/LaunchingAndUsingInstances.html?utm_source=chatgpt.com)
- [Amazon EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/?utm_source=chatgpt.com)
- [Amazon Machine Images (AMI)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html?utm_source=chatgpt.com)
- [EC2 User Data Scripts](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com)
- [Security Groups for EC2 Instances](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html?utm_source=chatgpt.com)
- [EC2 Status Checks](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html?utm_source=chatgpt.com)
- [Resize an EC2 Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-resize.html?utm_source=chatgpt.com)
- [Start and Stop EC2 Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Stop_Start.html?utm_source=chatgpt.com)
- [Amazon EC2 Service Quotas](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html?utm_source=chatgpt.com)
- [Termination Protection](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html?utm_source=chatgpt.com#termination-protection)

---

*End of Lab 3 Writeup* 🚀
