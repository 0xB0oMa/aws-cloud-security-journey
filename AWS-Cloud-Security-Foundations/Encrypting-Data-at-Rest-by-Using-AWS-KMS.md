# Lab 5.1: Encrypting Data at Rest by Using AWS KMS

---

## 1. Objectives

By the end of this lab, I was able to:

- Create an AWS KMS customer managed key to encrypt and decrypt data at rest.
- Store an encrypted object in an S3 bucket by using an encryption key.
- Attempt public access and signed access to an encrypted S3 object.
- Monitor encryption key usage by using the CloudTrail event history.
- Encrypt the root volume of an existing EC2 instance.
- Disable and re-enable an AWS KMS key and observe the effects on data access.

---

## 2. Architecture

### 2.1 Starting Architecture

- An empty S3 bucket: `ImageBucket`
- An EC2 instance: `LabInstance`, with an **unencrypted** root EBS volume attached

> <img width="1096" height="558" alt="image" src="https://github.com/user-attachments/assets/6aa6a704-05a5-4589-b060-dcc399c11102" />

### 2.2 Final Architecture

- `ImageBucket` containing an object encrypted with **SSE-KMS** (encrypted data key stored as object metadata)
- `LabInstance` root volume swapped for an **encrypted** EBS volume (created from a snapshot of the original)
- A customer managed KMS key, `MyKMSKey`, used by both S3 and EBS for encrypt/decrypt operations
- CloudTrail logging all KMS key usage as events

> <img width="1088" height="555" alt="image" src="https://github.com/user-attachments/assets/500f1e3e-1b0a-46d1-88df-4a814efca2a8" />

---

## 4. Task 1: Creating an AWS KMS Key

- Opened the **AWS KMS console** → **Customer managed keys** → **Create key**.
- **Key type:** Symmetric.
- **Alias:** `MyKMSKey`.
- **Key administrators:** `voclabs` role.
- **Key users:** `voclabs` role.
- Reviewed settings and chose **Finish**.

> <img width="1904" height="474" alt="image" src="https://github.com/user-attachments/assets/d9018e16-8056-4382-8cc2-7cd19055c923" />

> <img width="1859" height="321" alt="image" src="https://github.com/user-attachments/assets/f0820882-d453-45c1-9fe6-81830e58a185" />

> <img width="1873" height="496" alt="image" src="https://github.com/user-attachments/assets/4928bbe9-bf9c-49e2-afee-758aedcf6fd5" />

> <img width="1850" height="493" alt="image" src="https://github.com/user-attachments/assets/b4a46d83-2f9c-4b45-97e3-4dc4b2975b37" />

> <img width="1885" height="393" alt="image" src="https://github.com/user-attachments/assets/e73dc474-6ed8-43fe-afb2-f47e61098da6" />

**Note:** Symmetric KMS keys never leave AWS KMS unencrypted — this is a 256-bit secret key used to generate/encrypt/decrypt data keys, not directly to encrypt large data.

---

## 5. Task 2: Storing an Encrypted Object in an S3 Bucket

### 5.1 Reviewed Bucket Encryption Settings

- Downloaded `clock.png` to the local computer.
- Opened the **Amazon S3 console** → bucket containing `imagebucket` → **Properties** tab.
- Confirmed **Default encryption** is enabled (new objects are automatically encrypted using the bucket default).

> <img width="1867" height="439" alt="image" src="https://github.com/user-attachments/assets/e88a4ca3-a2ee-4904-bf98-416972c39dec" />

### 5.2 Uploaded `clock.png` with Explicit SSE-KMS Encryption

- **Objects** tab → **Upload** → **Add files** → selected `clock.png`.
- Expanded **Properties** → **Server-side encryption settings**:
  - Selected **Specify an encryption key**.
  - **Encryption key type:** AWS Key Management Service key (SSE-KMS).
  - **AWS KMS key:** Choose from your AWS KMS keys → `MyKMSKey`.
- Chose **Upload**, then **Close**.

> <img width="1240" height="725" alt="image" src="https://github.com/user-attachments/assets/fa7adf45-3c0d-4e3e-84ee-4dc34fc76862" />

> <img width="1847" height="647" alt="image" src="https://github.com/user-attachments/assets/b5f4a160-e103-4126-a658-9b0d3fe5ab19" />

### 5.3 Verified Object-Level Encryption Settings

- Opened `clock.png` → **Properties** tab → confirmed **SSE-KMS** is enabled on the object.

> <img width="1864" height="429" alt="image" src="https://github.com/user-attachments/assets/5b58363b-06ec-4ff0-b471-d6c4954e7f7f" />

**How it worked (per lab diagram):**
1. Requested upload of an encrypted object.
2. S3 requested a data key from KMS.
3. KMS generated a plaintext data key and encrypted a copy of it using `MyKMSKey`.
4. KMS returned both the plaintext and encrypted copies of the data key to S3.
5. S3 used the plaintext data key to encrypt the object, stored the object with the **encrypted** data key as metadata, and discarded the plaintext data key.

---

## 6. Task 3: Attempting Public Access to the Encrypted Object

### 6.1 Initial Attempt (Access Denied)

- Copied the **Object URL** for `clock.png` and opened it in a new tab.
- Result: **Access Denied** (bucket still has Block Public Access enabled by default).

> <img width="1306" height="288" alt="image" src="https://github.com/user-attachments/assets/978ae089-9ae3-4ab7-bffd-cf1cbb531ba2" />

### 6.2 Opened Up Public Access

- Bucket → **Permissions** tab → **Block public access** → **Edit** → cleared **Block all public access** → **Save changes** → typed `confirm`.
- **Permissions** tab → **Object Ownership** → **Edit** → selected **ACLs enabled** → acknowledged → kept **Bucket owner preferred** → **Save changes**.
- **Objects** tab → selected `clock.png` → **Actions > Make public using ACL** → **Make public** → **Close**.

> <img width="1543" height="587" alt="image" src="https://github.com/user-attachments/assets/5357d3b7-ca75-49e4-8554-9ec003b2538d" />

> <img width="741" height="363" alt="image" src="https://github.com/user-attachments/assets/791fe277-6173-4d8b-9bf1-a514e7f6503d" />

> <img width="1471" height="646" alt="image" src="https://github.com/user-attachments/assets/694b8bfb-d616-4e59-b0bf-544424a0867b" />

> <img width="1880" height="504" alt="image" src="https://github.com/user-attachments/assets/6cd0920b-1558-441f-8773-d1cae189a910" />

### 6.3 Re-tested the Object URL (Invalid Argument)

- Refreshed the previously opened object URL tab.
- Result changed from **Access Denied** to **Invalid Argument**, with a message indicating that SSE-KMS requests require AWS Signature Version 4.

> <img width="1348" height="299" alt="image" src="https://github.com/user-attachments/assets/43ed553f-d4a4-4e31-8ee2-43c1270c50b7" />

**Analysis:** Making the object public removed the *access* barrier, but the object is still **encrypted**. Public, unauthenticated requests can't supply the Signature Version 4 authentication that SSE-KMS requires, so the object remains unreadable to anonymous requesters. This demonstrates encryption as a second layer of defense even if access permissions are misconfigured.

---

## 7. Task 4: Attempting Signed Access to the Encrypted Object

- In the S3 console (authenticated session), selected `clock.png` → **Open**.
- The image opened successfully in a new tab.
- Inspected the resulting URL and noted multiple `X-Amz-*` query parameters — evidence of automatically-included Signature Version 4 authentication info.

> <img width="1435" height="849" alt="image" src="https://github.com/user-attachments/assets/416d2bc7-bbcb-46ee-8e57-f04bfde0040f" />

> <img width="1128" height="36" alt="image" src="https://github.com/user-attachments/assets/d7672520-b1a7-4ab4-a4a9-f6f95ab4aa43" />

**How it worked (per lab diagram):**
1. Requested to open the object via the console (authenticated).
2. S3 detected the object was encrypted; the request automatically included Signature Version 4 auth info.
3. S3 sent the encrypted data key to KMS.
4. KMS decrypted the data key using `MyKMSKey` (the key itself never left KMS).
5. KMS returned the plaintext data key to S3.
6. S3 decrypted the object, displayed it, and discarded the plaintext data key.

---

## 8. Task 5: Monitoring AWS KMS Activity by Using CloudTrail

- Opened the **CloudTrail console** → **Event history**.
- Changed the filter from **Read-only** to **Event source**, and filtered on `kms.amazonaws.com`.

> <img width="1862" height="707" alt="image" src="https://github.com/user-attachments/assets/fd90aa4c-9f68-4252-b377-29a37ca006d9" />

### 8.1 GenerateDataKey Event

- Opened the **GenerateDataKey** event and reviewed the event record.
- Confirmed the `keyId` matches `MyKMSKey`'s key ID, the target S3 object ARN, and the `principalId` of the requester.

> <img width="1099" height="344" alt="image" src="https://github.com/user-attachments/assets/afa23078-12b8-4236-9616-0d7036bfd2ca" />

### 8.2 Decrypt Event

- Returned to Event history → opened the **Decrypt** event.
- Confirmed it corresponds to opening `clock.png` from the console, again showing the requester identity, the KMS key used, and the decrypted S3 object.

> <img width="934" height="689" alt="image" src="https://github.com/user-attachments/assets/361841e7-5a4b-4ae4-84d4-64fdd154e2c8" />

**Analysis:** CloudTrail's Event history retains 90 days of API activity by default. For longer retention/analysis, a CloudTrail **trail** would need to be created.

---

## 9. Task 6: Encrypting the Root Volume of an Existing EC2 Instance

### 9.1 Reviewed Current (Unencrypted) Storage

- **EC2 console** → **Instances** → `LabInstance` → **Storage** tab.
- Confirmed the attached root volume shows **Not encrypted**.

> <img width="1579" height="638" alt="image" src="https://github.com/user-attachments/assets/e27b1fcb-2be6-42cb-9f96-e976cacb4be7" />

### 9.2 Stopped the Instance

- Selected `LabInstance` → **Instance state > Stop instance** → confirmed.

> <img width="1587" height="250" alt="image" src="https://github.com/user-attachments/assets/b494fa74-c846-4fd3-8091-aeecdfaae899" />

### 9.3 Created a Snapshot of the Unencrypted Volume

- **Storage** tab → opened the Volume ID link (twice, per instructions) to reach volume details.
- Noted the **Availability Zone** of the volume.
- **Actions > Create snapshot** → added tag `Name = Unencrypted Root Volume` → **Create snapshot**.

> <img width="1583" height="263" alt="image" src="https://github.com/user-attachments/assets/4da22ede-7a0b-4eb5-bd10-a9a498a89aed" />

> <img width="1852" height="704" alt="image" src="https://github.com/user-attachments/assets/1319b0ed-c173-46ce-9f11-3728f73bdb3a" />

### 9.4 Created an Encrypted Volume from the Snapshot

- **Snapshots** → opened the `Unencrypted Root Volume` snapshot → waited for status **Completed**.
- Confirmed the snapshot itself shows **Not encrypted**.
- **Actions > Create volume from snapshot**:
  - **Availability Zone:** matched the original volume's AZ.
  - Selected **Encrypt this volume**.
  - **KMS key:** `MyKMSKey`.
  - **Create volume**.

> <img width="1604" height="206" alt="image" src="https://github.com/user-attachments/assets/832ea7e1-30cc-47a9-88fe-503fe56c0e57" />

> <img width="1079" height="650" alt="image" src="https://github.com/user-attachments/assets/a5845ef8-ae5e-42ec-b2e3-9e84879629c6" />

### 9.5 Labeled the Volumes

- **Volumes** list now shows two volumes:
  - In-use volume → renamed to **Old unencrypted root volume**
  - Available volume → renamed to **New encrypted root volume**

> <img width="1590" height="245" alt="image" src="https://github.com/user-attachments/assets/83c1ec81-6b3e-413a-9006-8819c116f361" />

### 9.6 Swapped the Root Volume

- Selected **Old unencrypted root volume** → **Actions > Detach volume** → confirmed.
- Selected **New encrypted root volume** → **Actions > Attach volume**:
  - **Instance:** LabInstance (stopped)
  - **Device name:** `/dev/xvda`
  - **Attach volume**.

> <img width="1561" height="316" alt="image" src="https://github.com/user-attachments/assets/40884f7e-c2b8-4268-99fe-267d9abc3d29" />

> <img width="1857" height="647" alt="image" src="https://github.com/user-attachments/assets/dfc59622-8f61-471b-955c-a2871c393acb" />

### 9.7 Verified the New Encrypted Root Volume

- Returned to **Instances** → `LabInstance` → **Storage** tab.
- Confirmed the attached volume now shows **Encrypted** with a KMS Key ID.
- **Did not** start the instance yet (per instructions — reserved for Task 7).

> <img width="1586" height="582" alt="image" src="https://github.com/user-attachments/assets/c47b4633-6971-46e0-8877-28808700cd67" />

**Process summary (per lab diagram):** Stop instance → detach volume → snapshot unencrypted volume → create encrypted volume from snapshot → attach encrypted volume → start instance.

---

## 10. Task 7: Disabling the Encryption Key and Observing the Effects

### 10.1 Disabled MyKMSKey

- **KMS console** → **Customer managed keys** → selected `MyKMSKey` → **Key actions > Disable** → confirmed.

> <img width="737" height="588" alt="image" src="https://github.com/user-attachments/assets/450e8e50-fcd7-476e-9f8d-9a6f2aaddaac" />

> <img width="1542" height="218" alt="image" src="https://github.com/user-attachments/assets/0bf66162-ad91-4455-9697-6f0b3ec87450" />

### 10.2 Attempted to Start the EC2 Instance (Failed)

- **EC2 console** → selected `LabInstance` → **Instance state > Start instance**.
- Instance state briefly showed **Pending**, then reverted to **Stopped**.

> <img width="1580" height="220" alt="image" src="https://github.com/user-attachments/assets/70d2b55c-ffdc-440a-a0ef-e4b654dfb05f" />

### 10.3 Attempted to Open the S3 Object (Failed)

- **S3 console** → bucket → selected `clock.png` → **Open**.
- Result: **KMS.DisabledException** error — the same object that opened successfully in Task 4 now fails.

> <img width="1222" height="211" alt="image" src="https://github.com/user-attachments/assets/cb6d2088-a5ee-4758-b531-65d9ff3cbb98" />

### 10.4 Analyzed the Cause in CloudTrail

- **CloudTrail console** → **Event history**.
- Reviewed the **DisableKey** event (confirms when the key was disabled).
- Reviewed the **StartInstances** event immediately after (the API call itself succeeded).
- Reviewed the **CreateGrant** event immediately after that — showed an **error message indicating the key is disabled**.

> <img width="706" height="106" alt="image" src="https://github.com/user-attachments/assets/032f3828-92f8-43b8-b1e6-8fc01e03dfbb" />

> <img width="871" height="114" alt="image" src="https://github.com/user-attachments/assets/f2a34fc7-d3e0-45f0-82b7-f4017240873e" />

> <img width="1017" height="218" alt="image" src="https://github.com/user-attachments/assets/278cc24c-1ec9-4e6e-b807-74d9a92f1698" />

**Analysis:** Starting the instance required EC2 to ask KMS for the plaintext data key to decrypt the encrypted root volume. Because `MyKMSKey` was disabled, KMS refused to issue the data key, so the guest OS files on the root volume couldn't be decrypted and the instance could never reach the **Running** state — even though the `StartInstances` API call itself was accepted. The same logic explains why `clock.png` could no longer be decrypted: its data key also depends on `MyKMSKey`.

### 10.5 Re-enabled the Key and Restarted the Instance

- **KMS console** → selected `MyKMSKey` → **Key actions > Enable**.
- **EC2 console** → `LabInstance` → **Instance state > Start instance**.
- Waited for the instance state to reach **Running** before submitting the lab.

> <img width="1540" height="313" alt="image" src="https://github.com/user-attachments/assets/1e5042cd-dae3-4460-8793-fc016ba58a71" />

> <img width="1591" height="292" alt="image" src="https://github.com/user-attachments/assets/25510d91-6a70-49bd-9fbc-d4c9cf090706" />

---

## 11. Summary of Key Concepts

| Concept | Key Takeaway |
|---|---|
| Customer managed KMS key | A symmetric key used to generate, encrypt, and decrypt **data keys**; the key material never leaves KMS. |
| Envelope encryption | Data is encrypted with a plaintext data key, which is itself encrypted by the KMS key and stored alongside the data as metadata. |
| SSE-KMS on S3 | Encrypts objects using a KMS-managed data key; decryption requires both **authorization** (IAM/bucket policy) **and** access to the KMS key. |
| Signature Version 4 | Required for any request involving SSE-KMS objects — unauthenticated/unsigned requests (e.g., a plain object URL) cannot decrypt SSE-KMS content even if made "public." |
| CloudTrail + KMS | Every KMS API call (GenerateDataKey, Decrypt, DisableKey, CreateGrant, etc.) is logged, providing an audit trail of key usage. |
| Encrypting an existing volume | Requires stop → detach → snapshot → create encrypted volume from snapshot → attach → start (you cannot encrypt a volume in place). |
| Disabling a KMS key | Immediately blocks all new decrypt/data-key operations for anything encrypted under that key — including EC2 boot volumes and S3 objects — until the key is re-enabled. |

---
