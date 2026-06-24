# Using Resource-Based Policies to Secure an S3 Bucket

---

## 1. Objectives

By the end of this lab, I was able to:

- Recognize how IAM identity-based policies and resource-based policies provide fine-grained access control to AWS services and resources.
- Describe how an IAM user can assume an IAM role to gain different access permissions within an AWS account.
- Explain how S3 bucket policies and IAM identity-based policies (attached to IAM users/roles) affect what a user can see or modify across AWS services in the Management Console.

---

## 2. Architecture

### 2.1 Starting Architecture

The lab environment was pre-provisioned with:
- One IAM user (`devuser`) belonging to an IAM group (`DeveloperGroup`)
- An identity-based policy (`DeveloperGroupPolicy`) attached to that group
- Three S3 buckets: `bucket1`, `bucket2`, `bucket3`
- A pre-configured IAM role (`BucketsAccessRole`) with its own permissions

> <img width="738" height="354" alt="image" src="https://github.com/user-attachments/assets/06d37ddf-b4f3-4b56-9d71-2f82f72634f1" />


### 2.2 Final Architecture

> <img width="1088" height="515" alt="image" src="https://github.com/user-attachments/assets/760f67a2-1571-4f96-988b-6a4bf1730a41" />

---

## 3. Task 1: Accessing the Console as an IAM User

Steps performed:
1. Started the lab session and waited for the environment status indicator to turn green.
2. Retrieved the `IAMUserLoginURL` and `IAMUserPassword` from the **AWS Details** panel.
3. Logged in as IAM user `devuser`.

> <img width="523" height="814" alt="image" src="https://github.com/user-attachments/assets/79edd810-3a3d-49db-af3c-9d0fd47e7ae1" />

> <img width="591" height="395" alt="image" src="https://github.com/user-attachments/assets/2b88d983-2ad0-4a25-a756-19280c926426" />

---

## 4. Task 2: Attempting Read-Level Access to AWS Services

### 4.1 Amazon EC2 Console

- Navigated to **Services > Compute > EC2**.
- Observed multiple **API Error** messages on the EC2 Dashboard.
- Attempted to view **Instances** → received: *"You are not authorized to perform this operation."*
- Attempted to **Launch instances** → could not select a **Key pair name**, confirming that a required field could not be configured due to missing permissions.

> <img width="1376" height="586" alt="image" src="https://github.com/user-attachments/assets/4d6a9e09-0d08-40c2-9728-df450ac6feda" />

> <img width="1878" height="420" alt="image" src="https://github.com/user-attachments/assets/d4c3f4a3-c521-4009-9b77-96c4d7e07e35" />

> <img width="1127" height="431" alt="image" src="https://github.com/user-attachments/assets/11863359-3865-4182-ae90-d3c349bb6a23" />

**Observation:** `devuser` has no EC2 permissions at all — not even read access in most areas.

### 4.2 Amazon S3 Console

- Navigated to **Services > Storage > S3**.
- All three buckets (`bucket1`, `bucket2`, `bucket3`) were listed, but the **Access** column showed **"Insufficient permissions"** for all three.

> <img width="1448" height="536" alt="image" src="https://github.com/user-attachments/assets/68cb9425-922b-4936-9dc0-6f0d0fd5cc51" />

---

## 5. Task 3: Analyzing the Identity-Based Policy Applied to the IAM User

### 5.1 IAM Console Observations

- Navigated to **Services > Security, Identity, & Compliance > IAM**.
- The IAM dashboard displayed an error: *"User: arn:aws:iam:::user/devuser is not authorized to perform: iam:GetAccountSummary on resource: \*"*

> <img width="948" height="569" alt="image" src="https://github.com/user-attachments/assets/8ad66c05-959c-4bc3-9ca0-2dc1535581cc" />

### 5.2 Group Membership

- Navigated to **User groups** → opened **DeveloperGroup**.
- Confirmed `devuser` is a member (Users tab).
- Opened the **Permissions** tab and found the attached policy: **DeveloperGroupPolicy**.

> <img width="1411" height="645" alt="image" src="https://github.com/user-attachments/assets/811b1df8-bbcd-4f50-9908-32242716090b" />

> <img width="1412" height="624" alt="image" src="https://github.com/user-attachments/assets/9fd24f0e-cf02-4b12-8526-2b443b9115f9" />

### 5.3 Reviewing the Policy JSON

Expanded `DeveloperGroupPolicy` and reviewed the JSON document.

Key findings:
- **No EC2 actions** are allowed anywhere in the policy → explains the total lack of EC2 access.
- **Several read-level IAM actions** are allowed (which is why policy details could be viewed), but `iam:GetAccountSummary` is **not** included.
- For **S3**, only certain **bucket-level** actions are allowed — **no object-level actions** are granted.

> <img width="961" height="719" alt="image" src="https://github.com/user-attachments/assets/f4d2cddf-d32c-45a5-8434-4409769615af" />

**Action taken:** Copied the policy JSON and saved it locally as `DeveloperGroupPolicy.json` for later reference.

---

## 6. Task 4: Attempting Write-Level Access to AWS Services

### 6.1 Create an S3 Bucket (Succeeded)

- Created a new bucket using a unique name (initials + random 4-digit number), Region: **US East (N. Virginia) us-east-1**.
- Bucket creation **succeeded**.

> <img width="1843" height="70" alt="image" src="https://github.com/user-attachments/assets/fa4cd73c-e04d-49bc-85e7-8c5f83904a2f" />

### 6.2 Upload an Object (Failed)

- Opened the new bucket, chose **Upload > Add files**, selected `DeveloperGroupPolicy.json`.
- Upload **failed** with an **Access Denied** error.
- Message: *"You don't have permissions to upload files and folders."*

> <img width="1841" height="98" alt="image" src="https://github.com/user-attachments/assets/26181b80-478a-473e-830f-ed1a1f0e3dfb" />

> <img width="1116" height="507" alt="image" src="https://github.com/user-attachments/assets/8b7de6df-8b51-4715-80d8-95cb11d929e5" />

**Analysis:** `DeveloperGroupPolicy` grants `s3:CreateBucket` (bucket-level action) but does **not** grant `s3:PutObject` (object-level action) — this is consistent with what was observed in the policy JSON in Task 3.

---

## 7. Task 5: Assuming an IAM Role and Reviewing a Resource-Based Policy

### 7.1 Attempting Object Access as `devuser`

- Tried to download `Image2.jpg` from `bucket1` → **AccessDenied** error.
- Tried to download `Image1.jpg` from `bucket2` → same **AccessDenied** error.

> <img width="1920" height="231" alt="image" src="https://github.com/user-attachments/assets/1f4fc847-f7c7-47d7-abc3-71234872e518" />

> <img width="1920" height="235" alt="image" src="https://github.com/user-attachments/assets/e3974187-2c28-4274-bc90-a505ff06017b" />

**Analysis:** Consistent with `DeveloperGroupPolicy` — bucket-level actions are allowed, but object-level actions on `bucket1`/`bucket2` are not.

### 7.2 Assuming `BucketsAccessRole`

- Used **Switch role** (top-right menu) with the Account ID from the AWS Details panel and Role name `BucketsAccessRole`.
- Confirmed the role switch by seeing `BucketsAccessRole` displayed in the top-right corner instead of `devuser`.

> <img width="573" height="354" alt="image" src="https://github.com/user-attachments/assets/9b02e5a0-01b2-4f12-949d-f2cc8aa9b4df" />

### 7.3 Re-testing S3 Access as the Role

- Downloaded `Image2.jpg` from `bucket1` → **succeeded**.

> <img width="1116" height="75" alt="image" src="https://github.com/user-attachments/assets/f4aedf30-a015-4b7e-b016-602bb9a232fa" />

### 7.4 Re-testing IAM Access as the Role

- Navigated to **IAM > User groups** → received a new authorization error (role lacks `iam:ListGroups`).

> <img width="1908" height="528" alt="image" src="https://github.com/user-attachments/assets/de41f5d9-67b6-48e0-9b45-da34873c81fa" />

- Used **Switch back** to return to `devuser`.
- Confirmed access to **User groups** page was restored.

> <img width="1920" height="537" alt="image" src="https://github.com/user-attachments/assets/62f84a33-55ea-4bee-932e-7fb772c855ab" />

### 7.5 Analyzing Policies Attached to `BucketsAccessRole`

Opened **IAM > Roles > BucketsAccessRole** and reviewed two attached policies:

| Policy | Key Permissions |
|---|---|
| `ListAllBucketsPolicy` | `s3:ListAllMyBuckets` on all resources (`*`) |
| `GrantBucket1Access` | `s3:GetObject`, `s3:ListObjects`, `s3:ListBucket` — scoped only to `bucket1` and `bucket1/*` |

> <img width="1400" height="388" alt="image" src="https://github.com/user-attachments/assets/5a5449a3-fb3c-4f55-a244-8375e4f84248" />

> <img width="1409" height="505" alt="image" src="https://github.com/user-attachments/assets/79f3dd6d-c1c3-4809-b96b-9bf4859e25ef" />

**Note:** `GrantBucket1Access` does **not** include `s3:PutObject`, and its resource scope is limited strictly to `bucket1`.

**Action taken:** Saved the `GrantBucket1Access` JSON locally as `GrantBucket1Access.json`.

### 7.6 Trust Relationship

- Opened the **Trust relationships** tab on `BucketsAccessRole`.
- Confirmed `devuser` is listed as a trusted entity allowed to assume this role.
- Verified the account number in the trust policy matches the account number shown in the console (top-right corner).

> <img width="1465" height="451" alt="image" src="https://github.com/user-attachments/assets/6548e50f-eb98-4ad1-be00-2a52632f1f17" />

### 7.7 Unexpected Upload Success to `bucket2`

- While still assuming `BucketsAccessRole`, navigated to `bucket2` (confirmed it did not yet contain `Image2.jpg`).
- Uploaded `Image2.jpg` (previously downloaded from `bucket1`) to `bucket2`.
- Upload **succeeded** — surprising, since no role-based policy explicitly grants `s3:PutObject` on `bucket2`.

> <img width="1496" height="506" alt="image" src="https://github.com/user-attachments/assets/073adb5a-b9e9-4216-a349-0fc160f41e99" />

> <img width="1846" height="623" alt="image" src="https://github.com/user-attachments/assets/db139be7-06c4-487a-a0c9-6548effda9e9" />

**Question raised:** Why did this succeed if no IAM policy attached to the role grants this access? → Answered in Task 6.

---

## 8. Task 6: Understanding Resource-Based Policies

- On `bucket2`'s **Permissions** tab, reviewed the **Bucket policy**.

The bucket policy contains two statements:

| SID | Principal | Allowed Actions | Resource |
|---|---|---|---|
| `S3Write` | `BucketsAccessRole` | `s3:GetObject`, `s3:PutObject` | bucket2 |
| `ListBucket` | `BucketsAccessRole` | `s3:ListBucket` | bucket2 |

> <img width="963" height="705" alt="image" src="https://github.com/user-attachments/assets/17de0de6-ecc9-4216-bc1f-be014ba44987" />

### Key Takeaway

This lab demonstrated how **identity-based policies** (on the role) and **resource-based policies** (on the bucket) work **together**, additively:

- `BucketsAccessRole`'s own IAM policy (`GrantBucket1Access`) granted access **only to bucket1**.
- It granted **no explicit access to bucket2** — but it also did **not explicitly deny** it.
- Because `bucket2`'s **bucket policy** explicitly granted `BucketsAccessRole` permission to `GetObject`/`PutObject`/`ListBucket`, the role was still able to interact with `bucket2`.

This illustrates that **effective permissions are the union of all applicable identity-based and resource-based policies** (absent an explicit `Deny`).

> <img width="856" height="368" alt="image" src="https://github.com/user-attachments/assets/c52fd330-4c6b-4f30-a321-27b6bd03cd22" />

---

## 9. Challenge Task: Uploading `Image2.jpg` to `bucket3`

### 9.1 Attempt as `devuser` (No Role Assumed)

- Switched back to `devuser` (unassumed the role).
- Attempted to upload `Image2.jpg` to `bucket3` → **failed**.
- Attempted to view `bucket3`'s bucket policy → **could not view it** (insufficient permissions).

> <img width="1884" height="748" alt="image" src="https://github.com/user-attachments/assets/e1ee8f7d-2f8f-4769-b48d-fc5441911b6c" />

> <img width="1856" height="463" alt="image" src="https://github.com/user-attachments/assets/618bf7ca-6f12-4af1-bd18-623d0ae01e0c" />

### 9.2 Attempt as `BucketsAccessRole`

- Switched back into `BucketsAccessRole`.
- Re-attempted to view `bucket3`'s bucket policy → _[describe whether you could view it]_.
- Reviewed the policy to determine what principal/actions were allowed.
- Re-attempted the upload of `Image2.jpg` to `bucket3`.

> <img width="1440" height="705" alt="image" src="https://github.com/user-attachments/assets/a5347ad6-3b59-468f-9197-5c539d5487d6" />

> <img width="288" height="189" alt="assume role otherbucketaccessrole" src="https://github.com/user-attachments/assets/12c23473-42b8-49c9-ad85-7cd99afdb305" />

> <img width="919" height="310" alt="successfully uploaded an image" src="https://github.com/user-attachments/assets/6aa7758a-64dd-4cb1-b00e-991fdd14be1a" />

---

## 10. Summary of Key Concepts

- **Identity-based policies** (attached to users, groups, or roles) define what actions a principal can take, generally across resources they specify.
- **Resource-based policies** (e.g., S3 bucket policies) are attached directly to a resource and define which principals can access that specific resource.
- **Effective access = union of identity-based and resource-based policy grants**, unless an explicit `Deny` exists anywhere in the evaluation.
- **Assuming an IAM role** temporarily replaces a user's effective permissions with the role's permissions (via AWS STS temporary credentials), and access reverts when the role is "switched back."
- Bucket-level actions (e.g., `s3:CreateBucket`, `s3:ListAllMyBuckets`) are distinct from object-level actions (e.g., `s3:GetObject`, `s3:PutObject`) — a policy can grant one without the other.

---

## 11. Files Saved During This Lab

- `DeveloperGroupPolicy.json`
- `GrantBucket1Access.json`

---

## 12. Lab Completion

> 📸 **Screenshot 34:** Grades panel after submitting the lab, showing points earned per task.

**Final Notes / Reflections:** _[optional — add anything you found surprising, confusing, or want to remember for the exam]_
