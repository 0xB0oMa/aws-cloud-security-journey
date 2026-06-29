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

> 📸 **Screenshot 5:** Event history filtered by event source = cloudformation.amazonaws.com.

> 📸 **Screenshot 6:** CreateStack event record showing userIdentity, eventTime, and awsRegion.

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

> 📸 **Screenshot 7:** "Choose trail attributes" page filled in with MyLabCloudTrail settings.

> 📸 **Screenshot 8:** "Choose log events" page showing Management events / Read and Write selected.

> 📸 **Screenshot 9:** Final review page before choosing Cancel (demonstrating the configuration without completing creation).

### 4.3 Analyzing the Existing Trail

- Reviewed the preexisting trail, **LabCloudTrail**, and confirmed it uses equivalent settings (CloudWatch Logs enabled) to what was just configured manually.

> 📸 **Screenshot 10:** LabCloudTrail trail details page showing CloudWatch Logs integration.

**Analysis:** This pre-existing trail (with CloudWatch Logs enabled) is the foundation for the EventBridge and CloudWatch alarm work in later tasks, since EventBridge's `AWS API Call via CloudTrail` event pattern relies on a trail with logging enabled.

---

## 5. Task 2: Creating an SNS Topic and Subscribing to It

### 5.1 Created the Topic

- **Amazon SNS console** → **Topics** → **Create topic**:
  - **Type:** Standard
  - **Name:** `MySNSTopic`
  - **Access policy:** Publish = Everyone, Subscribe = Everyone
- Chose **Create topic**.

> 📸 **Screenshot 11:** Create topic form with name and access policy settings configured.

> 📸 **Screenshot 12:** MySNSTopic details page after creation, showing its ARN.

### 5.2 Created an Email Subscription

- **Create subscription**:
  - **Topic ARN:** pre-filled
  - **Protocol:** Email
  - **Endpoint:** _[your email]_
- Chose **Create subscription**.

> 📸 **Screenshot 13:** Create subscription form with Email protocol and endpoint entered.

### 5.3 Confirmed the Subscription

- Checked email inbox for the AWS Notifications confirmation message.
- Chose **Confirm subscription** link → confirmed successfully in browser.

> 📸 **Screenshot 14:** Confirmation email from AWS Notifications.

> 📸 **Screenshot 15:** Browser page confirming the subscription was successful.

---

## 6. Task 3: Creating an EventBridge Rule to Monitor Security Groups

### 6.1 Defined the Rule

- **Amazon EventBridge console** → **Create rule**:
  - **Name:** `MonitorSecurityGroups`
  - **Event bus:** default
  - **Rule type:** Rule with an event pattern

> 📸 **Screenshot 16:** "Define rule detail" screen with name, event bus, and rule type configured.

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

> 📸 **Screenshot 17:** Event pattern JSON editor with the custom pattern entered.

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

> 📸 **Screenshot 18:** Select targets section with SNS topic chosen and execution role unchecked.

> 📸 **Screenshot 19:** Input transformer configuration with the Input path and Template entered.

> 📸 **Screenshot 20:** Final "Review and create" page before choosing Create rule.

> 📸 **Screenshot 21:** EventBridge rules list showing `MonitorSecurityGroups` created.

### 6.4 Tested the Rule

- **EC2 console** → **Instances** → selected `LabInstance` → **Security** tab → opened `LabSecurityGroup`.
- **Inbound rules** → **Edit inbound rules** → **Add rule**:
  - **Type:** SSH
  - **Source:** Anywhere-IPv4
- **Save rules**.

> 📸 **Screenshot 22:** Edit inbound rules dialog with the new SSH rule added before saving.

### 6.5 Verified in CloudTrail

- **CloudTrail console** → **Event history** → located the new **AuthorizeSecurityGroupIngress** event.
- Opened the event and confirmed `fromPort`/`toPort` = **22** (not 80), matching the SSH rule just added.

> 📸 **Screenshot 23:** CloudTrail Event history showing the new AuthorizeSecurityGroupIngress entry.

> 📸 **Screenshot 24:** Event record showing fromPort/toPort = 22.

### 6.6 Verified the Email Notification

- Checked the subscribed inbox for a message from AWS Notifications describing the `AuthorizeSecurityGroupIngress` call.

> 📸 **Screenshot 25:** Email received describing the AuthorizeSecurityGroupIngress event, matching the input transformer template.

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

> 📸 **Screenshot 26:** Filter pattern entry screen.

> 📸 **Screenshot 27:** Metric details screen (namespace, name, value) before creating the filter.

> 📸 **Screenshot 28:** Metric filters tab showing `ConsoleLoginErrors` created.

### 7.2 Created the Alarm

- Selected `ConsoleLoginErrors` → **Create alarm**:
  - **Condition:** `ConsoleLoginFailureCount` **Greater/Equal** than **3** (within a 5-minute period)
  - **SNS topic:** Select an existing topic → `MySNSTopic`
  - **Alarm name:** `FailedLogins`
- Chose **Create alarm**.

> 📸 **Screenshot 29:** Alarm condition screen (Greater/Equal, threshold = 3).

> 📸 **Screenshot 30:** Configure actions screen with MySNSTopic selected.

> 📸 **Screenshot 31:** Add name and description screen with "FailedLogins" entered.

> 📸 **Screenshot 32:** Alarms list showing `FailedLogins` created.

### 7.3 Triggered the Alarm

- **IAM console** → **Users** → opened the `test` user → **Security credentials** tab → copied the console sign-in link.
- Opened the sign-in link in a new tab and attempted to log in **at least 3 times** with:
  - **IAM user name:** `test`
  - **Password:** `test` (intentionally incorrect)

> 📸 **Screenshot 33:** Console sign-in page with the test user's sign-in link loaded.

> 📸 **Screenshot 34:** Failed authentication message shown after an incorrect login attempt (repeat as needed to show multiple attempts).

### 7.4 Re-authenticated as voclabs

- Closed all console tabs and reopened the console via the **AWS** link to restore the `voclabs` session.

> 📸 **Screenshot 35:** Console reopened and signed in as voclabs again.

### 7.5 Graphed the Metric

- **CloudWatch console** → **Metrics > All metrics** → **Custom namespaces > CloudTrailMetrics** → **Metrics with no dimensions** → `ConsoleLoginFailureCount`.
- Observed a data point on the graph corresponding to the failed login attempts.

> 📸 **Screenshot 36:** CloudWatch metric graph showing a data point for ConsoleLoginFailureCount.

### 7.6 Checked the Alarm State

- **Alarms > All alarms** → confirmed `FailedLogins` shows state **In alarm**.
- Opened the alarm → **History** tab to confirm it was recently invoked.

> 📸 **Screenshot 37:** Alarms list showing FailedLogins in "In alarm" state.

> 📸 **Screenshot 38:** Alarm History tab showing the state transition.

### 7.7 Verified the Email Notification

- Checked the subscribed inbox for a message about multiple failed login attempts.

> 📸 **Screenshot 39:** Email received from CloudWatch/SNS describing the failed login alarm.

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

> 📸 **Screenshot 40:** Logs Insights query editor with the query entered, before running.

> 📸 **Screenshot 41:** Query results table showing counts grouped by Source_IP, Reason, AWS_Region, and IAM_Arn.

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

## 10. Lab Completion

> 📸 **Screenshot 42:** Grades panel after submitting the lab, showing points earned per task.

**Final Notes / Reflections:** _[optional — e.g., compare when you'd use an EventBridge rule vs. a CloudWatch metric filter/alarm for a given monitoring use case]_
