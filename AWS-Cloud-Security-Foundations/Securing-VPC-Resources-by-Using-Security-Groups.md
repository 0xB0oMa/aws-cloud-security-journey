# Lab 4.1: Securing VPC Resources by Using Security Groups

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

### 2.2 Final Architecture (after all tasks)

> 📸 **Screenshot 2:** Final architecture diagram showing the bastion host, Session Manager connection path, and updated security group associations.

---

## 3. Accessing the AWS Management Console

- Started the lab and waited for the environment status to turn green.
- Opened the AWS Management Console by choosing the **AWS** link (not via a separate login URL, as in the IAM lab).

> 📸 **Screenshot 3:** Lab landing page with the green status indicator and AWS Details panel visible.

> 📸 **Screenshot 4:** AWS Management Console opened in a new tab.

---

## 4. Task 1: Analyzing the VPC and Private Subnet Resource Settings

### 4.1 VPCs and Subnets

- Opened the **Amazon VPC console**.
- Confirmed two VPCs exist: the default VPC (`172.31.0.0/16`) and `LabVPC` (`10.0.0.0/16`).
- Viewed **Subnets**, sorted by VPC column, and identified the three subnets in `LabVPC`: `PrivateSubnet`, `PublicSubnetA`, `PublicSubnetB`.

> 📸 **Screenshot 5:** "Your VPCs" list showing both VPCs and their CIDR ranges.

> 📸 **Screenshot 6:** Subnets list sorted by VPC, showing the three LabVPC subnets.

### 4.2 PrivateSubnet Routing

- Selected `PrivateSubnet` → **Route table** tab → opened the associated route table.
- Observed routes:
  - `10.0.0.0/16` → **local**
  - `0.0.0.0/0` → **NAT gateway**
- Renamed the route table from `changeme` to **Private** via the Tags tab.
- Opened the NAT gateway details and confirmed it runs in `PublicSubnetA`.

> 📸 **Screenshot 7:** Route table routes tab showing local + NAT gateway routes for PrivateSubnet.

> 📸 **Screenshot 8:** Tags tab — renaming the route table to "Private."

> 📸 **Screenshot 9:** NAT gateway details page confirming it resides in PublicSubnetA.

**Analysis:** Because the private subnet routes outbound traffic through a NAT gateway (not directly to an internet gateway), instances there can initiate outbound connections to the internet, but they are not directly reachable from the internet and have no public IP.

### 4.3 AppServer Instance & Security Group

- Opened the **Amazon EC2 console** (in a new tab) → **Instances** → selected `AppServer`.
- Confirmed: running in `PrivateSubnet`, has a private IPv4 address only (no public IP).
- Opened the **Security** tab → followed the link to `AppServerSG`.
- **Inbound rules:** HTTP (TCP 80) from source `0.0.0.0/0` (anywhere).
- **Outbound rules:** All traffic allowed (default).

> 📸 **Screenshot 10:** AppServer "Details" tab showing private-only IP addressing.

> 📸 **Screenshot 11:** AppServerSG inbound rules showing HTTP/80 open to 0.0.0.0/0.

> 📸 **Screenshot 12:** AppServerSG outbound rules showing all traffic allowed.

**Key concept noted:** Security groups are **stateful** — return traffic for an allowed outbound/inbound request is automatically permitted regardless of the rules in the other direction.

---

## 5. Task 2: Analyzing the Public Subnet Resource Settings

### 5.1 PublicSubnetA Routing

- Selected `PublicSubnetA` → **Route table** tab.
- Confirmed `0.0.0.0/0` routes to an **internet gateway** (not a NAT gateway) — this is what makes it a "public" subnet.
- Verified the internet gateway is associated with `LabVPC`.

> 📸 **Screenshot 13:** PublicSubnetA route table showing the route to the internet gateway.

> 📸 **Screenshot 14:** Internet gateway details page.

### 5.2 ProxyServer1 (PublicSubnetA)

- Selected `ProxyServer1` in the EC2 console.
- Confirmed it has a **public IPv4 address** (unlike AppServer).
- Security tab → security group `ProxySG` → inbound rule allows HTTP/80 from anywhere.

> 📸 **Screenshot 15:** ProxyServer1 details tab showing its public IP.

> 📸 **Screenshot 16:** ProxySG inbound rules showing HTTP/80 open to all sources.

### 5.3 ProxyServer2 (PublicSubnetB) — Add Inbound Rule

- Selected `ProxyServer2`, confirmed it runs in `PublicSubnetB` with a public IP, and is associated with `ProxySG2`.
- Edited `ProxySG2` inbound rules → added a new rule: **Type = HTTP**, **Source = Anywhere-IPv4** → saved.

> 📸 **Screenshot 17:** ProxyServer2 details tab confirming subnet and public IP.

> 📸 **Screenshot 18:** "Edit inbound rules" dialog for ProxySG2 with the new HTTP rule added before saving.

> 📸 **Screenshot 19:** ProxySG2 inbound rules after saving, showing HTTP/80 now allowed.

---

## 6. Task 3: Testing HTTP Connectivity from Public EC2 Instances

- Copied `ProxyServer1`'s **public IPv4 address** and loaded it in a new browser tab → the AppServer's website loaded successfully (forwarded via the proxy).
- Repeated the same test with `ProxyServer2`'s public IP → website also loaded successfully.
- Closed both tabs.

> 📸 **Screenshot 20:** Website loaded successfully via ProxyServer1's public IP.

> 📸 **Screenshot 21:** Website loaded successfully via ProxyServer2's public IP.

**Analysis:** At this point, both `ProxySG` and `ProxySG2` allow inbound HTTP from anywhere, and `AppServerSG` allows inbound HTTP from anywhere — so both proxy paths to the AppServer succeed.

---

## 7. Task 4: Restricting HTTP Access by Using an IP Address

- Retrieved `ProxyServer1PrivateIP` from the **AWS Details** panel.
- Edited `AppServerSG` inbound rules:
  - Removed the existing `0.0.0.0/0` source on the HTTP rule.
  - Added `ProxyServer1PrivateIP/32` as the new source.
  - Saved rules.

> 📸 **Screenshot 22:** AWS Details panel showing the ProxyServer1PrivateIP value (can redact other fields).

> 📸 **Screenshot 23:** AppServerSG "Edit inbound rules" showing the new restricted source (ProxyServer1's private IP /32).

### 7.1 Re-test Access

- **Via ProxyServer1's public IP:** website loaded successfully (source IP matches the new rule).
- **Via ProxyServer2's public IP:** connection **timed out** (source IP does not match the rule).

> 📸 **Screenshot 24:** Successful page load via ProxyServer1 after the restriction.

> 📸 **Screenshot 25:** Connection timeout / failure when accessing via ProxyServer2.

**Analysis:** `AppServerSG` now only accepts HTTP traffic whose source IP is `ProxyServer1`'s private IP, so traffic forwarded from `ProxyServer2` is rejected even though `ProxySG2` itself still allows the inbound connection from the internet to ProxyServer2.

---

## 8. Task 5: Scaling Restricted HTTP Access by Referencing a Security Group

Goal: Avoid hardcoding individual IPs by referencing a **security group** as the allowed source instead.

### 8.1 Update AppServerSG to Reference ProxySG

- Edited `AppServerSG` inbound rules:
  - Deleted the IP-based rule from Task 4.
  - Added a new rule: **Type = HTTP**, **Source = Custom**, typed `sg` in the source field, and selected the `ProxySG` security group from the dropdown.
  - Saved rules.

> 📸 **Screenshot 26:** AppServerSG inbound rule configured with ProxySG referenced as the source.

### 8.2 Reassign ProxyServer2 to ProxySG

- Selected `ProxyServer2` → **Actions > Security > Change security groups**.
- Removed `ProxySG2`, added `ProxySG`, saved.

> 📸 **Screenshot 27:** "Change security groups" dialog for ProxyServer2 — ProxySG2 removed, ProxySG added.

### 8.3 Re-test Access

- **Via ProxyServer1's public IP:** website loaded.
- **Via ProxyServer2's public IP:** website **now also loads** (since ProxyServer2 is now in `ProxySG`, which `AppServerSG` trusts).

> 📸 **Screenshot 28:** Successful page load via ProxyServer1.

> 📸 **Screenshot 29:** Successful page load via ProxyServer2 after the security group reassignment.

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

> 📸 **Screenshot 30:** Network ACL inbound rules showing the new Deny rule (#99) for HTTP.

### 9.2 Re-test Access

- Loaded `ProxyServer1`'s public IP → connection **timed out**.

> 📸 **Screenshot 31:** Connection timeout after the NACL deny rule was added.

**Analysis:** The NACL operates at the **subnet level**, in addition to security groups at the **instance/ENI level**. Even though the security groups still allow HTTP traffic, the NACL's explicit Deny blocks it — both layers must permit traffic for it to succeed.

### 9.3 Add an Allow Rule with a Lower Rule Number (Rule #98)

- Added a second inbound rule:
  - **Rule number:** 98
  - **Type:** HTTP
  - **Allow/Deny:** Allow
- Saved changes.

> 📸 **Screenshot 32:** Network ACL inbound rules showing both rule #98 (Allow) and #99 (Deny).

### 9.4 Re-test Access

- Loaded `ProxyServer1`'s public IP again → website **loaded successfully**.

> 📸 **Screenshot 33:** Successful page load after adding the lower-numbered Allow rule.

**Analysis:** NACL rules are evaluated **in order by rule number, lowest first**, and the **first matching rule wins** — rule #98 (Allow) is evaluated before rule #99 (Deny), so traffic is permitted despite the higher-numbered Deny rule still being present.

---

## 10. Task 7: Connecting to the AppServer by Using a Bastion Host and SSH

### 10.1 Repurpose ProxyServer2 as a Bastion Host

- Renamed `ProxyServer2` to **Bastion** (via the inline edit icon on the instance Name field).

> 📸 **Screenshot 34:** Instance renamed to "Bastion" in the EC2 console.

### 10.2 Create BastionSG

- Created a new security group:
  - **Name:** BastionSG
  - **Description:** BastionSG
  - **VPC:** LabVPC
  - **Inbound rule:** SSH (TCP 22) from **Anywhere-IPv4**

> 📸 **Screenshot 35:** "Create security group" form for BastionSG with the SSH inbound rule.

### 10.3 Reassign Security Group on the Bastion Instance

- Selected the `Bastion` instance → **Actions > Security > Change security groups**.
- Removed `ProxySG`, added `BastionSG`, saved.

> 📸 **Screenshot 36:** "Change security groups" dialog — ProxySG removed, BastionSG added for the Bastion instance.

### 10.4 Allow SSH from the Bastion in AppServerSG

- Retrieved `BastionPrivateIP` from the AWS Details panel.
- Edited `AppServerSG` inbound rules → added:
  - **Type:** SSH
  - **Source:** Custom → `BastionPrivateIP/32`
- Saved rules.

> 📸 **Screenshot 37:** AppServerSG inbound rules showing the new SSH rule scoped to the Bastion's private IP.

### 10.5 Connect via SSH Agent Forwarding

Ran the following from the lab terminal:

```bash
exec ssh-agent bash
ssh-add ~/.ssh/labsuser.pem
ssh -i ~/.ssh/labsuser.pem -A ec2-user@<BastionPublicIP>
```

> 📸 **Screenshot 38:** Terminal showing successful SSH connection to the bastion host (`[ec2-user@bastion]` prompt).

From the bastion host, connected onward to the AppServer:

```bash
ssh ec2-user@<AppServerPrivateIP>
touch newfile.txt
```

> 📸 **Screenshot 39:** Terminal showing the prompt change to `[ec2-user@appserver]` after connecting.

> 📸 **Screenshot 40:** Terminal confirming `newfile.txt` was created on the AppServer (e.g., via `ls`).

Disconnected from both hosts:

```bash
exit   # back to bastion
exit   # back to local Ubuntu terminal
```

> 📸 **Screenshot 41:** Terminal showing the prompt returning to the local Ubuntu shell after both `exit` commands.

**Analysis:** The `-A` flag forwarded the SSH agent (holding `labsuser.pem`) from the local machine through the bastion host to the AppServer — the private key itself was never copied onto the bastion. The connection succeeded because `AppServerSG` explicitly trusts SSH traffic sourced from the bastion's private IP.

---

## 11. Task 8: Connecting Directly to a Private-Subnet Host by Using Session Manager

- In the EC2 console, selected `AppServer` → **Connect** → **Session Manager** tab → **Connect**.
- A new browser tab opened with a direct shell session on the AppServer (no bastion, no open SSH port required).

> 📸 **Screenshot 42:** Session Manager "Connect" tab before connecting.

> 📸 **Screenshot 43:** Active Session Manager terminal session connected to the AppServer.

- Modified the hosted website's HTML via the session:

```bash
sudo sed -i 's/instance!/instance! Session manager was used to edit this file./g' /var/www/html/index.html
```

> 📸 **Screenshot 44:** Terminal showing the `sed` command executed successfully in the Session Manager session.

- Re-tested the website via `ProxyServer1`'s public IP → page loaded and now displayed the updated text confirming the edit.

> 📸 **Screenshot 45:** Browser showing the updated webpage text ("Session manager was used to edit this file.").

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
