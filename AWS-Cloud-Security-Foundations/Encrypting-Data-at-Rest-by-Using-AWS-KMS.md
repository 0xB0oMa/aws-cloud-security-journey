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

> 📸 **Screenshot 4:** "Create key" — Key type step (Symmetric selected).

> 📸 **Screenshot 5:** Alias step showing `MyKMSKey` entered.

> 📸 **Screenshot 6:** Key administrators step with `voclabs` role selected.

> 📸 **Screenshot 7:** Key usage permissions step with `voclabs` role selected as key user.

> 📸 **Screenshot 8:** Final Review page before choosing Finish.

> 📸 **Screenshot 9:** `MyKMSKey` listed under Customer managed keys after creation.

**Note:** Symmetric KMS keys never leave AWS KMS unencrypted — this is a 256-bit secret key used to generate/encrypt/decrypt data keys, not directly to encrypt large data.

---

## 5. Task 2: Storing an Encrypted Object in an S3 Bucket

### 5.1 Reviewed Bucket Encryption Settings

- Downloaded `clock.png` to the local computer.
- Opened the **Amazon S3 console** → bucket containing `imagebucket` → **Properties** tab.
- Confirmed **Default encryption** is enabled (new objects are automatically encrypted using the bucket default).

> 📸 **Screenshot 10:** Bucket Properties tab showing Default encryption enabled.

### 5.2 Uploaded `clock.png` with Explicit SSE-KMS Encryption

- **Objects** tab → **Upload** → **Add files** → selected `clock.png`.
- Expanded **Properties** → **Server-side encryption settings**:
  - Selected **Specify an encryption key**.
  - **Encryption key type:** AWS Key Management Service key (SSE-KMS).
  - **AWS KMS key:** Choose from your AWS KMS keys → `MyKMSKey`.
- Chose **Upload**, then **Close**.

> 📸 **Screenshot 11:** Upload screen with Server-side encryption settings configured for SSE-KMS using MyKMSKey.

> 📸 **Screenshot 12:** Bucket Objects list showing `clock.png` after a successful upload.

### 5.3 Verified Object-Level Encryption Settings

- Opened `clock.png` → **Properties** tab → confirmed **SSE-KMS** is enabled on the object.

> 📸 **Screenshot 13:** clock.png Properties tab showing SSE-KMS server-side encryption enabled.

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

> 📸 **Screenshot 14:** Access Denied error when loading the object URL directly.

### 6.2 Opened Up Public Access

- Bucket → **Permissions** tab → **Block public access** → **Edit** → cleared **Block all public access** → **Save changes** → typed `confirm`.
- **Permissions** tab → **Object Ownership** → **Edit** → selected **ACLs enabled** → acknowledged → kept **Bucket owner preferred** → **Save changes**.
- **Objects** tab → selected `clock.png` → **Actions > Make public using ACL** → **Make public** → **Close**.

> 📸 **Screenshot 15:** Block public access settings being edited (checkbox cleared).

> 📸 **Screenshot 16:** Confirmation dialog after typing "confirm."

> 📸 **Screenshot 17:** Object Ownership edit screen with "ACLs enabled" selected.

> 📸 **Screenshot 18:** "Make public using ACL" confirmation for clock.png.

### 6.3 Re-tested the Object URL (Invalid Argument)

- Refreshed the previously opened object URL tab.
- Result changed from **Access Denied** to **Invalid Argument**, with a message indicating that SSE-KMS requests require AWS Signature Version 4.

> 📸 **Screenshot 19:** "Invalid Argument" error after making the object public.

**Analysis:** Making the object public removed the *access* barrier, but the object is still **encrypted**. Public, unauthenticated requests can't supply the Signature Version 4 authentication that SSE-KMS requires, so the object remains unreadable to anonymous requesters. This demonstrates encryption as a second layer of defense even if access permissions are misconfigured.

---

## 7. Task 4: Attempting Signed Access to the Encrypted Object

- In the S3 console (authenticated session), selected `clock.png` → **Open**.
- The image opened successfully in a new tab.
- Inspected the resulting URL and noted multiple `X-Amz-*` query parameters — evidence of automatically-included Signature Version 4 authentication info.

> 📸 **Screenshot 20:** clock.png image successfully displayed after opening from the console.

> 📸 **Screenshot 21:** Browser address bar showing the signed URL with X-Amz- parameters (credentials redacted/blurred if needed).

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

> 📸 **Screenshot 22:** CloudTrail Event history filtered by event source = kms.amazonaws.com.

### 8.1 GenerateDataKey Event

- Opened the **GenerateDataKey** event and reviewed the event record.
- Confirmed the `keyId` matches `MyKMSKey`'s key ID, the target S3 object ARN, and the `principalId` of the requester.

> 📸 **Screenshot 23:** GenerateDataKey event record with keyId, eventName, and S3 object ARN highlighted/visible.

### 8.2 Decrypt Event

- Returned to Event history → opened the **Decrypt** event.
- Confirmed it corresponds to opening `clock.png` from the console, again showing the requester identity, the KMS key used, and the decrypted S3 object.

> 📸 **Screenshot 24:** Decrypt event record showing requester identity, key used, and object decrypted.

**Analysis:** CloudTrail's Event history retains 90 days of API activity by default. For longer retention/analysis, a CloudTrail **trail** would need to be created.

---

## 9. Task 6: Encrypting the Root Volume of an Existing EC2 Instance

### 9.1 Reviewed Current (Unencrypted) Storage

- **EC2 console** → **Instances** → `LabInstance` → **Storage** tab.
- Confirmed the attached root volume shows **Not encrypted**.

> 📸 **Screenshot 25:** LabInstance Storage tab showing the unencrypted root volume.

### 9.2 Stopped the Instance

- Selected `LabInstance` → **Instance state > Stop instance** → confirmed.

> 📸 **Screenshot 26:** Instance state showing "Stopped" after confirming the stop action.

### 9.3 Created a Snapshot of the Unencrypted Volume

- **Storage** tab → opened the Volume ID link (twice, per instructions) to reach volume details.
- Noted the **Availability Zone** of the volume.
- **Actions > Create snapshot** → added tag `Name = Unencrypted Root Volume` → **Create snapshot**.

> 📸 **Screenshot 27:** Volume details page showing the Availability Zone.

> 📸 **Screenshot 28:** Create snapshot dialog with the Name tag added.

### 9.4 Created an Encrypted Volume from the Snapshot

- **Snapshots** → opened the `Unencrypted Root Volume` snapshot → waited for status **Completed**.
- Confirmed the snapshot itself shows **Not encrypted**.
- **Actions > Create volume from snapshot**:
  - **Availability Zone:** matched the original volume's AZ.
  - Selected **Encrypt this volume**.
  - **KMS key:** `MyKMSKey`.
  - **Create volume**.

> 📸 **Screenshot 29:** Snapshot status = Completed, encryption = Not encrypted.

> 📸 **Screenshot 30:** "Create volume from snapshot" dialog with Encrypt this volume + MyKMSKey selected.

### 9.5 Labeled the Volumes

- **Volumes** list now shows two volumes:
  - In-use volume → renamed to **Old unencrypted root volume**
  - Available volume → renamed to **New encrypted root volume**

> 📸 **Screenshot 31:** Volumes list showing both volumes renamed appropriately.

### 9.6 Swapped the Root Volume

- Selected **Old unencrypted root volume** → **Actions > Detach volume** → confirmed.
- Selected **New encrypted root volume** → **Actions > Attach volume**:
  - **Instance:** LabInstance (stopped)
  - **Device name:** `/dev/xvda`
  - **Attach volume**.

> 📸 **Screenshot 32:** Detach volume confirmation for the old unencrypted volume.

> 📸 **Screenshot 33:** Attach volume dialog showing LabInstance and device name `/dev/xvda`.

### 9.7 Verified the New Encrypted Root Volume

- Returned to **Instances** → `LabInstance` → **Storage** tab.
- Confirmed the attached volume now shows **Encrypted** with a KMS Key ID.
- **Did not** start the instance yet (per instructions — reserved for Task 7).

> 📸 **Screenshot 34:** LabInstance Storage tab showing the new volume as Encrypted with a KMS Key ID.

**Process summary (per lab diagram):** Stop instance → detach volume → snapshot unencrypted volume → create encrypted volume from snapshot → attach encrypted volume → start instance.

---

## 10. Task 7: Disabling the Encryption Key and Observing the Effects

### 10.1 Disabled MyKMSKey

- **KMS console** → **Customer managed keys** → selected `MyKMSKey` → **Key actions > Disable** → confirmed.

> 📸 **Screenshot 35:** Confirmation dialog for disabling MyKMSKey.

> 📸 **Screenshot 36:** Customer managed keys list showing MyKMSKey status = Disabled.

### 10.2 Attempted to Start the EC2 Instance (Failed)

- **EC2 console** → selected `LabInstance` → **Instance state > Start instance**.
- Instance state briefly showed **Pending**, then reverted to **Stopped**.

> 📸 **Screenshot 37:** Instance state cycling from Pending back to Stopped after the failed start attempt.

### 10.3 Attempted to Open the S3 Object (Failed)

- **S3 console** → bucket → selected `clock.png` → **Open**.
- Result: **KMS.DisabledException** error — the same object that opened successfully in Task 4 now fails.

> 📸 **Screenshot 38:** KMS.DisabledException error when attempting to open clock.png.

### 10.4 Analyzed the Cause in CloudTrail

- **CloudTrail console** → **Event history**.
- Reviewed the **DisableKey** event (confirms when the key was disabled).
- Reviewed the **StartInstances** event immediately after (the API call itself succeeded).
- Reviewed the **CreateGrant** event immediately after that — showed an **error message indicating the key is disabled**.

> 📸 **Screenshot 39:** DisableKey event record.

> 📸 **Screenshot 40:** StartInstances event record (API call succeeded).

> 📸 **Screenshot 41:** CreateGrant event record showing the error indicating the key is disabled.

**Analysis:** Starting the instance required EC2 to ask KMS for the plaintext data key to decrypt the encrypted root volume. Because `MyKMSKey` was disabled, KMS refused to issue the data key, so the guest OS files on the root volume couldn't be decrypted and the instance could never reach the **Running** state — even though the `StartInstances` API call itself was accepted. The same logic explains why `clock.png` could no longer be decrypted: its data key also depends on `MyKMSKey`.

### 10.5 Re-enabled the Key and Restarted the Instance

- **KMS console** → selected `MyKMSKey` → **Key actions > Enable**.
- **EC2 console** → `LabInstance` → **Instance state > Start instance**.
- Waited for the instance state to reach **Running** before submitting the lab.

> 📸 **Screenshot 42:** MyKMSKey status showing Enabled again.

> 📸 **Screenshot 43:** LabInstance state showing Running after re-enabling the key.

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
