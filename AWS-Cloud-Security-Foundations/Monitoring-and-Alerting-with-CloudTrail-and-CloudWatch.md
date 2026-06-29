# Lab 6.1: Monitoring and Alerting with CloudTrail and CloudWatch
---

## 1. Objectives

By the end of this lab, I was able to:

- Analyze event details in the CloudTrail event history.
- Create a CloudTrail trail with CloudWatch logging enabled.
- Create an SNS topic and an email subscription to it.
- Configure an EventBridge rule to monitor changes to resources in an AWS account.
- Create CloudWatch metric filters and CloudWatch alarms.
- Query CloudTrail logs by using CloudWatch Logs Insights.

---

## 2. Architecture

### 2.1 Starting Architecture

- An EC2 instance (`LabInstance`) associated with a security group (`LabSecurityGroup`)
- A preconfigured CloudTrail trail (`LabCloudTrail`) already writing to CloudWatch Logs
- A preconfigured IAM user (`test`) used later to simulate failed console logins

### 2.2 Architecture After Task 3

- Added: SNS topic (`MySNSTopic`) with an email subscription
- Added: EventBridge rule (`MonitorSecurityGroups`) that publishes to the SNS topic on security group changes

> <img width="951" height="424" alt="image" src="https://github.com/user-attachments/assets/06a106ba-05bb-4b5d-b019-402522dfeb94" />

### 2.3 Architecture After Task 5

- Added: CloudWatch metric filter (`ConsoleLoginErrors`) and alarm (`FailedLogins`) on the CloudTrail log group
- Added: CloudWatch Logs Insights queries against `CloudTrailLogGroup`

> <img width="978" height="434" alt="image" src="https://github.com/user-attachments/assets/3091a1e0-6283-4b41-a28e-ee40f120d02b" />

---

## 4. Task 1: Creating a CloudTrail Trail with CloudWatch Logs Enabled

### 4.1 Analyzing Existing Event History

- Opened the **CloudTrail console** → **Event history**.
- Changed the filter from **Read-only** to **Event source**, filtered on `cloudformation.amazonaws.com`.
- Opened the most recent **CreateStack** event and reviewed the record (userIdentity, eventTime, awsRegion, etc.).

> <img width="1864" height="629" alt="image" src="https://github.com/user-attachments/assets/6c8239d6-9494-400a-abbc-c8eca9ac3f18" />

> <img width="814" height="692" alt="image" src="https://github.com/user-attachments/assets/0c2d47de-4fa7-4822-89b0-6175a2afef9a" />

**Note:** Event history shows the last 90 days of **management events** by default per Region. A configured **trail** is needed to retain history longer and to optionally capture data events / read-only activity.

### 4.2 Attempting to Create a New Trail

- **Trails** → **Create trail**:
  - **Trail name:** `MyLabCloudTrail`
  - **Storage location:** Create a new S3 bucket (default name with `aws-cloudtrail-logs`)
  - **Log file SSE-KMS encryption:** disabled
  - **CloudWatch Logs:** Enabled, new log group (default name)
  - **IAM Role:** Existing → `LabCloudTrailRole`
- On the **Choose log events** page: kept **Management events**, **Read and Write**; did not select Data events or Insights events.
- At the final step, **chose Cancel** — the lab account's permissions intentionally prevent creating a new trail with CloudWatch Logs enabled.

> <img width="1480" height="642" alt="image" src="https://github.com/user-attachments/assets/76b718b5-b63e-45dd-a68e-57ce1eb8b4db" />

> <img width="1087" height="667" alt="image" src="https://github.com/user-attachments/assets/74adcdaa-f27e-44d1-94a8-3b126f26fc28" />

### 4.3 Analyzing the Existing Trail

- Reviewed the preexisting trail, **LabCloudTrail**, and confirmed it uses equivalent settings (CloudWatch Logs enabled) to what was just configured manually.

> <img width="1841" height="322" alt="image" src="https://github.com/user-attachments/assets/766c8ac9-64e5-45b4-a429-0e4dead19096" />

**Analysis:** This pre-existing trail (with CloudWatch Logs enabled) is the foundation for the EventBridge and CloudWatch alarm work in later tasks, since EventBridge's `AWS API Call via CloudTrail` event pattern relies on a trail with logging enabled.

---

## 5. Task 2: Creating an SNS Topic and Subscribing to It

### 5.1 Created the Topic

- **Amazon SNS console** → **Topics** → **Create topic**:
  - **Type:** Standard
  - **Name:** `MySNSTopic`
  - **Access policy:** Publish = Everyone, Subscribe = Everyone
- Chose **Create topic**.

> <img width="1583" height="454" alt="image" src="https://github.com/user-attachments/assets/6b0b22c1-2606-4f2c-9174-57ab12d0241f" />

> <img width="1531" height="433" alt="image" src="https://github.com/user-attachments/assets/d62ef6c0-0c9c-4d9b-a4d4-77d9464664c4" />

### 5.2 Created an Email Subscription

- **Create subscription**:
  - **Topic ARN:** pre-filled
  - **Protocol:** Email
  - **Endpoint:** _[your email]_
- Chose **Create subscription**.

> <img width="1525" height="517" alt="image" src="https://github.com/user-attachments/assets/39592fab-66dd-4332-9874-28b21761a370" />

### 5.3 Confirmed the Subscription

- Checked email inbox for the AWS Notifications confirmation message.
- Chose **Confirm subscription** link → confirmed successfully in browser.

> <img width="1410" height="522" alt="image" src="https://github.com/user-attachments/assets/b30efcc4-f819-40a0-b190-a9cd9bcd3215" />

> <img width="915" height="397" alt="image" src="https://github.com/user-attachments/assets/f36150f0-f1b0-46e0-ba1f-6449e237b69b" />

---

## 6. Task 3: Creating an EventBridge Rule to Monitor Security Groups

### 6.1 Defined the Rule

- **Amazon EventBridge console** → **Create rule**:
  - **Name:** `MonitorSecurityGroups`
  - **Event bus:** default
  - **Rule type:** Rule with an event pattern

### 6.2 Built the Event Pattern

- **Event source:** AWS events or EventBridge partner events
- Used **Custom patterns (JSON editor)** and entered:

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": ["AuthorizeSecurityGroupIngress", "ModifyNetworkInterfaceAttribute"]
  }
}
```

### 6.3 Configured the Target

- **Target 1:**
  - **Target types:** AWS service
  - **Select a target:** SNS topic
  - **Topic:** `MySNSTopic`
  - **Permissions:** unchecked "Use execution role (recommended)"
- **Additional settings > Configure target input:** Input transformer
  - **Input path:**
    ```json
    {"name":"$.detail.requestParameters.groupId","source":"$.detail.eventName","time":"$.time","value":"$.detail"}
    ```
  - **Template:**
    ```
    "The <source> API call was made against the <name> security group on <time> with the following details:"
    " <value> "
    ```
- Chose **Confirm**, then **Next** through tags and review, and **Create rule**.

> <img width="1520" height="634" alt="image" src="https://github.com/user-attachments/assets/977ef3be-7efc-47ed-899c-839fe923947f" />

> <img width="1485" height="293" alt="image" src="https://github.com/user-attachments/assets/31cc8d48-3828-4e7c-bda2-a8ba10c0b811" />

### 6.4 Tested the Rule

- **EC2 console** → **Instances** → selected `LabInstance` → **Security** tab → opened `LabSecurityGroup`.
- **Inbound rules** → **Edit inbound rules** → **Add rule**:
  - **Type:** SSH
  - **Source:** Anywhere-IPv4
- **Save rules**.

> <img width="1905" height="526" alt="image" src="https://github.com/user-attachments/assets/987c3019-8039-4647-a38a-7c6431448ef7" />

### 6.5 Verified in CloudTrail

- **CloudTrail console** → **Event history** → located the new **AuthorizeSecurityGroupIngress** event.
- Opened the event and confirmed `fromPort`/`toPort` = **22** (not 80), matching the SSH rule just added.

> <img width="1615" height="397" alt="image" src="https://github.com/user-attachments/assets/9003c509-07f6-4edb-88f2-875ccbee159e" />

> <img width="760" height="587" alt="image" src="https://github.com/user-attachments/assets/757ddc37-9fc9-444d-a5f7-0e28d02ba32c" />

### 6.6 Verified the Email Notification

- Checked the subscribed inbox for a message from AWS Notifications describing the `AuthorizeSecurityGroupIngress` call.

**Analysis:** The EventBridge rule matches on `AWS API Call via CloudTrail` events sourced from `ec2.amazonaws.com` with specific event names. Because the CloudTrail trail was already logging to CloudWatch, the matching API call triggered the rule, which used an **input transformer** to format a readable message and published it to the SNS topic — resulting in the email alert.

---

## 7. Task 4: Creating a CloudWatch Alarm Based on a Metrics Filter

### 7.1 Created a Metric Filter

- **CloudWatch console** → **Logs > Log groups** → selected `CloudTrailLogGroup`.
- **Actions > Create metric filter**:
  - **Filter pattern:**
    ```
    { ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") }
    ```
  - **Filter name:** `ConsoleLoginErrors`
  - **Metric namespace:** `CloudTrailMetrics`
  - **Metric name:** `ConsoleLoginFailureCount`
  - **Metric value:** `1`
- Chose **Create metric filter**.

> <img width="1568" height="281" alt="image" src="https://github.com/user-attachments/assets/22f285b0-615d-45fb-81b3-fbe101f5a478" />

> <img width="822" height="734" alt="image" src="https://github.com/user-attachments/assets/cb9a2aaf-d389-482b-8cb7-eb81d40686e2" />

> <img width="1563" height="646" alt="image" src="https://github.com/user-attachments/assets/5fced64a-65bf-42ee-8132-75a447024572" />

### 7.2 Created the Alarm

- Selected `ConsoleLoginErrors` → **Create alarm**:
  - **Condition:** `ConsoleLoginFailureCount` **Greater/Equal** than **3** (within a 5-minute period)
  - **SNS topic:** Select an existing topic → `MySNSTopic`
  - **Alarm name:** `FailedLogins`
- Chose **Create alarm**.

> <img width="1476" height="440" alt="image" src="https://github.com/user-attachments/assets/f66b15d0-760d-4f41-bfb6-bd51a8ae3a75" />

> <img width="1510" height="670" alt="image" src="https://github.com/user-attachments/assets/1c772636-849c-4b6c-b0df-6c0fd882ad6c" />

> <img width="1513" height="555" alt="image" src="https://github.com/user-attachments/assets/3fa16bb6-dc3a-431f-9477-de1a1fa54d6f" />

> <img width="1580" height="401" alt="image" src="https://github.com/user-attachments/assets/6fbbc7ff-de4f-4a5b-9bf5-3af19250b521" />

### 7.3 Triggered the Alarm

- **IAM console** → **Users** → opened the `test` user → **Security credentials** tab → copied the console sign-in link.
- Opened the sign-in link in a new tab and attempted to log in **at least 3 times** with:
  - **IAM user name:** `test`
  - **Password:** `test` (intentionally incorrect)

> <img width="523" height="761" alt="image" src="https://github.com/user-attachments/assets/8265a33d-27f4-4b8a-afc6-ac4d5c5eff40" />

> <img width="532" height="839" alt="image" src="https://github.com/user-attachments/assets/7136f4be-921a-473d-af0b-37e763eb32f5" />

### 7.4 Re-authenticated as voclabs

- Closed all console tabs and reopened the console via the **AWS** link to restore the `voclabs` session.

### 7.5 Graphed the Metric

- **CloudWatch console** → **Metrics > All metrics** → **Custom namespaces > CloudTrailMetrics** → **Metrics with no dimensions** → `ConsoleLoginFailureCount`.
- Observed a data point on the graph corresponding to the failed login attempts.

> <img width="1126" height="447" alt="image" src="https://github.com/user-attachments/assets/6feb8908-c5eb-4127-9e54-2e7d16603512" />

### 7.6 Checked the Alarm State

- **Alarms > All alarms** → confirmed `FailedLogins` shows state **In alarm**.
- Opened the alarm → **History** tab to confirm it was recently invoked.

> <img width="1920" height="390" alt="image" src="https://github.com/user-attachments/assets/ab5969f8-72f9-44e5-893d-e508f15eb36c" />

> <img width="1189" height="287" alt="image" src="https://github.com/user-attachments/assets/960275ee-ce69-456b-8243-77de0c04a55c" />

### 7.7 Verified the Email Notification

- Checked the subscribed inbox for a message about multiple failed login attempts.

> <img width="1386" height="746" alt="image" src="https://github.com/user-attachments/assets/73085777-b624-4760-be2e-d229fd601deb" />

**Analysis:** This pipeline differs from Task 3's: instead of EventBridge directly matching individual API events, a **CloudWatch metric filter** scans the CloudTrail log group for a specific pattern (`ConsoleLogin` events with `errorMessage = "Failed authentication"`), increments a custom metric for each match, and a **CloudWatch alarm** evaluates that metric against a threshold (≥3 within 5 minutes) before notifying the SNS topic.

---

## 8. Task 5: Querying CloudTrail Logs by Using CloudWatch Logs Insights

- **CloudWatch console** → **Logs Insights**.
- **Selection criteria:** `CloudTrailLogGroup`.
- Replaced the default query with:

```
filter eventSource="signin.amazonaws.com" and eventName="ConsoleLogin" and responseElements.ConsoleLogin="Failure"
| stats count(*) as Total_Count by sourceIPAddress as Source_IP, errorMessage as Reason, awsRegion as AWS_Region, userIdentity.arn as IAM_Arn
```

- Chose **Run query**.

> <img width="1888" height="233" alt="image" src="https://github.com/user-attachments/assets/550f8ea0-97f0-4835-962b-fb3f004e75bd" />

> <img width="1895" height="599" alt="image" src="https://github.com/user-attachments/assets/b2c7142b-d206-4f21-a763-90f7b1ae6b4d" />

**Analysis:** This query aggregates all failed console sign-in attempts captured in the CloudTrail logs (including the ones generated against the `test` user in Task 4) and groups them by source IP, failure reason, Region, and the IAM identity attempted — a pattern useful for investigating brute-force login attempts or unauthorized access attempts.

---

## 9. Summary of Key Concepts

| Concept | Key Takeaway |
|---|---|
| CloudTrail Event history vs. Trail | Event history = 90-day default view of management events per Region; a Trail provides configurable, longer-term, optionally cross-Region logging (e.g., to S3/CloudWatch Logs). |
| EventBridge rule (event pattern) | Reacts in near real-time to individual matching CloudTrail-sourced events (e.g., a single `AuthorizeSecurityGroupIngress` call) and can route them directly to a target like SNS. |
| Input transformer | Reshapes the raw event JSON into a custom, human-readable message before it's delivered to the target. |
| CloudWatch metric filter | Scans log data for a pattern and increments a custom metric each time it matches — useful for **counting/aggregating** occurrences over time rather than reacting to a single event. |
| CloudWatch alarm | Evaluates a metric against a threshold over a time window and triggers actions (e.g., SNS notification) when the condition is met. |
| CloudWatch Logs Insights | Enables ad hoc, SQL-like querying and aggregation directly against log data for investigation, without needing a pre-defined metric filter. |

---
