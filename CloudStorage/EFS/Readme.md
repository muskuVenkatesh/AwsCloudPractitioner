# Amazon EFS (Elastic File System) — Comprehensive Master Guide

A complete, production-grade reference manual covering architecture, storage classes, networking, multi-instance mounting, security, encryption, access points, performance optimization, troubleshooting, and interview preparation.

---

## 📑 Table of Contents
1. [Introduction to Amazon EFS](#1-introduction-to-amazon-efs)
   - [What is Amazon EFS?](#what-is-amazon-efs)
   - [Why Use Amazon EFS?](#why-use-amazon-efs)
   - [Key Features](#key-features)
   - [Storage Comparison: EFS vs. EBS vs. S3](#storage-comparison-efs-vs-ebs-vs-s3)
2. [EFS Architecture & Multi-AZ Mechanics](#2-efs-architecture--multi-az-mechanics)
   - [Regional vs. One Zone File Systems](#regional-vs-one-zone-file-systems)
   - [Availability Zones & Mount Targets](#availability-zones--mount-targets)
   - [Multi-Instance Shared Access Architecture](#multi-instance-shared-access-architecture)
3. [EFS Storage Classes & Lifecycle Management](#3-efs-storage-classes--lifecycle-management)
   - [Storage Classes Overview (Standard, IA, Archive)](#storage-classes-overview)
   - [EFS Lifecycle Management & Auto-Tiering](#efs-lifecycle-management--auto-tiering)
   - [Storage Class Cost & Performance Comparison](#storage-class-cost--performance-comparison)
4. [Creating an EFS File System (Console & CLI)](#4-creating-an-efs-file-system-console--cli)
   - [AWS Management Console Step-by-Step](#aws-management-console-step-by-step)
   - [AWS CLI Creation Commands](#aws-cli-creation-commands)
5. [EFS Networking, Mount Targets & Security Groups](#5-efs-networking-mount-targets--security-groups)
   - [VPC & Subnet Requirements](#vpc--subnet-requirements)
   - [Security Group Design: The Gold Standard](#security-group-design-the-gold-standard)
   - [NFS Protocol & Port 2049](#nfs-protocol--port-2049)
   - [DNS Resolution Mechanics](#dns-resolution-mechanics)
6. [Attaching and Mounting EFS on an EC2 Linux Instance](#6-attaching-and-mounting-efs-on-an-ec2-linux-instance)
   - [Prerequisites (amazon-efs-utils vs nfs-utils)](#prerequisites)
   - [Mounting via EFS Mount Helper (`mount -t efs`)](#mounting-via-efs-mount-helper)
   - [Mounting via Standard NFS4 Client (`mount -t nfs4`)](#mounting-via-standard-nfs4-client)
   - [Verifying the Mounted File System](#verifying-the-mounted-file-system)
7. [Mounting EFS on Multiple EC2 Instances](#7-mounting-efs-on-multiple-ec2-instances)
   - [Real-World Multi-Instance Setup Demonstration](#real-world-multi-instance-setup-demonstration)
   - [Distributed File Locking & Concurrency](#distributed-file-locking--concurrency)
8. [Automatic Mounting After Reboot (`/etc/fstab`)](#8-automatic-mounting-after-reboot-etcfstab)
   - [Configuring `/etc/fstab` with EFS Mount Helper](#configuring-etcfstab-with-efs-mount-helper)
   - [Configuring `/etc/fstab` with Standard NFS4](#configuring-etcfstab-with-standard-nfs4)
   - [The Critical `_netdev` and `nofail` Mount Options](#the-critical-_netdev-and-nofail-mount-options)
9. [EFS Encryption Architecture](#9-efs-encryption-architecture)
   - [Encryption at Rest (AWS KMS)](#encryption-at-rest-aws-kms)
   - [Encryption in Transit (TLS via Stunnel)](#encryption-in-transit-tls-via-stunnel)
   - [Enforcing In-Transit Encryption via File System Policy](#enforcing-in-transit-encryption-via-file-system-policy)
10. [EFS Access Points & IAM Authorization](#10-efs-access-points--iam-authorization)
    - [What are EFS Access Points?](#what-are-efs-access-points)
    - [Enforcing POSIX Identities (UID/GID)](#enforcing-posix-identities-uidgid)
    - [Root Directory Jailing & Automated Provisioning](#root-directory-jailing--automated-provisioning)
    - [IAM Client Authorization Policies](#iam-client-authorization-policies)
11. [File Permissions, Ownership & POSIX Semantics](#11-file-permissions-ownership--posix-semantics)
    - [Linux Permissions & Directory Ownership](#linux-permissions--directory-ownership)
    - [Root Squashing Explained](#root-squashing-explained)
    - [Remediating Permission Issues](#remediating-permission-issues)
12. [Monitoring, Throughput & Performance Modes](#12-monitoring-throughput--performance-modes)
    - [Throughput Modes (Elastic vs Provisioned)](#throughput-modes-elastic-vs-provisioned)
    - [Performance Modes (General Purpose vs Max I/O)](#performance-modes-general-purpose-vs-max-io)
    - [Key Amazon CloudWatch Metrics](#key-amazon-cloudwatch-metrics)
13. [Unmounting EFS Safely](#13-unmounting-efs-safely)
    - [Safe Unmount Commands](#safe-unmount-commands)
    - [Resolving "Device or Resource Busy" Errors](#resolving-device-or-resource-busy-errors)
14. [Backup, Disaster Recovery & Replication](#14-backup-disaster-recovery--replication)
    - [Centralized Backups with AWS Backup](#centralized-backups-with-aws-backup)
    - [EFS Native Cross-Region Replication](#efs-native-cross-region-replication)
15. [Diagnostic Troubleshooting Matrix](#15-diagnostic-troubleshooting-matrix)
    - [Common Connection, DNS, and Mount Failures](#common-connection-dns-and-mount-failures)
16. [Best Practices & Interview Preparation](#16-best-practices--interview-preparation)
    - [Architecture & Cost Optimization Best Practices](#architecture--cost-optimization-best-practices)
    - [High-Yield Technical Interview Q&A](#high-yield-technical-interview-qa)
    - [Quick Command Cheat Sheet](#quick-command-cheat-sheet)

---

## 1. Introduction to Amazon EFS

### What is Amazon EFS?
**Amazon Elastic File System (Amazon EFS)** is a fully managed, serverless, elastic, cloud-native file storage service built on the **Network File System version 4 (NFSv4.0 and NFSv4.1)** protocol.

It provides a standard POSIX-compliant filesystem that can be mounted concurrently by **thousands of compute instances**—including Amazon EC2, AWS Lambda, Amazon ECS, Amazon EKS, AWS Fargate, and on-premises physical servers via AWS Direct Connect or AWS VPN.

```
+-------------------------------------------------------------------------------------------------+
|                                           Amazon EFS                                            |
|                              (Fully Managed Serverless Shared Storage)                          |
+-------------------------------------------------------------------------------------------------+
               ^                                   ^                                   ^
               | NFSv4 (Port 2049)                 | NFSv4 (Port 2049)                 | NFSv4 (Port 2049)
               |                                   |                                   |
+--------------+-------------+     +---------------+-------------+     +---------------+-------------+
|        EC2 Instance A      |     |        EC2 Instance B       |     |       ECS / EKS Container   |
|     (Availability Zone 1)  |     |     (Availability Zone 2)   |     |     (Availability Zone 3)   |
|   Mount: /mnt/shared_data  |     |   Mount: /mnt/shared_data   |     |    Mount: /var/www/uploads  |
+----------------------------+     +-----------------------------+     +-----------------------------+
```

### Why Use Amazon EFS?
1. **Concurrent Shared Storage:** Unlike standard EBS volumes that are bound to a single EC2 instance in a single AZ, EFS allows hundreds to thousands of clients to read and write to the same files simultaneously across multiple Availability Zones.
2. **Serverless & Fully Elastic:** There is no storage provisioning. You do not define volume sizes upfront. EFS scales on demand automatically up to petabytes as you add files, and shrinks automatically as you delete files. You pay only for what you store.
3. **High Availability & Regional Durability:** By default, Regional EFS file systems synchronously mirror data across multiple physically separated Availability Zones (minimum 3 AZs) within an AWS Region, delivering **99.999999999% (11 9s)** of data durability.
4. **POSIX Compliance:** Supports standard Linux file operations, file locks (`flock`, `fcntl`), permissions (`chmod`, `chown`), symbolic links, and nested directory trees.
5. **Native Container & Serverless Integration:** Mounts directly into ECS tasks, Kubernetes (EKS) Pods via the EFS CSI driver, and AWS Lambda functions for persistent stateful workloads.

### Key Features
* **Storage Protocol:** NFSv4.0 and NFSv4.1 over TCP port 2049.
* **Elasticity:** Zero pre-provisioning; seamless expansion and contraction.
* **Storage Tiers:** Automatic cost reduction using Lifecycle Management to transition cold files to Infrequent Access (IA) and Archive tiers.
* **Access Control:** Layered security using VPC Security Groups, IAM authorization policies, and EFS Access Points.
* **Encryption:** Native AES-256 encryption at rest (KMS) and Transport Layer Security (TLS) in transit.

---

### Storage Comparison: EFS vs. EBS vs. S3

| Feature | Amazon EFS | Amazon EBS | Amazon S3 |
| :--- | :--- | :--- | :--- |
| **Storage Type** | **File Storage** (NFSv4.1) | **Block Storage** (Raw disk / volume) | **Object Storage** (REST API) |
| **Data Structure** | Hierarchical files & folders | Raw unformatted block sectors | Flat bucket with object keys & metadata |
| **Access Model** | **Concurrent multi-instance read/write** across multiple AZs | Single instance (or Multi-Attach `io1`/`io2` in single AZ) | Distributed internet-scale HTTP REST API |
| **Scope / Availability** | **Regional (Multi-AZ)** or One Zone | **Single Availability Zone** (AZ-locked) | **Regional / Global** namespace |
| **Capacity Sizing** | **Elastic** (Automatic auto-scaling, no pre-allocation) | **Pre-allocated** volume capacity (e.g., 50 GiB) | **Virtually unlimited** (Auto-scales per object) |
| **Latency Profile** | Low single-digit ms (NFS overhead) | **Sub-millisecond to low single-digit ms** | 100–200 milliseconds initial time-to-first-byte |
| **Throughput / IOPS** | Scales up to 10+ GB/s and 500k+ IOPS | Up to 256,000 IOPS and 4,000 MB/s (`io2`) | Virtually unlimited horizontal scalability |
| **Ideal Workloads** | Shared web farms (WordPress), CMS, dev tools (Git/Jenkins), shared analytics | Boot volumes, transactional databases (PostgreSQL, Oracle, MySQL) | Media streaming, big data lakes, static assets, long-term backups |

---

## 2. EFS Architecture & Multi-AZ Mechanics

### Regional vs. One Zone File Systems

AWS offers two architectural deployment models for EFS:

```
+---------------------------------------------------------------------------------------------------------+
|                                    AWS Region (e.g., ap-south-1)                                        |
|                                                                                                         |
|   +--------------------------+  +--------------------------+  +--------------------------+              |
|   |   Availability Zone 1a   |  |   Availability Zone 1b   |  |   Availability Zone 1c   |              |
|   |                          |  |                          |  |                          |              |
|   |  [Mount Target: 10.0.1.5]|  |  [Mount Target: 10.0.2.8]|  |  [Mount Target: 10.0.3.9]|              |
|   |             ^            |  |             ^            |  |             ^            |              |
|   |             |            |  |             |            |  |             |            |              |
|   |      [EC2 Instance]      |  |      [EC2 Instance]      |  |      [EKS Container]     |              |
|   +-------------+------------+  +-------------+------------+  +-------------+------------+              |
|                 |                             |                             |                           |
|                 +=============================+=============================+                           |
|                                               |                                                         |
|                                  Synchronous Regional Storage                                           |
|                            (Mirroring Across Minimum 3 AZs in Region)                                   |
|                                99.999999999% (11 9s) Durability                                         |
+---------------------------------------------------------------------------------------------------------+
```

| Dimension | Regional File System (Recommended Default) | One Zone File System |
| :--- | :--- | :--- |
| **Availability & Durability** | Stored redundantly across **minimum 3 AZs**; 99.999999999% (11 9s) durability; 99.99% availability SLA. | Stored redundantly within a **single AZ**; 99.999999999% (11 9s) durability within that AZ; 99.9% availability SLA. |
| **Disaster Resilience** | Resilient against total loss of an entire Availability Zone. | Data unrecoverable if that specific AZ suffers physical destruction. |
| **Mount Targets** | One Mount Target per Availability Zone in the VPC. | Exactly **one** Mount Target in the chosen AZ. |
| **Cost Profile** | Standard pricing. | **47% cheaper** base storage cost compared to Regional. |
| **Primary Use Cases** | Production applications, enterprise web fleets, shared file shares. | Dev/test environments, build artifacts, secondary batch processing. |

---

### Availability Zones & Mount Targets

To access an EFS file system from an EC2 instance, you connect through a **Mount Target**:
* A Mount Target is an **Elastic Network Interface (ENI)** created by AWS inside a specific subnet of your VPC.
* It receives a private IPv4 address from that subnet's CIDR block.
* In a Regional deployment, you provision **one Mount Target per Availability Zone**.
* When an EC2 instance in `ap-south-1a` connects to EFS, it automatically talks to the Mount Target located in `ap-south-1a`, keeping network traffic localized, fast, and free of cross-AZ data transfer fees.

---

### Multi-Instance Shared Access Architecture

```
                  +----------------------------------------------------+
                  |               Amazon EFS File System               |
                  |                (fs-0123456789abcdef0)              |
                  +----------------------------------------------------+
                                    |                |
             +----------------------+                +----------------------+
             |                                                              |
             v                                                              v
   [Mount Target in AZ-1a]                                        [Mount Target in AZ-1b]
    (Private IP: 10.0.1.25)                                        (Private IP: 10.0.2.40)
             ^                                                              ^
             | Port 2049                                                    | Port 2049
             |                                                              |
    +--------+--------+                                            +--------+--------+
    |  EC2 Instance 1 |                                            |  EC2 Instance 2 |
    |   (Web Server)  |                                            |   (API Server)  |
    | Subnet: 10.0.1.0|                                            | Subnet: 10.0.2.0|
    +-----------------+                                            +-----------------+
```

---

## 3. EFS Storage Classes & Lifecycle Management

EFS optimizes storage spend by providing automatic tiering across active, infrequent, and archive access patterns.

### Storage Classes Overview

```
Active Data (High IOPS)              Infrequently Accessed (>30 Days)           Rarely Accessed (>90 Days)
+-----------------------+              +------------------------------+           +------------------------+
|     EFS Standard      |  ==[DLM]==>  |  EFS Infrequent Access (IA)  | ==[DLM]==>|      EFS Archive       |
|  $0.30 / GB-month     |              |  $0.025 / GB-month           |           |  $0.011 / GB-month     |
| (Lowest Read Latency) |              | (92% Cost Reduction + Read $) |           | (Lowest Multi-AZ Cost) |
+-----------------------+              +------------------------------+           +------------------------+
            ^                                         |
            +============[On First Access]============+
```

1. **EFS Standard:**
   * Designed for active, frequently accessed data requiring the lowest single-digit millisecond read/write latency.
   * Baseline cost is ~$0.30 per GB-month (pricing varies slightly by region).
2. **EFS Infrequent Access (EFS IA):**
   * Designed for files not accessed in the last 7 to 90 days.
   * Storage cost is **~92% lower** (~$0.025 per GB-month), plus a small access fee per GB read/written.
3. **EFS Archive:**
   * Optimized for cold data accessed only a few times per year.
   * Delivers the lowest cost storage for long-term historical records (~$0.011 per GB-month).
4. **EFS One Zone & One Zone-IA:**
   * Single-AZ equivalents designed to lower costs further for non-critical workloads.

---

### EFS Lifecycle Management & Auto-Tiering

Lifecycle management policies automate moving files between storage classes based on access timestamps (`atime`):

* **Transition into IA / Archive:**
  * Options: `1`, `7`, `14`, `30`, `60`, or `90 days` since last access.
  * *Example:* If set to `30 days`, any file unread for 30 consecutive days is silently moved to the IA tier.
* **Transition out of IA (Back to Standard):**
  * Option: **On first access**.
  * Prevents files that suddenly become hot from incurring recurring IA data retrieval charges. When a client reads the file, EFS moves it back to Standard storage.

---

### Storage Class Cost & Performance Comparison

| Attribute | EFS Standard | EFS Infrequent Access (IA) | EFS Archive |
| :--- | :--- | :--- | :--- |
| **Typical Storage Cost** | ~$0.30 / GB-month | ~$0.025 / GB-month | ~$0.011 / GB-month |
| **Access / Retrieval Fee** | **$0.00** (Free) | ~$0.01 per GB transferred | ~$0.03 per GB transferred |
| **First-Byte Latency** | Low single-digit ms | Low double-digit ms | Tens of milliseconds |
| **Durability** | 99.999999999% (11 9s) | 99.999999999% (11 9s) | 99.999999999% (11 9s) |
| **Minimum Billable Size** | Actual file size | 128 KiB per file | 128 KiB per file |

---

## 4. Creating an EFS File System (Console & CLI)

### AWS Management Console Step-by-Step

```
Navigation: AWS Management Console → EFS Console → [Create file system]
```

1. **Step 1: Basic Configuration**
   * Click **Customize** (do not use default "Quick Create" if you need specific security groups or custom mount targets).
   * **Name:** `production-shared-storage`
   * **File system type:** Select **Regional** (multi-AZ durability).
2. **Step 2: Automatic Backups & Lifecycle Management**
   * Check **Enable automatic backups** (integrates with AWS Backup).
   * **Lifecycle management:**
     * Transition into IA: `30 days since last access`.
     * Transition out of IA: `On first access`.
     * Transition into Archive: `90 days since last access`.
   * **Encryption:** Check **Enable encryption at rest** and choose your KMS Key (`aws/elasticfilesystem` or a Custom Managed Key).
3. **Step 3: Performance Settings**
   * **Throughput mode:** Select **Elastic** (recommended default: scales throughput automatically based on workload demands).
   * **Performance mode:** Select **General Purpose** (lowest latency per file operation).
4. **Step 4: Network Settings (Mount Targets)**
   * Select your **VPC**.
   * For each Availability Zone (`ap-south-1a`, `ap-south-1b`, `ap-south-1c`):
     * Select the specific **Subnet**.
     * IP address: Leave as **Automatic** (or assign static IP).
     * **Security groups:** Remove the default VPC security group and attach your dedicated **EFS Mount Target Security Group** (`efs-mount-target-sg`).
5. **Step 5: Review and Create**
   * Review all configuration parameters and click **Create**.
   * Wait until the File system state changes from `Creating` to **`Available`**.

---

### AWS CLI Creation Commands

Execute the following commands to provision an EFS file system and its mount targets via the CLI:

```bash
# 1. Create the encrypted EFS file system
aws efs create-file-system \
    --region ap-south-1 \
    --performance-mode generalPurpose \
    --throughput-mode elastic \
    --encrypted \
    --kms-key-id alias/aws/elasticfilesystem \
    --tags Key=Name,Value=prod-shared-efs \
    --backup

# Expected Output snippet: "FileSystemId": "fs-0123456789abcdef0"

# 2. Configure lifecycle management policy (Transition to IA after 30 days)
aws efs put-lifecycle-configuration \
    --file-system-id fs-0123456789abcdef0 \
    --lifecycle-policies TransitionToIA=AFTER_30_DAYS,TransitionToPrimaryStorageClass=AFTER_1_ACCESS

# 3. Create Mount Targets in each target subnet
aws efs create-mount-target \
    --file-system-id fs-0123456789abcdef0 \
    --subnet-id subnet-01fdd11d51e49c845 \
    --security-groups sg-0111111111efsmount
```

---

## 5. EFS Networking, Mount Targets & Security Groups

### VPC & Subnet Requirements
* EFS operates entirely within an Amazon Virtual Private Cloud (VPC).
* In each Availability Zone where you have EC2 instances or container hosts, you must provision an EFS Mount Target inside that AZ's subnet.
* The subnet must have available private IP addresses from its CIDR block.

---

### Security Group Design: The Gold Standard

> [!CAUTION]
> **The #1 Cause of EFS Mount Hangs:** Opening NFS port 2049 to `0.0.0.0/0` is a major security risk, while omitting the rule entirely causes the Linux `mount` command to hang indefinitely. Use the **Security Group Reference Pattern**!

```
+------------------------------------+           TCP Port 2049 (NFS)           +------------------------------------+
|       EC2 Security Group           | ======================================> |    EFS Mount Target Security Group |
|          (sg-0aaa1111)             |     Source: sg-0aaa1111 (EC2 SG ID)     |             (sg-0bbb2222)          |
|  Attached to: Web / App Instances  |                                         |     Attached to: EFS Mount Target  |
+------------------------------------+                                         +------------------------------------+
```

#### Security Group Configuration Rules:

1. **EC2 Instance Security Group (`ec2-instance-sg`):**
   * **Outbound Rule:**
     * Type: `NFS`
     * Protocol: `TCP`
     * Port: `2049`
     * Destination: `efs-mount-target-sg` (or `0.0.0.0/0` default outbound).

2. **EFS Mount Target Security Group (`efs-mount-target-sg`):**
   * **Inbound Rule:**
     * Type: `NFS`
     * Protocol: `TCP`
     * Port: `2049`
     * **Source:** Select **`ec2-instance-sg`** (Enter the Security Group ID of your EC2 instances!).
   * *Why this is superior:* Any EC2 instance attached to `ec2-instance-sg` is automatically permitted to connect to EFS without hardcoding subnet IP addresses or exposing port 2049 to unintended resources.

---

### NFS Protocol & Port 2049
* EFS strictly utilizes **NFSv4.0 and NFSv4.1**.
* Unlike legacy NFSv3, NFSv4 does not require port mappers (`rpcbind`), `mountd`, or multiple random ports. It operates cleanly over a **single dedicated TCP port: 2049**.

---

### DNS Resolution Mechanics

AWS provides two DNS endpoints for every EFS file system:

1. **Regional DNS Name (Recommended):**
   ```
   file-system-id.efs.aws-region.amazonaws.com
   # Example: fs-0123456789abcdef0.efs.ap-south-1.amazonaws.com
   ```
   * *How it works:* When an EC2 instance performs a DNS lookup on this name, AWS Route 53 automatically resolves the query to the **local private IP address of the Mount Target in that instance's Availability Zone**.
2. **Zone-Specific DNS Name:**
   ```
   az-name.file-system-id.efs.aws-region.amazonaws.com
   # Example: ap-south-1a.fs-0123456789abcdef0.efs.ap-south-1.amazonaws.com
   ```

---

## 6. Attaching and Mounting EFS on an EC2 Linux Instance

### Prerequisites

Connect to your EC2 instance via SSH or EC2 Instance Connect, and install the required tools.

```
                      WHICH UTILITY TO INSTALL?
                                 |
         +-----------------------+-----------------------+
         |                                               |
  Amazon Linux / RHEL / CentOS                    Ubuntu / Debian
         v                                               v
  sudo dnf install -y amazon-efs-utils            sudo apt update
                                                  sudo apt install -y amazon-efs-utils
                                                  (or: sudo apt install -y nfs-common)
```

#### Installing `amazon-efs-utils` (Recommended):
* On **Amazon Linux 2023 / Amazon Linux 2**:
  ```bash
  sudo dnf install -y amazon-efs-utils
  ```
* On **Ubuntu 20.04 / 22.04 / 24.04**:
  ```bash
  sudo apt update
  sudo apt install -y binutils git
  # If amazon-efs-utils package is not available directly in your repo:
  git clone https://github.com/aws/efs-utils
  cd efs-utils && ./build-deb.sh && sudo apt install -y ./build/amazon-efs-utils*deb
  ```

*On systems where you cannot install `amazon-efs-utils`, install standard NFS:*
```bash
# RHEL / CentOS
sudo dnf install -y nfs-utils

# Ubuntu / Debian
sudo apt install -y nfs-common
```

---

### Step-by-Step Practical Mounting Walkthrough

#### Step 1: Create a local directory for the mount point
```bash
sudo mkdir -p /mnt/efs
```
*Explanation:* Creates the target folder in the Linux directory tree where the EFS filesystem will be mounted.

---

#### Step 2: Mount the file system

##### Method A: Using EFS Mount Helper (Recommended with TLS Encryption)
The EFS mount helper manages client-side TLS certificates and automatic stunnel creation:
```bash
sudo mount -t efs -o tls fs-0123456789abcdef0:/ /mnt/efs
```
*Parameter Breakdown:*
* `-t efs`: Invokes the AWS EFS mount helper wrapper script.
* `-o tls`: Enforces encrypted in-transit communication over TLS 1.2+.
* `fs-0123456789abcdef0:/`: The File System ID and root path.
* `/mnt/efs`: Local mount target directory.

##### Method B: Using Standard Linux NFSv4 Client
If `amazon-efs-utils` is not installed:
```bash
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0123456789abcdef0.efs.ap-south-1.amazonaws.com:/ /mnt/efs
```
*Parameter Breakdown:*
* `nfsvers=4.1`: Selects NFS protocol version 4.1.
* `rsize=1048576,wsize=1048576`: Sets maximum read/write buffer size (1 MiB) to maximize throughput.
* `hard`: Causes application requests to wait indefinitely if EFS becomes temporarily unreachable (prevents data corruption).
* `timeo=600`: Sets timeout to 60 seconds (600 tenths of a second).
* `retrans=2`: Retransmits unacknowledged requests twice before timeout recovery.
* `noresvport`: Tells the client to use non-privileged source ports (required by AWS EFS).

---

### Verifying the Mounted File System

#### 1. Check mounted filesystems:
```bash
df -hT /mnt/efs
```
*Sample Output:*
```
Filesystem                                            Type  Size  Used Avail Use% Mounted on
fs-0123456789abcdef0.efs.ap-south-1.amazonaws.com:/  nfs4  8.0E     0  8.0E   0% /mnt/efs
```
> [!NOTE]
> EFS reports an apparent size of **8.0 Exabytes (8.0E)** in `df -h`. Because EFS is an elastic serverless storage pool, it presents an arbitrarily large virtual capacity to POSIX clients.

#### 2. Test reading and writing:
```bash
# Create a test file
echo "EFS shared storage initialized at $(date)" | sudo tee /mnt/efs/test_shared.txt

# Verify content
cat /mnt/efs/test_shared.txt
```

---

## 7. Mounting EFS on Multiple EC2 Instances

The primary power of Amazon EFS is simultaneous multi-host read and write access.

```
+---------------------------------------------------------------------------------------------------+
|                                 Amazon EFS: fs-0123456789abcdef0                                  |
|                                     Directory: /mnt/efs/                                          |
+---------------------------------------------------------------------------------------------------+
                     |                                                   |
                     | 1. Writes file: "server-01.log"                   | 2. Instantly reads:
                     v                                                   v    "server-01.log"
        +-------------------------+                         +-------------------------+
        |     EC2 Instance 1      |                         |     EC2 Instance 2      |
        |  Hostname: web-app-01   |                         |  Hostname: web-app-02   |
        |  AZ: ap-south-1a        |                         |  AZ: ap-south-1b        |
        +-------------------------+                         +-------------------------+
```

### Real-World Multi-Instance Setup Demonstration

1. **On EC2 Instance 1 (`web-app-01` in AZ-1a):**
   ```bash
   sudo mkdir -p /mnt/efs
   sudo mount -t efs -o tls fs-0123456789abcdef0:/ /mnt/efs
   echo "Written by Instance 1 ($(hostname)) on $(date)" | sudo tee /mnt/efs/instance1.txt
   ```

2. **On EC2 Instance 2 (`web-app-02` in AZ-1b):**
   ```bash
   sudo mkdir -p /mnt/efs
   sudo mount -t efs -o tls fs-0123456789abcdef0:/ /mnt/efs
   # Instantly view the file written by Instance 1:
   cat /mnt/efs/instance1.txt
   # Append new data from Instance 2:
   echo "Appended by Instance 2 ($(hostname)) on $(date)" | sudo tee -a /mnt/efs/instance1.txt
   ```

3. **Verify back on Instance 1:**
   ```bash
   cat /mnt/efs/instance1.txt
   # Output contains data from both instances!
   ```

---

### Distributed File Locking & Concurrency
* EFS supports standard POSIX advisory locks using `fcntl()` and `flock()`.
* Multiple application threads across different physical servers can lock files or byte ranges within files to ensure safe transactional updates (e.g., SQLite databases, CMS uploads, logging engines).

---

## 8. Automatic Mounting After Reboot (`/etc/fstab`)

To ensure your EC2 instance automatically mounts EFS when booting up, add an entry to `/etc/fstab`.

> [!CAUTION]
> If network file systems are listed in `/etc/fstab` without proper networking flags (`_netdev`, `nofail`), your EC2 instance will **hang during system boot** and become unreachable over SSH!

---

### Configuring `/etc/fstab` with EFS Mount Helper (Recommended)

1. **Backup `/etc/fstab`:**
   ```bash
   sudo cp /etc/fstab /etc/fstab.bak
   ```
2. **Open `/etc/fstab` and append:**
   ```
   fs-0123456789abcdef0:/  /mnt/efs  efs  _netdev,tls,nofail  0  0
   ```

*(If using an EFS Access Point):*
```
fs-0123456789abcdef0:/  /mnt/efs  efs  _netdev,tls,accesspoint=fsap-01122334455667788,nofail  0  0
```

---

### Configuring `/etc/fstab` with Standard NFS4

If you are not using `amazon-efs-utils`:
```
fs-0123456789abcdef0.efs.ap-south-1.amazonaws.com:/  /mnt/efs  nfs4  nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev,nofail  0  0
```

---

### The Critical `_netdev` and `nofail` Mount Options

| Parameter | Purpose & Failure Consequence |
| :--- | :--- |
| **`_netdev`** | **CRITICAL:** Informs systemd and the Linux init system that this filesystem requires network connectivity. Systemd delays mounting until all network interfaces (DHCP, routing, DNS) are fully active. |
| **`nofail`** | **CRITICAL:** Instructs Linux to continue the system boot sequence even if the EFS file system cannot be mounted (e.g., security group misconfiguration or DNS delay). **Without `nofail`, the server drops into emergency recovery mode upon reboot!** |
| `tls` | Enables background stunnel daemon encryption for data in transit. |
| `0 0` | Disables dump filesystem backup and fsck disk checking (NFS does not support local fsck). |

#### Test `/etc/fstab` immediately without rebooting:
```bash
# Unmount current session
sudo umount /mnt/efs

# Test mounting all filesystems defined in /etc/fstab
sudo mount -a

# Verify success
df -hT /mnt/efs
```

---

## 9. EFS Encryption Architecture

Amazon EFS provides comprehensive, defense-in-depth encryption for data at rest and data in transit.

```
[ EC2 Instance OS ]
        |
        v (Local Mount Helper)
   [ stunnel ] ================== TLS 1.2+ In Transit (Port 2049) ==================> [ EFS Mount Target ]
                                                                                             |
                                                                                             v
                                                                                   [ EFS Storage Server ]
                                                                                   (AES-256 at Rest via
                                                                                    AWS KMS Key)
```

### Encryption at Rest (AWS KMS)
* **Standard:** Industry-standard **AES-256** encryption algorithm.
* **Scope:** Encrypts file data, directory metadata, file names, symlinks, and directory structures.
* **Key Types:**
  * **AWS Managed Key (`alias/aws/elasticfilesystem`):** Default, zero cost, automatically provisioned.
  * **Customer Managed Key (CMK):** Allows custom key policies, automated yearly key rotation, and cross-account access grants.
* **Important Constraint:** Encryption at rest **must be enabled when creating the file system**. You cannot toggle encryption on an existing unencrypted EFS file system. (To encrypt an unencrypted EFS, you must create a new encrypted file system and replicate/sync data using AWS DataSync).

---

### Encryption in Transit (TLS via Stunnel)
* Implemented transparently by `amazon-efs-utils`.
* When you specify `-o tls`, the EFS mount helper spawns a local background supervisor process (`amazon-efs-mount-watchdog`) running an encrypted **stunnel** client.
* Traffic between your application and the local stunnel port is plaintext loopback, but all traffic exiting the network interface towards the EFS Mount Target is strictly encrypted with **TLS 1.2+**.

---

### Enforcing In-Transit Encryption via File System Policy

You can enforce compliance by configuring an EFS File System Policy that denies any client attempting to connect over unencrypted NFS:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EnforceTLSOnly",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "*",
            "Resource": "arn:aws:elasticfilesystem:ap-south-1:022278165851:file-system/fs-0123456789abcdef0",
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```

---

## 10. EFS Access Points & IAM Authorization

### What are EFS Access Points?
An **EFS Access Point** provides an application-specific gateway into an EFS file system, simplifying shared storage management for containerized and multi-tenant environments.

```
                              EFS FILE SYSTEM (fs-0123456789)
+-----------------------------------------------------------------------------------------+
|  Root Directory: /                                                                      |
|  ├── /production/app1/  <============= Access Point 1 (Enforces UID: 1001, Root: /app1) |
|  │    └── data.txt                     Used by: ECS Task Service A                      |
|  └── /production/app2/  <============= Access Point 2 (Enforces UID: 1002, Root: /app2) |
|       └── database.db                  Used by: ECS Task Service B                      |
+-----------------------------------------------------------------------------------------+
```

### Key Capabilities of Access Points:

1. **Enforced POSIX User Identity:**
   * Overrides whatever UID/GID the client OS claims to have.
   * All file operations initiated through the access point are executed using the configured POSIX UID (e.g., `1001`) and GID (e.g., `1001`), regardless of whether the connecting client is logged in as `root` (`UID 0`) or `ec2-user` (`UID 1000`).
2. **Root Directory Jailing (Chroot-like behavior):**
   * Configures a custom root path (e.g., `/app1/uploads`).
   * When an EC2 instance or Docker container mounts the access point, it is "jailed" to that directory. The client cannot see, traverse, or access parent folders.
3. **Automated Directory Creation:**
   * If the target directory path does not exist on EFS, AWS will automatically create it upon first mount, applying the pre-configured owner UID, GID, and octal permissions (e.g., `0755`).

#### Mounting an EFS Access Point:
```bash
sudo mount -t efs -o tls,accesspoint=fsap-01122334455667788 fs-0123456789abcdef0:/ /mnt/app
```

---

### IAM Client Authorization Policies

Control client access through IAM policies attached to EC2 Instance Profiles or ECS Task Roles:

| IAM Action | What It Permits |
| :--- | :--- |
| `elasticfilesystem:ClientMount` | Grants permission to connect and read files. |
| `elasticfilesystem:ClientWrite` | Grants permission to write and modify files. |
| `elasticfilesystem:ClientRootAccess` | Disables root squashing, allowing `UID 0` to maintain root privileges. |

---

## 11. File Permissions, Ownership & POSIX Semantics

### Linux Permissions & Directory Ownership

When an EFS file system is first mounted, the root directory (`/`) is owned by `root:root` with default permissions `0755` (`rwxr-xr-x`):
* The `root` user has full read/write/execute rights.
* Non-root users (like `ec2-user`, `ubuntu`, or `www-data`) have **read and execute rights only**.
* Attempting to write as a non-root user results in: `touch: cannot touch 'file': Permission denied`.

#### Fixing Permissions for Multi-User Access:
```bash
# Option 1: Give ownership to a specific user (e.g., ec2-user)
sudo chown -R ec2-user:ec2-user /mnt/efs

# Option 2: Give ownership to a shared group (e.g., www-data or devops)
sudo chown -R root:devops /mnt/efs
sudo chmod -R 775 /mnt/efs

# Option 3: Enable sticky bit for shared collaborative directories
sudo chmod 1777 /mnt/efs
```

---

### Root Squashing Explained
* **Default Security Behavior:** By default, if a client running as `root` (`UID 0`) attempts to create or modify files over NFS, EFS maps (squashes) that user's identity to an unprivileged anonymous user: `nfsnobody` (or `nobody`, `UID 65534`).
* **Why this matters:** Prevents unauthorized root administrators on client machines from overriding security permissions on shared network files.
* **Bypassing Root Squash:** If your application genuinely requires root privileges over the network, grant the IAM permission `elasticfilesystem:ClientRootAccess` to the client's IAM role.

---

## 12. Monitoring, Throughput & Performance Modes

### Throughput Modes

```
+------------------------------------+------------------------------------+
|            Elastic Mode            |          Provisioned Mode          |
|        (Recommended Default)       |                                    |
+------------------------------------+------------------------------------+
| Automatically scales throughput    | You manually dial in a specific    |
| up and down up to 10+ GB/s based   | throughput (e.g., 200 MB/s).       |
| on workload activity.              |                                    |
| Pay only for throughput read/wrote.| Pay for provisioned rate hourly.   |
| Best for: Unpredictable, bursty web| Best for: Constant high-throughput |
| traffic and container workloads.   | batch processing & ETL workloads.  |
+------------------------------------+------------------------------------+
```

---

### Performance Modes

> [!IMPORTANT]
> The Performance Mode **must be chosen at creation time** and cannot be changed after the file system is created!

1. **General Purpose (Recommended Default):**
   * Optimized for lowest latency per file operation (sub-millisecond for metadata operations).
   * Limit: 35,000 IOPS.
   * Ideal for web hosting, CMS, enterprise software development, container persistent volumes.
2. **Max I/O:**
   * Scaled for massive parallel processing across thousands of EC2 instances.
   * Delivers virtually unlimited aggregate throughput and IOPS at the cost of slightly higher per-operation metadata latency.
   * Ideal for Big Data analysis, genomics sequencing, video rendering farms.

---

### Key Amazon CloudWatch Metrics

| CloudWatch Metric Name | Units | Description & Health Threshold |
| :--- | :--- | :--- |
| `MeteredIOBytes` | Bytes | Measures total volume of data read and written. Categorized by storage tier (`Value=Standard`, `Value=IA`). |
| `PermittedThroughput` | Bytes/s | The maximum throughput permitted based on file system size and throughput mode. |
| `PercentIOLimit` | Percentage (%) | Percentage of available I/O limit consumed in General Purpose mode. **If this reaches 100%, consider moving to Max I/O mode!** |
| `ClientConnections` | Count | Number of client EC2/container instances currently connected to the file system. |
| `BurstCreditBalance` | Bytes | Burst bucket reserve balance (for legacy Bursting throughput mode). |

---

## 13. Unmounting EFS Safely

### Safe Unmount Commands

Before detaching or shutting down, cleanly unmount EFS from your operating system.

#### Step 1: Check for processes holding open files on the mount
```bash
sudo lsof +D /mnt/efs
# Or:
sudo fuser -v -m /mnt/efs
```

#### Step 2: Unmount the directory
```bash
sudo umount /mnt/efs
```

Verify that the device has detached:
```bash
df -h | grep efs
```

---

### Resolving "Device or Resource Busy" Errors

If you execute `sudo umount /mnt/efs` and receive:
```
umount: /mnt/efs: target is busy.
```

**Remediation Steps:**
1. Check if your current shell terminal is inside the directory:
   ```bash
   pwd
   # If output is /mnt/efs, change directory out of it!
   cd ~
   ```
2. Kill all processes holding open handles on the directory:
   ```bash
   sudo fuser -k -m /mnt/efs
   ```
3. Use a **Lazy Unmount** if an unresponsive NFS lock persists:
   ```bash
   sudo umount -l /mnt/efs
   ```
   *Explanation:* Detaches the filesystem immediately from the Linux directory hierarchy and cleans up remaining file references once processes terminate.

---

## 14. Backup, Disaster Recovery & Replication

### Centralized Backups with AWS Backup
* EFS natively integrates with **AWS Backup**, a policy-based enterprise backup service.
* During EFS creation, the **Automatic Backups** toggle creates a default daily backup plan with a **35-day retention schedule**.
* Backups are point-in-time, incremental, and encrypted using AWS KMS.
* **Restoration Options:**
  1. **Full File System Restore:** Creates a brand-new EFS file system containing all recovered files.
  2. **Item-Level (Directory) Restore:** Restores selected directories directly into an existing live EFS file system under a subfolder named `/aws-backup-restore-[timestamp]`.

---

### EFS Native Cross-Region Replication

For enterprise Disaster Recovery (DR), EFS provides native, automatic, continuous asynchronous replication:
* Replicates data to a secondary EFS file system in another AWS Region (e.g., from `ap-south-1` to `us-east-1`) or within the same Region.
* **Recovery Point Objective (RPO):** Typically under **15 minutes**.
* The replica file system is read-only. In a disaster recovery event, you failover by deleting the replication configuration, which instantly converts the replica into a read-write production file system.

```
+------------------------------------+           Continuous Asynchronous          +------------------------------------+
|       Primary EFS File System      | =========================================> |       Replica EFS File System      |
|         Region: ap-south-1         |                 Replication                |          Region: us-east-1         |
|             (Read/Write)           |               (RPO < 15 mins)              |             (Read-Only)            |
+------------------------------------+                                            +------------------------------------+
```

---

## 15. Diagnostic Troubleshooting Matrix

| Symptom / Error Message | Root Cause | Remediation Command / Fix |
| :--- | :--- | :--- |
| **Mount command hangs indefinitely** | TCP Port 2049 blocked in Security Group or wrong subnet. | Verify that EFS Mount Target Security Group allows **Inbound TCP 2049** from the EC2 instance's security group. |
| **`Failed to resolve "fs-xxxx.efs...amazonaws.com"`** | VPC DNS Resolution or DNS Hostnames disabled. | Open VPC Console → Select VPC → Edit VPC Settings → Check **Enable DNS hostnames** and **Enable DNS resolution**. |
| **`mount.efs: command not found`** | Package `amazon-efs-utils` is not installed. | Run `sudo dnf install -y amazon-efs-utils` (Amazon Linux) or `sudo apt install -y nfs-common` (Ubuntu). |
| **`Permission Denied` when writing files** | Root squashing active or local folder owned by `root:root (0755)`. | Run `sudo chown -R ec2-user:ec2-user /mnt/efs` or configure an EFS Access Point with enforced POSIX UID/GID. |
| **EC2 hangs on reboot / emergency mode** | Missing `_netdev` or `nofail` flag in `/etc/fstab`. | Edit `/etc/fstab` to ensure `_netdev,nofail` are present in mount options. |
| **Slow read/write performance on small files** | NFS metadata round-trip latency on thousands of tiny files. | Combine small files into tar archives, parallelize transfers with `fpsync`, or use local caching. |

---

## 16. Best Practices & Interview Preparation

### Architecture & Cost Optimization Best Practices

#### Security & Compliance:
1. **Always Use Security Group Referencing:** Never open port 2049 to CIDR blocks; always whitelist the source EC2 Security Group ID.
2. **Enforce Encryption in Transit:** Mount using `tls` and enforce TLS via File System IAM Policies.
3. **Use Access Points for Multi-Tenant Environments:** Enforce directory jailing and standard POSIX UIDs for container workloads.

#### High Availability & Durability:
1. **Prefer Regional Over One Zone:** Always select Regional EFS for production workloads to survive total AZ outages.
2. **Never Omit `_netdev` & `nofail` in `/etc/fstab`:** Protect your EC2 instances from boot failure loops.
3. **Deploy Mount Targets in Every Application Subnet:** Ensure low-latency, free local-AZ traffic routing.

#### Cost Optimization:
1. **Enable EFS Lifecycle Management:** Transitioning files to Infrequent Access after 14 or 30 days saves up to **92% on storage costs**.
2. **Select Elastic Throughput Mode:** Avoid paying for provisioned throughput capacity you are not consuming.

---

### High-Yield Technical Interview Q&A

#### Q1: What is the main difference between Amazon EBS and Amazon EFS?
> **Answer:** Amazon EBS provides block-level storage designed for single EC2 instances within a single Availability Zone (like a dedicated hard disk). Amazon EFS provides file-level storage based on the NFSv4 protocol that can be mounted concurrently by thousands of EC2 instances and containers across multiple Availability Zones.

#### Q2: What happens if an EC2 instance in `us-east-1a` mounts an EFS file system? How does AWS route the traffic?
> **Answer:** If the instance mounts using the Regional DNS name (`file-system-id.efs.us-east-1.amazonaws.com`), AWS Route 53 automatically resolves the DNS name to the private IP address of the Mount Target located inside `us-east-1a`. This keeps traffic local to that Availability Zone, ensuring maximum performance and zero cross-AZ data transfer fees.

#### Q3: Why is the `_netdev` option critical in `/etc/fstab` for EFS?
> **Answer:** Because EFS is a network-attached filesystem. The `_netdev` parameter instructs systemd and the Linux bootloader to delay mounting the filesystem until the network stack and networking interfaces have fully initialized. Without `_netdev`, Linux tries to mount EFS before the network is up, causing the boot process to freeze.

#### Q4: What is an EFS Access Point, and why is it useful in Kubernetes (EKS)?
> **Answer:** An EFS Access Point is an application-specific gateway that enforces a specific POSIX user identity (UID/GID) and restricts client access to a specific root directory path. In Kubernetes (EKS), Access Points allow different pods and microservices to share the same EFS file system while remaining isolated inside their own provisioned subdirectories without permission conflicts.

#### Q5: Can you mount Amazon EFS from on-premises servers?
> **Answer:** Yes. On-premises physical servers can mount an EFS file system over an **AWS Direct Connect** or **AWS Site-to-Site VPN** connection by targeting the private IP address or DNS name of a Mount Target in the connected VPC.

---

### Quick Command Cheat Sheet

```bash
# ==========================================
# AMAZON EFS LINUX ADMINISTRATION CHEAT SHEET
# ==========================================

# 1. Install EFS Utilities
sudo dnf install -y amazon-efs-utils            # Amazon Linux 2023 / RHEL
sudo apt install -y nfs-common                   # Ubuntu / Debian

# 2. Mount Commands
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls fs-0123456789abcdef0:/ /mnt/efs                      # EFS helper with TLS
sudo mount -t efs -o tls,accesspoint=fsap-xxxx fs-0123456789abcdef0:/ /mnt/efs # EFS with Access Point
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-xxxx.efs.region.amazonaws.com:/ /mnt/efs # Standard NFS4

# 3. Persistent Mount (/etc/fstab entry)
# fs-0123456789abcdef0:/  /mnt/efs  efs  _netdev,tls,nofail  0  0

# 4. Verification & Diagnostic
df -hT /mnt/efs                                  # Check filesystem type and capacity
sudo mount -a                                    # Test /etc/fstab entries without rebooting
sudo lsof +D /mnt/efs                            # Find open files on EFS mount

# 5. Safe Unmount
sudo umount /mnt/efs                             # Normal unmount
sudo umount -l /mnt/efs                          # Lazy unmount (if resource is busy)
```
