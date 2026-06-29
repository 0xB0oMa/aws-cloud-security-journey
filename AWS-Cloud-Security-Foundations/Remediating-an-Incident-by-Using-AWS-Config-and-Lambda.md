# Lab 7.1: Remediating an Incident by Using AWS Config and Lambda
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

> 📸 **Screenshot 1:** Starting architecture diagram (from the lab overview page).

### 2.2 Ending Architecture

- AWS Config enabled and tracking EC2 SecurityGroup resources
- A custom AWS Config rule (`EC2SecurityGroup`) wired to invoke the Lambda function on configuration changes
- Lambda function auto-remediating any non-compliant inbound rule changes
- CloudWatch Logs capturing the remediation actions

> 📸 **Screenshot 2:** Ending architecture diagram showing the Config → Lambda → Security Group remediation flow.

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

> 📸 **Screenshot 4:** Expanded `awsconfig_lambda_ec2_sg_role_policy` JSON for AwsConfigLambdaSGRole.

**Analysis:** This role will be attached to the Lambda function later, defining exactly what the function is permitted to do — log its activity and add/remove security group ingress rules.

### 4.2 Updated AwsConfigRole

- **Roles** → `AwsConfigRole` → **Permissions** tab → expanded `S3Access` (existing policy: grants S3 ACL read + conditional object upload, used by AWS Config to deliver logs to S3).
- **Add permissions > Attach policies** → searched **Config** → selected **AWS_ConfigRole** → **Add permissions**.

> 📸 **Screenshot 5:** Existing S3Access policy expanded for AwsConfigRole.

> 📸 **Screenshot 6:** Attach policies search results showing AWS_ConfigRole selected.

> 📸 **Screenshot 7:** AwsConfigRole permissions list after AWS_ConfigRole was successfully attached.

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

> 📸 **Screenshot 8:** Recording strategy / resource type selection screen.

> 📸 **Screenshot 9:** IAM role selection screen showing AwsConfigRole chosen.

> 📸 **Screenshot 10:** Delivery channel settings (default S3 bucket).

> 📸 **Screenshot 11:** Final review screen before choosing Confirm.

> 📸 **Screenshot 12:** AWS Config Dashboard after setup completes.

### 5.1 Reviewed the Resource Inventory

- **Resources** (left nav) → reviewed the Resource Inventory listing EC2-related resources.

> 📸 **Screenshot 13:** Resource Inventory page listing discovered EC2 security groups and related resources (e.g., internet gateways, network ACLs).

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

> 📸 **Screenshot 14:** Edit inbound rules dialog showing all four rules (HTTP, HTTPS, SMTPS, IMAPS) configured before saving.

> 📸 **Screenshot 15:** LabSG1 inbound rules tab after saving, showing all four rules now present.

**Note:** SMTPS and IMAPS are **not** part of the desired configuration — this step deliberately creates a non-compliant state that the Config rule + Lambda function will later detect and revert.

---

## 7. Task 4: Creating an AWS Config Rule That Calls a Lambda Function

### 7.1 Retrieved the Lambda ARN

- Opened the **AWS Details** panel and copied the `LambdaFunctionARN` value.

> 📸 **Screenshot 16:** AWS Details panel showing the LambdaFunctionARN value.

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

> 📸 **Screenshot 17:** Rules list showing no rules defined before creating one.

> 📸 **Screenshot 18:** "Configure rule" page with the Lambda ARN, name, description, and trigger type filled in.

> 📸 **Screenshot 19:** Parameters section showing debug = true added.

> 📸 **Screenshot 20:** Rules list showing `EC2SecurityGroup` created after saving.

### 7.3 Observed Rule Evaluation

- Opened the **EC2SecurityGroup** rule → **Resources in scope** → set the **Noncompliant** filter to **All**.
- Watched **Last successful evaluation** change from "Not available" to a timestamp.
- Waited for the **Compliance** value to change from "No results available" to **Compliant** for each in-scope security group.
- Noted the **Annotation** column showing **"Permissions were modified."**

> 📸 **Screenshot 21:** Rule details page showing "Last successful evaluation" with a timestamp.

> 📸 **Screenshot 22:** Resources in scope list showing security groups now marked Compliant, with the "Permissions were modified" annotation visible.

**Analysis:** Setting the trigger type to "When configuration changes," scoped to `AWS EC2 SecurityGroup`, means any modification to an in-scope security group automatically triggers a new compliance evaluation — which in turn invokes the Lambda function.

---

## 8. Task 5: Revisiting the Security Group Configuration

### 8.1 Re-examined LabSG1's Inbound Rules

- **VPC console** → `Lab VPC` → **Security groups** → `LabSG1` → **Inbound rules** tab.
- Observed that only **HTTP** and **HTTPS** rules remain — the SMTPS and IMAPS rules added in Task 3 are **gone**.
- Also noted the HTTP/HTTPS rules now apply to both IPv4 **and** IPv6 (rather than just the IPv4 sources originally configured).

> 📸 **Screenshot 23:** LabSG1 inbound rules tab now showing only HTTP and HTTPS rules (SMTPS/IMAPS removed).

**Analysis:** The Lambda function automatically reverted the security group back to its desired (compliant) state — removing the unauthorized SMTPS/IMAPS rules added in Task 3 — confirming that the AWS Config rule successfully detected and remediated the simulated incident.

### 8.2 Reviewed the Lambda Function Code

- **Lambda console** → **Functions** → `awsconfig_lambda_security_group` → **Code source** → opened `awsconfig_lambda_security_group.py`.
- Noted key details:
  - Line 2: imports `boto3` (AWS SDK for Python).
  - Line 9: `REQUIRED_PERMISSIONS` array defines the desired ingress rules for in-scope security groups.
  - Line 117: calls `describe_security_groups()` to compare actual vs. required permissions.
  - Line 129: checks the `debug` parameter (set to `true` in Task 4) to control verbose logging output.

> 📸 **Screenshot 24:** Lambda console Code source view showing the imports and REQUIRED_PERMISSIONS definition (lines 2 and 9 area).

> 📸 **Screenshot 25:** Code view showing the describe_security_groups() call and the debug parameter check (lines ~117 and ~129).

**Analysis:** The incident (manual modification) actually occurred *before* the Config rule and Lambda function existed, so the first compliance evaluation is what caught and fixed it. Going forward, **any future modification** to an in-scope security group (including the default security groups, which are also monitored) would trigger a new evaluation, re-invoke the Lambda function, and be automatically reverted to the desired state.

---

## 9. Task 6: Using CloudWatch Logs for Verification

- **CloudWatch console** → **Logs > Log groups** → opened `awsconfig_lambda_security_group` log group.
- Selected **Search all** across log streams.
- **Filter events** search field → entered `revoking` → pressed Enter.
- Expanded matching log events and reviewed their contents.

> 📸 **Screenshot 26:** Log group view showing multiple log streams.

> 📸 **Screenshot 27:** Filtered log events search for "revoking."

> 📸 **Screenshot 28:** Expanded log event showing the SMTPS (port 465) and IMAPS (port 993) rules being revoked from LabSG1.

> 📸 **Screenshot 29 (optional):** Additional filtered log events showing the Lambda function evaluating/remediating the other security groups in scope (e.g., the default security group).

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
