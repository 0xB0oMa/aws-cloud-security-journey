Securing VPC Resources by Using Security Groups

---

## 1. Objectives

By the end of this lab, I was able to:

- Examine security groups to determine what traffic is allowed.
- Change which security groups are applied to EC2 instances.
- Create new security groups.
- Update inbound rules on security groups to follow the principle of least privilege.
- Understand how security groups can reference other security groups.
- Configure a network ACL (NACL) to block traffic on a specific TCP port.
- Connect to an instance in a private subnet by using SSH (via a bastion host).
- Connect to an instance in a private subnet by using AWS Systems Manager Session Manager.

---

## 2. Architecture

### 2.1 Starting Architecture

The lab environment included:
- A custom VPC (`LabVPC`, CIDR `10.0.0.0/16`) alongside the default VPC (`172.31.0.0/16`)
- Three subnets: `PrivateSubnet`, `PublicSubnetA`, `PublicSubnetB`
- A NAT gateway (in `PublicSubnetA`) and an internet gateway
- Three EC2 instances:
  - `AppServer` — private subnet, runs the application web server
  - `ProxyServer1` — public subnet A, forwards HTTP traffic to AppServer
  - `ProxyServer2` — public subnet B, forwards HTTP traffic to AppServer
- Security groups: `AppServerSG`, `ProxySG`, `ProxySG2`

> <img width="748" height="501" alt="image" src="https://github.com/user-attachments/assets/f2f42dbe-6fa0-491f-a421-21a647798441" />

---

## 4. Task 1: Analyzing the VPC and Private Subnet Resource Settings

### 4.1 VPCs and Subnets

- Opened the **Amazon VPC console**.
- Confirmed two VPCs exist: the default VPC (`172.31.0.0/16`) and `LabVPC` (`10.0.0.0/16`).
- Viewed **Subnets**, sorted by VPC column, and identified the three subnets in `LabVPC`: `PrivateSubnet`, `PublicSubnetA`, `PublicSubnetB`.

> <img width="1920" height="376" alt="image" src="https://github.com/user-attachments/assets/92b1e80b-c185-4783-a1e4-1683970bd47f" />

> <img width="1920" height="432" alt="image" src="https://github.com/user-attachments/assets/ee1a93ed-5c87-4781-887e-2e9ce262756f" />

### 4.2 PrivateSubnet Routing

- Selected `PrivateSubnet` → **Route table** tab → opened the associated route table.
- Observed routes:
  - `10.0.0.0/16` → **local**
  - `0.0.0.0/0` → **NAT gateway**
- Renamed the route table from `changeme` to **Private** via the Tags tab.
- Opened the NAT gateway details and confirmed it runs in `PublicSubnetA`.

> <img width="1891" height="240" alt="image" src="https://github.com/user-attachments/assets/50ac5405-6d46-4cca-8b39-0e6cde96bdc7" />

> <img width="1558" height="373" alt="image" src="https://github.com/user-attachments/assets/ca53e162-1c4a-4b40-a93a-b8ec4aad7aff" />

> <img width="1557" height="102" alt="image" src="https://github.com/user-attachments/assets/24cb4b2e-cb15-4f57-8d67-86d0c4f422b3" />

> <img width="746" height="499" alt="image" src="https://github.com/user-attachments/assets/609ab6a3-471c-43ef-9857-77be3373d473" />

**Analysis:** Because the private subnet routes outbound traffic through a NAT gateway (not directly to an internet gateway), instances there can initiate outbound connections to the internet, but they are not directly reachable from the internet and have no public IP.

### 4.3 AppServer Instance & Security Group

- Opened the **Amazon EC2 console** (in a new tab) → **Instances** → selected `AppServer`.
- Confirmed: running in `PrivateSubnet`, has a private IPv4 address only (no public IP).
- Opened the **Security** tab → followed the link to `AppServerSG`.
- **Inbound rules:** HTTP (TCP 80) from source `0.0.0.0/0` (anywhere).
- **Outbound rules:** All traffic allowed (default).

> <img width="1557" height="587" alt="image" src="https://github.com/user-attachments/assets/75aad526-2faf-4d16-bdfe-199d6c14f3a0" />

> <img width="1887" height="322" alt="image" src="https://github.com/user-attachments/assets/3d7b311c-48a5-4661-b8f2-af5e22e97b45" />

> <img width="1885" height="319" alt="image" src="https://github.com/user-attachments/assets/03c7cba0-fe03-48d3-beab-8c96eaaedd44" />

**Key concept noted:** Security groups are **stateful** — return traffic for an allowed outbound/inbound request is automatically permitted regardless of the rules in the other direction.

---

## 5. Task 2: Analyzing the Public Subnet Resource Settings

### 5.1 PublicSubnetA Routing

- Selected `PublicSubnetA` → **Route table** tab.
- Confirmed `0.0.0.0/0` routes to an **internet gateway** (not a NAT gateway) — this is what makes it a "public" subnet.
- Verified the internet gateway is associated with `LabVPC`.

> <img width="1573" height="298" alt="image" src="https://github.com/user-attachments/assets/5aca541e-8d37-4833-89f3-811fc5ce2997" />

> <img width="1566" height="254" alt="image" src="https://github.com/user-attachments/assets/6dc86c69-4f59-4c4c-b4d0-004c34195596" />

### 5.2 ProxyServer1 (PublicSubnetA)

- Selected `ProxyServer1` in the EC2 console.
- Confirmed it has a **public IPv4 address** (unlike AppServer).
- Security tab → security group `ProxySG` → inbound rule allows HTTP/80 from anywhere.

> <img width="1560" height="358" alt="image" src="https://github.com/user-attachments/assets/d71602c5-a481-4997-8a6a-73e38cfa314e" />

> <img width="1888" height="319" alt="image" src="https://github.com/user-attachments/assets/adcd8c5f-6695-4c16-8298-9fbfd19b8f91" />

### 5.3 ProxyServer2 (PublicSubnetB) — Add Inbound Rule

- Selected `ProxyServer2`, confirmed it runs in `PublicSubnetB` with a public IP, and is associated with `ProxySG2`.
- Edited `ProxySG2` inbound rules → added a new rule: **Type = HTTP**, **Source = Anywhere-IPv4** → saved.

> <img width="1555" height="400" alt="image" src="https://github.com/user-attachments/assets/d4af9d72-4e06-42fb-85c6-f8e5661413b3" />

> <img width="1901" height="426" alt="image" src="https://github.com/user-attachments/assets/c7eda788-d770-4eb6-ba88-3dd1ba91365a" />

> <img width="1896" height="575" alt="image" src="https://github.com/user-attachments/assets/a88e3885-5174-4b11-a33f-1d64d0b41102" />

> <img width="751" height="499" alt="image" src="https://github.com/user-attachments/assets/5a7aaf21-69cb-48fa-a3d7-42dd6f7af57b" />

---

## 6. Task 3: Testing HTTP Connectivity from Public EC2 Instances

- Copied `ProxyServer1`'s **public IPv4 address** and loaded it in a new browser tab → the AppServer's website loaded successfully (forwarded via the proxy).
- Repeated the same test with `ProxyServer2`'s public IP → website also loaded successfully.
- Closed both tabs.

> <img width="1670" height="169" alt="image" src="https://github.com/user-attachments/assets/1e48726b-ab10-4b6b-8aa2-9f1e8ebf0d35" />

> <img width="1662" height="152" alt="image" src="https://github.com/user-attachments/assets/e8562820-8e66-40c9-9a6d-37414624680b" />

**Analysis:** At this point, both `ProxySG` and `ProxySG2` allow inbound HTTP from anywhere, and `AppServerSG` allows inbound HTTP from anywhere — so both proxy paths to the AppServer succeed.

---

## 7. Task 4: Restricting HTTP Access by Using an IP Address

<img width="899" height="497" alt="image" src="https://github.com/user-attachments/assets/678bbc13-dda9-404f-a3ab-24bb28920f0a" />

- Retrieved `ProxyServer1PrivateIP` from the **AWS Details** panel.
- Edited `AppServerSG` inbound rules:
  - Removed the existing `0.0.0.0/0` source on the HTTP rule.
  - Added `ProxyServer1PrivateIP/32` as the new source.
  - Saved rules.

> <img width="1231" height="408" alt="image" src="https://github.com/user-attachments/assets/8c04513c-98cb-403d-b4e8-39323baddf75" />

> <img width="1892" height="325" alt="image" src="https://github.com/user-attachments/assets/5626388f-4b3f-4cfe-9bcf-4351a2d95913" />

### 7.1 Re-test Access

- **Via ProxyServer1's public IP:** website loaded successfully (source IP matches the new rule).
- **Via ProxyServer2's public IP:** connection **timed out** (source IP does not match the rule).

> <img width="1672" height="157" alt="image" src="https://github.com/user-attachments/assets/c519250b-ebbb-41cd-8a20-6245c74a3dd1" />

> <img width="1492" height="828" alt="image" src="https://github.com/user-attachments/assets/7e88ba35-f9cf-4215-9cb7-08554a992618" />

**Analysis:** `AppServerSG` now only accepts HTTP traffic whose source IP is `ProxyServer1`'s private IP, so traffic forwarded from `ProxyServer2` is rejected even though `ProxySG2` itself still allows the inbound connection from the internet to ProxyServer2.

---

## 8. Task 5: Scaling Restricted HTTP Access by Referencing a Security Group

Goal: Avoid hardcoding individual IPs by referencing a **security group** as the allowed source instead.

### 8.1 Update AppServerSG to Reference ProxySG

- Edited `AppServerSG` inbound rules:
  - Deleted the IP-based rule from Task 4.
  - Added a new rule: **Type = HTTP**, **Source = Custom**, typed `sg` in the source field, and selected the `ProxySG` security group from the dropdown.
  - Saved rules.

> <img width="1896" height="521" alt="image" src="https://github.com/user-attachments/assets/32dc5e6e-43e4-4396-a6fb-6262125206d2" />

### 8.2 Reassign ProxyServer2 to ProxySG

- Selected `ProxyServer2` → **Actions > Security > Change security groups**.
- Removed `ProxySG2`, added `ProxySG`, saved.

> <img width="1853" height="386" alt="image" src="https://github.com/user-attachments/assets/0db74f9c-d243-4a6c-a482-2ebd924100d0" />

### 8.3 Re-test Access

- **Via ProxyServer1's public IP:** website loaded.
- **Via ProxyServer2's public IP:** website **now also loads** (since ProxyServer2 is now in `ProxySG`, which `AppServerSG` trusts).

> <img width="1662" height="141" alt="image" src="https://github.com/user-attachments/assets/e53a1be1-0b68-41da-b1f5-209c6d6d8859" />

> <img width="1657" height="144" alt="image" src="https://github.com/user-attachments/assets/7a42b03e-a1f0-4290-b707-f30abba5626e" />

> <img width="910" height="499" alt="image" src="https://github.com/user-attachments/assets/f0abd79d-84d2-4064-a628-a08e1ba5f2b2" />

**Analysis:** Referencing a security group as a source (instead of static IPs) means any current or future instance placed in `ProxySG` automatically gains access to `AppServer` — no manual IP maintenance required.

---

## 9. Task 6: Restricting HTTP Access by Using a Network ACL

### 9.1 Add a Deny Rule (Rule #99)

- In the VPC console, opened **Network ACLs** → selected the NACL associated with `LabVPC`.
- Edited inbound rules → added:
  - **Rule number:** 99
  - **Type:** HTTP
  - **Allow/Deny:** Deny
- Saved changes.

> <img width="1539" height="654" alt="image" src="https://github.com/user-attachments/assets/49d57f8e-c0a3-4a3b-bab7-426aefc65cd2" />

### 9.2 Re-test Access

- Loaded `ProxyServer1`'s public IP → connection **timed out**.

> <img width="1489" height="833" alt="image" src="https://github.com/user-attachments/assets/daebce71-5002-4570-ba84-e74cc3622136" />

**Analysis:** The NACL operates at the **subnet level**, in addition to security groups at the **instance/ENI level**. Even though the security groups still allow HTTP traffic, the NACL's explicit Deny blocks it — both layers must permit traffic for it to succeed.

### 9.3 Add an Allow Rule with a Lower Rule Number (Rule #98)

- Added a second inbound rule:
  - **Rule number:** 98
  - **Type:** HTTP
  - **Allow/Deny:** Allow
- Saved changes.

> <img width="1899" height="558" alt="image" src="https://github.com/user-attachments/assets/b8ba330f-d20e-477a-90a0-e6752d9b4ea1" />

### 9.4 Re-test Access

- Loaded `ProxyServer1`'s public IP again → website **loaded successfully**.

> <img width="1656" height="143" alt="image" src="https://github.com/user-attachments/assets/96860953-b395-44fc-95d2-ec6827a1a9c8" />

**Analysis:** NACL rules are evaluated **in order by rule number, lowest first**, and the **first matching rule wins** — rule #98 (Allow) is evaluated before rule #99 (Deny), so traffic is permitted despite the higher-numbered Deny rule still being present.

---

## 10. Task 7: Connecting to the AppServer by Using a Bastion Host and SSH

### 10.1 Repurpose ProxyServer2 as a Bastion Host

- Renamed `ProxyServer2` to **Bastion** (via the inline edit icon on the instance Name field).

> <img width="1550" height="243" alt="image" src="https://github.com/user-attachments/assets/a135a5a3-6f7f-4a11-86ac-cdb7677eb9b5" />

### 10.2 Create BastionSG

- Created a new security group:
  - **Name:** BastionSG
  - **Description:** BastionSG
  - **VPC:** LabVPC
  - **Inbound rule:** SSH (TCP 22) from **Anywhere-IPv4**

> <img width="1567" height="683" alt="image" src="https://github.com/user-attachments/assets/1feea8b6-b95a-4fc3-889f-750b1e70333c" />

### 10.3 Reassign Security Group on the Bastion Instance

- Selected the `Bastion` instance → **Actions > Security > Change security groups**.
- Removed `ProxySG`, added `BastionSG`, saved.

> <img width="1870" height="649" alt="image" src="https://github.com/user-attachments/assets/3b57b5a7-9436-470a-a3e1-c10f9f16d30c" />

### 10.4 Allow SSH from the Bastion in AppServerSG

- Retrieved `BastionPrivateIP` from the AWS Details panel.
- Edited `AppServerSG` inbound rules → added:
  - **Type:** SSH
  - **Source:** Custom → `BastionPrivateIP/32`
- Saved rules.

> <img width="1888" height="412" alt="image" src="https://github.com/user-attachments/assets/94e14e8d-f317-439a-8cb1-caa92a366cde" />

### 10.5 Connect via SSH Agent Forwarding

Ran the following from the lab terminal:

```bash
exec ssh-agent bash
ssh-add ~/.ssh/labsuser.pem
ssh -i ~/.ssh/labsuser.pem -A ec2-user@<BastionPublicIP>
```

> <img width="636" height="454" alt="image" src="https://github.com/user-attachments/assets/9bb4c6a7-e6f8-4461-82f1-5fcba38bbde1" />

From the bastion host, connected onward to the AppServer:

```bash
ssh ec2-user@<AppServerPrivateIP>
touch newfile.txt
```

> <img width="409" height="470" alt="image" src="https://github.com/user-attachments/assets/8e62e56d-3ac9-439e-9f6f-fb28d8248e7a" />

> <img width="403" height="143" alt="image" src="https://github.com/user-attachments/assets/310d0279-2c2a-4d72-9a3d-43d8b3e0311a" />

Disconnected from both hosts:

```bash
exit   # back to bastion
exit   # back to local Ubuntu terminal
```

> <img width="471" height="95" alt="image" src="https://github.com/user-attachments/assets/2a8fb475-3537-42ac-a388-f513fa9e7a98" />

**Analysis:** The `-A` flag forwarded the SSH agent (holding `labsuser.pem`) from the local machine through the bastion host to the AppServer — the private key itself was never copied onto the bastion. The connection succeeded because `AppServerSG` explicitly trusts SSH traffic sourced from the bastion's private IP.

---

## 11. Task 8: Connecting Directly to a Private-Subnet Host by Using Session Manager

- In the EC2 console, selected `AppServer` → **Connect** → **Session Manager** tab → **Connect**.
- A new browser tab opened with a direct shell session on the AppServer (no bastion, no open SSH port required).

> <img width="1865" height="534" alt="image" src="https://github.com/user-attachments/assets/dd78a7a2-ead7-4183-a177-1ba4e18c898c" />

> <img width="1920" height="187" alt="image" src="https://github.com/user-attachments/assets/1342dc58-e5ab-4c3d-bee8-13442b877644" />

- Modified the hosted website's HTML via the session:

```bash
sudo sed -i 's/instance!/instance! Session manager was used to edit this file./g' /var/www/html/index.html
```

> <img width="1920" height="210" alt="image" src="https://github.com/user-attachments/assets/0acc06d7-d618-4811-ba3d-a65e807888f7" />

- Re-tested the website via `ProxyServer1`'s public IP → page loaded and now displayed the updated text confirming the edit.

> <img width="1920" height="202" alt="image" src="https://github.com/user-attachments/assets/5729c069-300c-430b-b766-54123934af08" />

**Analysis:** Session Manager connects to the instance's SSM Agent over an outbound HTTPS connection it initiates — no inbound port 22 needs to be open, and the NACL's port-22 posture (untouched in this lab) is irrelevant. This avoids the operational overhead of maintaining a bastion host while still providing an auditable connection path (integrable with CloudTrail/CloudWatch).

---

## 12. Summary of Key Concepts

| Concept | Key Takeaway |
|---|---|
| Security groups | Stateful, instance/ENI-level firewalls; only support **Allow** rules. |
| Network ACLs | Stateless, subnet-level; support both **Allow** and **Deny**; evaluated in order by rule number (lowest first, first match wins). |
| SG referencing | A security group can be used as a *source* in another SG's rule, scaling access control without hardcoding IPs. |
| Public vs. private subnet | Determined by the subnet's route table — internet gateway (public) vs. NAT gateway (private). |
| Bastion host + SSH agent forwarding | Lets you reach a private-subnet instance without storing the private key on the intermediate host. |
| Session Manager | Eliminates the need for an open SSH port or a bastion host entirely, using an outbound connection to the SSM service. |

---
