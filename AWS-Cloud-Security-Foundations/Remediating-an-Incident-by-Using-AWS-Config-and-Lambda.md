# Remediating an Incident by Using AWS Config and Lambda
---

## 1. Objectives

By the end of this lab, I was able to:

- Explain how to use IAM roles to grant AWS services access to other AWS services.
- Enable AWS Config to monitor resources in an AWS account.
- Create and enable a custom AWS Config rule that uses a pre-created Lambda function.
- Test the behavior of an AWS Config rule to ensure it's working as intended.
- Analyze CloudWatch logs to audit when AWS Config rules are invoked.

---

## 2. Architecture

### 2.1 Starting Architecture

- Two pre-provisioned IAM roles: `AwsConfigLambdaSGRole` and `AwsConfigRole`
- A pre-created Lambda function: `awsconfig_lambda_security_group`
- A default VPC with a default security group
- A custom VPC (`Lab VPC`) containing security group `LabSG1`

> <img width="3138" height="1670" alt="image" src="https://github.com/user-attachments/assets/eea7ad3d-6764-4651-8b38-d266836303df" />

### 2.2 Ending Architecture

- AWS Config enabled and tracking EC2 SecurityGroup resources
- A custom AWS Config rule (`EC2SecurityGroup`) wired to invoke the Lambda function on configuration changes
- Lambda function auto-remediating any non-compliant inbound rule changes
- CloudWatch Logs capturing the remediation actions

> <img width="3144" height="1680" alt="image" src="https://github.com/user-attachments/assets/2d969882-e81c-41d2-8a86-f445847c1425" />

**Remediation flow:**
1. The AWS Config rule monitors for changes to security groups tracked in the Config resource inventory.
2. When a change is detected, the rule invokes the Lambda function.
3. The function remediates the situation by restoring the desired inbound rule configuration.

---

## 4. Task 1: Examining and Updating IAM Roles

### 4.1 Reviewed AwsConfigLambdaSGRole

- **IAM console** → **Roles** → `AwsConfigLambdaSGRole` → **Permissions** tab → expanded `awsconfig_lambda_ec2_sg_role_policy`.
- Confirmed the policy grants:
  - `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` (for CloudWatch Logs)
  - `config:PutEvaluations`, `ec2:DescribeSecurityGroups`, `ec2:AuthorizeSecurityGroupIngress`, `ec2:RevokeSecurityGroupIngress` (for inspecting/modifying security group rules)

> <img width="1480" height="497" alt="image" src="https://github.com/user-attachments/assets/23dfda1b-bf75-4107-a210-9f6cc9d5dc31" />

**Analysis:** This role will be attached to the Lambda function later, defining exactly what the function is permitted to do — log its activity and add/remove security group ingress rules.

### 4.2 Updated AwsConfigRole

- **Roles** → `AwsConfigRole` → **Permissions** tab → expanded `S3Access` (existing policy: grants S3 ACL read + conditional object upload, used by AWS Config to deliver logs to S3).
- **Add permissions > Attach policies** → searched **Config** → selected **AWS_ConfigRole** → **Add permissions**.

> <img width="1473" height="469" alt="image" src="https://github.com/user-attachments/assets/fd1fc195-0c69-4a16-9428-c789aa3a4d9a" />

> <img width="1863" height="557" alt="image" src="https://github.com/user-attachments/assets/6740db55-8b75-4406-aa8e-3c7d560e7993" />

> <img width="1421" height="544" alt="image" src="https://github.com/user-attachments/assets/39fc5a31-8392-441b-b8ab-ce90c5ba7e1b" />

**Analysis:** `AWS_ConfigRole` grants broad read-level (`Get`/`List`/`Describe`) access across many services — this is what allows AWS Config to actually discover and evaluate the state of resources across the account.

---

## 5. Task 2: Setting Up AWS Config to Monitor Resources

- **AWS Config console** → **Get started**:
  - **Recording strategy:** Specific resource types
  - **Resource type:** AWS EC2 SecurityGroup
  - **Frequency:** Continuous
  - **IAM role for AWS Config:** Choose a role from your account → `AwsConfigRole`
  - **Delivery channel:** kept default (S3 bucket for findings)
- **AWS Managed Rules** page → **Next** (no managed rules added).
- Reviewed setup details → **Confirm**.

> <img width="1369" height="626" alt="image" src="https://github.com/user-attachments/assets/27662a33-0d8f-496d-bf53-1dbbe7e40e21" />

> <img width="1358" height="326" alt="image" src="https://github.com/user-attachments/assets/f39c5d77-f8ad-4296-aab4-e391278b3907" />

> <img width="1365" height="507" alt="image" src="https://github.com/user-attachments/assets/5f02f1ca-c5b5-4c77-9e3d-6f25d41984b6" />

### 5.1 Reviewed the Resource Inventory

- **Resources** (left nav) → reviewed the Resource Inventory listing EC2-related resources.

**Analysis:** Even though only `AWS EC2 SecurityGroup` was selected as the tracked resource type, AWS Config also surfaces **related** resources, since changes to those can affect the behavior of the primary tracked resources.

---

## 6. Task 3: Modifying a Security Group that AWS Config Monitors

*(This step intentionally simulates a security incident.)*

- **VPC console** → **Filter by VPC** → `Lab VPC` → **Security groups** → selected `LabSG1`.
- **Inbound rules** → **Edit inbound rules**:
  - Changed the existing **HTTP** rule's source to **Anywhere-IPv4**.
  - **Add rule:** HTTPS → Anywhere-IPv4
  - **Add rule:** SMTPS → Anywhere-IPv4
  - **Add rule:** IMAPS → Anywhere-IPv4
- **Save rules**.

> <img width="1809" height="586" alt="image" src="https://github.com/user-attachments/assets/e744a0b1-4c4f-48a6-a801-488581f5e5ad" />

> <img width="1487" height="638" alt="image" src="https://github.com/user-attachments/assets/eae131d6-82d1-4ca1-acc2-24c3b754243b" />

**Note:** SMTPS and IMAPS are **not** part of the desired configuration — this step deliberately creates a non-compliant state that the Config rule + Lambda function will later detect and revert.

---

## 7. Task 4: Creating an AWS Config Rule That Calls a Lambda Function

### 7.1 Retrieved the Lambda ARN

- Opened the **AWS Details** panel and copied the `LambdaFunctionARN` value.

> <img width="829" height="662" alt="image" src="https://github.com/user-attachments/assets/e6f48fe1-3eb9-41d2-94f2-feb02fdee21f" />

### 7.2 Created the Custom Config Rule

- **AWS Config console** → **Rules** → confirmed no rules existed yet → **Add rule**.
- **Select rule type:** Create custom Lambda rule.
- **Configure rule:**
  - **AWS Lambda function ARN:** pasted value from AWS Details
  - **Name:** `EC2SecurityGroup`
  - **Description:** `Restrict inbound ports to HTTP and HTTPS`
  - **Trigger type:** When configuration changes
  - **Scope of changes:** Resources
  - **Resource type:** AWS EC2 SecurityGroup
  - **Parameters:** Key = `debug`, Value = `true`
- **Next** → **Save**.

> <img width="1912" height="579" alt="image" src="https://github.com/user-attachments/assets/e7824bd7-322c-4295-a774-74ee0ecac005" />

> <img width="1297" height="529" alt="image" src="https://github.com/user-attachments/assets/ab31c417-c1bf-4156-a9e2-1556658e1942" />
> <img width="1359" height="620" alt="image" src="https://github.com/user-attachments/assets/a8185707-9e98-425e-a665-63257b18659d" />

> <img width="1359" height="324" alt="image" src="https://github.com/user-attachments/assets/e670a08b-c308-490b-af69-41fa7a780e38" />

> <img width="1881" height="622" alt="image" src="https://github.com/user-attachments/assets/878dcf2f-7416-4ae7-91c3-a25c0e0ccdfc" />

### 7.3 Observed Rule Evaluation

- Opened the **EC2SecurityGroup** rule → **Resources in scope** → set the **Noncompliant** filter to **All**.
- Watched **Last successful evaluation** change from "Not available" to a timestamp.
- Waited for the **Compliance** value to change from "No results available" to **Compliant** for each in-scope security group.
- Noted the **Annotation** column showing **"Permissions were modified."**

> <img width="1420" height="446" alt="image" src="https://github.com/user-attachments/assets/dd4bf70f-fdfe-4048-837c-e9bb28aa2fd7" />

> <img width="1382" height="351" alt="image" src="https://github.com/user-attachments/assets/16079b7f-14b2-4bd5-9c8d-958a406b1416" />

**Analysis:** Setting the trigger type to "When configuration changes," scoped to `AWS EC2 SecurityGroup`, means any modification to an in-scope security group automatically triggers a new compliance evaluation — which in turn invokes the Lambda function.

---

## 8. Task 5: Revisiting the Security Group Configuration

### 8.1 Re-examined LabSG1's Inbound Rules

- **VPC console** → `Lab VPC` → **Security groups** → `LabSG1` → **Inbound rules** tab.
- Observed that only **HTTP** and **HTTPS** rules remain — the SMTPS and IMAPS rules added in Task 3 are **gone**.
- Also noted the HTTP/HTTPS rules now apply to both IPv4 **and** IPv6 (rather than just the IPv4 sources originally configured).

> <img width="1461" height="637" alt="image" src="https://github.com/user-attachments/assets/fcef1577-4488-409f-8d2b-3687d22ecc2c" />

**Analysis:** The Lambda function automatically reverted the security group back to its desired (compliant) state — removing the unauthorized SMTPS/IMAPS rules added in Task 3 — confirming that the AWS Config rule successfully detected and remediated the simulated incident.

### 8.2 Reviewed the Lambda Function Code

- **Lambda console** → **Functions** → `awsconfig_lambda_security_group` → **Code source** → opened `awsconfig_lambda_security_group.py`.
- Noted key details:
  - Line 2: imports `boto3` (AWS SDK for Python).
  - Line 9: `REQUIRED_PERMISSIONS` array defines the desired ingress rules for in-scope security groups.
  - Line 117: calls `describe_security_groups()` to compare actual vs. required permissions.
  - Line 129: checks the `debug` parameter (set to `true` in Task 4) to control verbose logging output.

> <img width="1853" height="639" alt="image" src="https://github.com/user-attachments/assets/293b910c-1319-4bcb-8f81-dfcd1d43cfb7" />

> <img width="810" height="62" alt="image" src="https://github.com/user-attachments/assets/5c211941-c28e-49b6-87d8-6cd1ece14a2e" />

**Analysis:** The incident (manual modification) actually occurred *before* the Config rule and Lambda function existed, so the first compliance evaluation is what caught and fixed it. Going forward, **any future modification** to an in-scope security group (including the default security groups, which are also monitored) would trigger a new evaluation, re-invoke the Lambda function, and be automatically reverted to the desired state.

---

## 9. Task 6: Using CloudWatch Logs for Verification

- **CloudWatch console** → **Logs > Log groups** → opened `awsconfig_lambda_security_group` log group.
- Selected **Search all** across log streams.
- **Filter events** search field → entered `revoking` → pressed Enter.
- Expanded matching log events and reviewed their contents.

> <img width="1402" height="381" alt="image" src="https://github.com/user-attachments/assets/80245351-84ee-4074-a8dd-d0c8b422bf06" />

> <img width="1501" height="613" alt="image" src="https://github.com/user-attachments/assets/bcf9d7a0-ce7d-4cb2-80f7-90348635798f" />

> <img width="1434" height="319" alt="image" src="https://github.com/user-attachments/assets/aa521d48-7493-434f-87db-eefd1040c878" />

**Analysis:** The CloudWatch logs provide a clear audit trail confirming the Lambda function executed, identified the non-compliant inbound rules (SMTPS/IMAPS), and called `RevokeSecurityGroupIngress` to remove them — completing the automated remediation loop.

---

## 10. Summary of Key Concepts

| Concept | Key Takeaway |
|---|---|
| Service-to-service IAM roles | A role's trust policy lets a service (e.g., AWS Config, Lambda) assume it; its permissions policy defines what that service can then do. |
| `AwsConfigRole` vs. `AwsConfigLambdaSGRole` | One grants **AWS Config** broad read access to discover/evaluate resources; the other grants the **Lambda function** the specific write actions needed to remediate (revoke/authorize ingress rules). |
| AWS Config resource inventory | Tracks specified resource types plus related resources that could affect their behavior. |
| Custom Config rule (Lambda-backed) | Triggers Lambda evaluation logic on a configurable trigger (e.g., "When configuration changes") scoped to a resource type. |
| Auto-remediation pattern | Config detects drift → invokes Lambda → Lambda compares actual vs. desired state → Lambda calls EC2 APIs to revert non-compliant changes. |
| CloudWatch Logs as audit trail | Provides verifiable, searchable evidence of exactly what the remediation Lambda did and when. |

---
