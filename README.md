# AWS Hybrid Storage — S3 + Active Directory Integration

A **hybrid file server** implementation combining **Amazon S3** as the storage backend with **Windows Server (AD DS)** for login authentication, bridged by **AWS Storage Gateway (File Gateway)**, and accessible from on-premise/local computers via **Site-to-Site VPN**.

![AWS Architecture Diagram](./image/s3-hybrid-file-servers.drawio.svg)

---

## 1. Background & Objective

The initial requirement for this project was to build a **file server** accessible both from the cloud and from a local (on-premise) network, with Active Directory-integrated authentication — replicating the experience of a traditional office SMB file share.

The first option evaluated was **Amazon FSx for Windows File Server**, since it natively supports SMB and AD integration without requiring additional components. However, after cost analysis, **FSx turned out to be significantly more expensive** for this use case, because:

- FSx charges for **provisioned storage** (not actual storage used), with a minimum capacity requirement per file system.
- **Throughput capacity** is billed separately and increases significantly as higher performance is needed.
- The **Multi-AZ** scheme (for HA) adds a substantial cost premium.
- It's well suited for enterprise workloads with high, consistent I/O, but becomes cost-inefficient for file storage use cases with low-to-moderate access patterns.

Since this project's needs lean more toward **archival/general-purpose file storage with low-to-medium access patterns** rather than I/O-intensive workloads, an **S3 + Storage Gateway** approach was chosen instead, based on the following considerations:

- **S3** bills based on actual storage consumed (pay-as-you-go), is significantly cheaper per GB than FSx, and can be further optimized with **lifecycle policies** (Standard → IA → Glacier).
- **Storage Gateway (File Gateway)** provides a local cache (EBS) for fast access to frequently used data, while "cold" data remains cheaply stored in S3.
- Login authentication can still be integrated through **Windows Server AD DS**, joined by Storage Gateway (SMB security: Active Directory authentication), preserving the familiar AD-based SMB file share experience for users.

In short: **this architecture was chosen purely for cost optimization reasons**, at the trade-off of slightly more complexity (requiring a Storage Gateway appliance + EBS cache) compared to FSx, which is plug-and-play but expensive.

---

## 2. Architecture

### Core Components

| Component | Function |
|---|---|
| **Windows Server (AD DS)** | Domain controller — handles user login authentication for the file share |
| **Storage Gateway (File Gateway)** | Data plane appliance — exposes the SMB share, maintains local cache, syncs to S3 |
| **EBS 500 GB (Cache)** | Local cache volume for Storage Gateway, holding frequently accessed data |
| **S3 Bucket** | Primary storage backend — durably and cheaply stores all file share data |
| **Storage Gateway Service (AWS managed)** | AWS control plane — manages activation, configuration, and orchestration of the appliance to S3 |
| **NAT Gateway** | Outbound internet access for Storage Gateway/Windows Server (updates, gateway activation) |
| **Virtual Private Gateway (VGW)** | VPN endpoint on the AWS side, where traffic from on-premise enters the VPC |
| **Site-to-Site VPN + Customer Gateway** | Connects the local network to the VPC privately, allowing the SMB share to be mounted from local computers |

### Traffic Flow

**Authentication & file access flow (from local computer):**
```
Local Computer → Site-to-Site VPN → Customer Gateway → Virtual Private Gateway
→ Route Table (private subnet) → Storage Gateway (SMB 445) ↔ Windows Server AD DS (authentication)
```

**Data flow to S3 (behind the scenes):**
```
Storage Gateway → Elastic Network Interface → Storage Gateway Service (control plane)
→ S3 Bucket (data plane, actual object storage)
```

**Outbound internet flow (separate from VPN):**
```
Storage Gateway / Windows Server → Private Subnet route table → NAT Gateway (public subnet) → Internet Gateway → Internet
```

> Note: NAT Gateway is **not** traversed by VPN traffic. VPN traffic enters through the VGW, which is attached directly to the VPC. NAT Gateway is used solely for outbound internet access (gateway activation, patching, etc.).

---

## 3. Cost Estimate

> The figures below are a **template** — adjust them to match your project's actual region, capacity, and usage patterns. For the most accurate and up-to-date numbers, use the [AWS Pricing Calculator](https://calculator.aws/).

### 3.1 Option Comparison: FSx vs S3 + Storage Gateway

| Item | Amazon FSx for Windows File Server | S3 + Storage Gateway (chosen) |
|---|---|---|
| Pricing model | Provisioned storage + throughput capacity (billed even if not fully used) | Pay-as-you-go based on actual storage used |
| Minimum storage | Minimum capacity required per file system | No minimum, granular per object |
| Throughput cost | Billed separately, increases quickly for higher performance | Determined by the Storage Gateway instance/appliance + EBS cache |
| Multi-AZ HA | Significant additional premium | S3 HA is built-in (11 nines durability); appliance HA requires extra effort |
| Cheap lifecycle/tiering | No automatic tiering to cheaper storage | Available — S3 Lifecycle (Standard → IA → Glacier) |
| Best suited for | High, consistent I/O workloads needing native SMB performance without a cache layer | Low-to-medium access workloads, cost efficiency as priority |

### 3.2 Monthly Cost Breakdown Template (Region: _____________)

| Component | Specification | Unit Cost (USD) | Qty/Usage | Estimated Monthly Cost (USD) |
|---|---|---|---|---|
| Windows Server EC2 (AD DS) | Instance type: _______ | | 730 hrs | |
| EBS Cache Volume (gp3) | 500 GB | | 500 GB | |
| Storage Gateway (File Gateway) | Appliance (EC2-based) | | 730 hrs | |
| S3 Standard Storage | Estimated total data | | ___ GB | |
| S3 Lifecycle (IA/Glacier) | Cold storage data | | ___ GB | |
| S3 Requests (PUT/GET) | Estimated requests/month | | | |
| NAT Gateway | Per hour + per GB processed | | 730 hrs + ___ GB | |
| Site-to-Site VPN | Per connection/hour | | 730 hrs | |
| Data Transfer Out | Estimated outbound transfer | | ___ GB | |
| Virtual Private Gateway | (usually included in VPN cost) | | | |
| **Total Estimated/Month** | | | | **$______** |

### 3.3 Comparison Reference (optional, to include as a README comparison)

| Scenario | Estimated Monthly Cost |
|---|---|
| FSx for Windows File Server (Single-AZ, equivalent capacity) | *fill in after checking AWS Pricing Calculator* |
| FSx for Windows File Server (Multi-AZ) | *fill in after checking AWS Pricing Calculator* |
| S3 + Storage Gateway (this architecture) | *fill in from table 3.2 above* |

> Fill in the figures above using the AWS Pricing Calculator based on your project's actual region and estimated capacity. If needed, I can help calculate a more detailed estimate once you provide the instance type, data capacity, and region used.

---

## 4. Production Considerations (Known Limitations)

This architecture was built for **portfolio/demo purposes with a focus on cost optimization**. For a stricter production deployment, the following should be considered further:

- **High Availability**: Windows Server AD DS and Storage Gateway are currently single instances each (single point of failure). For production, ideally deploy at least 2 Domain Controllers across 2 different AZs, or migrate to **AWS Managed Microsoft AD**.
- **Security**: Security Group/NACL rules should be restricted to allow only SMB (445), Kerberos (88), and LDAP (389) from the on-premise CIDR range — not broadly open.
- **Encryption**: enable EBS volume encryption and S3 server-side encryption (SSE-S3/SSE-KMS).
- **VPC Endpoint for Storage Gateway**: to keep appliance ↔ Storage Gateway Service traffic from traversing the NAT Gateway/internet, use an Interface VPC Endpoint (`com.amazonaws.<region>.storagegateway`) — more secure and reduces NAT Gateway data transfer costs.
- **Monitoring**: add CloudWatch alarms for Storage Gateway cache hit ratio & upload buffer, plus CloudTrail for S3 access auditing.
- **Backup & DR**: ensure a backup strategy is in place for the EBS cache volume and S3 versioning/replication according to data retention requirements.

---

## 5. Tech Stack

- Amazon S3
- AWS Storage Gateway (File Gateway)
- Amazon EC2 (Windows Server, AD DS)
- Amazon EBS (gp3)
- Amazon VPC (Public/Private Subnet, NAT Gateway)
- Site-to-Site VPN, Virtual Private Gateway, Customer Gateway

---

## 6. Roadmap / Next Steps

- [ ] Implement VPC Endpoint for Storage Gateway
- [ ] Add S3 Lifecycle Policy
- [ ] Set up Multi-AZ / redundant Domain Controller
- [ ] Add CloudWatch monitoring + dashboard
- [ ] Step-by-step deployment documentation (Terraform/CloudFormation)
