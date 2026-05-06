# Part 1: Introduction to AWS Storage Services

---

## Table of Contents

1. [Why Storage Matters in AWS](#1-why-storage-matters-in-aws)
2. [The Three Storage Paradigms: Block, File, Object](#2-the-three-storage-paradigms-block-file-object)
3. [Block Storage — What It Is and How It Works](#3-block-storage--what-it-is-and-how-it-works)
4. [File Storage — What It Is and How It Works](#4-file-storage--what-it-is-and-how-it-works)
5. [Object Storage — What It Is and How It Works](#5-object-storage--what-it-is-and-how-it-works)
6. [AWS Storage Services Overview](#6-aws-storage-services-overview)
7. [EBS — Elastic Block Store](#7-ebs--elastic-block-store)
8. [EFS — Elastic File System](#8-efs--elastic-file-system)
9. [Amazon FSx](#9-amazon-fsx)
10. [S3 — Simple Storage Service](#10-s3--simple-storage-service)
11. [Storage Services Comparison](#11-storage-services-comparison)
12. [EBS vs EC2 Instance Store](#12-ebs-vs-ec2-instance-store)
13. [Self-Check Questions — Answered](#13-self-check-questions--answered)

---

## 1. Why Storage Matters in AWS

Every application needs to store data — databases need disk volumes, web servers need shared configuration files, data pipelines need a place to dump terabytes of logs. In on-premises infrastructure, you buy hard drives, plug them in, and format them. In AWS, storage is a service you provision on-demand, and the type of storage you choose affects everything: performance, cost, availability, and how your application architecture fits together.

AWS offers multiple storage services because no single storage type solves every problem. A relational database running on EC2 needs low-latency block storage attached directly to the instance. A fleet of 50 web servers sharing the same static assets needs a network file system they can all mount. A data lake ingesting millions of small files per day needs an object store with unlimited capacity and built-in durability.

Choosing wrong has real consequences. If you use EBS (block storage) where you needed EFS (file storage), you will spend weeks building workarounds for cross-instance file sharing that EFS provides out of the box. If you use EFS where you needed S3, you will pay significantly more per GB for a feature set you don't need. This is why understanding the fundamental differences — not just the service names — is the starting point for every storage decision in AWS.

---

## 2. The Three Storage Paradigms: Block, File, Object

Before looking at any AWS service, you need to understand the three storage paradigms that all cloud and on-premises storage systems are built on. Every AWS storage service maps to exactly one of these three types.

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                     THE THREE STORAGE PARADIGMS                                    │
│                                                                                    │
│   BLOCK                    FILE                      OBJECT                        │
│   ─────                    ────                      ──────                        │
│                                                                                    │
│   ┌──┬──┬──┐               /root                     ┌──────────────────────┐      │
│   │01│02│03│               ├── etc/                   │  Key: photos/a.jpg   │      │
│   ├──┼──┼──┤               │   ├── nginx.conf         │  Value: <binary>     │      │
│   │04│05│06│               │   └── hosts              │  Metadata:           │      │
│   ├──┼──┼──┤               ├── var/                   │    content-type: jpg │      │
│   │07│08│09│               │   └── log/               │    size: 2.4MB       │      │
│   └──┴──┴──┘               │       └── app.log        │    created: 2024-01  │      │
│                             └── home/                  └──────────────────────┘      │
│   Raw numbered blocks       └── user/                                               │
│   No structure               Hierarchical tree        Flat key-value pairs          │
│   OS adds filesystem         Shared via NFS/SMB       Accessed via HTTP API         │
│                                                                                    │
│   AWS: EBS                  AWS: EFS, FSx             AWS: S3                       │
└────────────────────────────────────────────────────────────────────────────────────┘
```

These are not interchangeable. Each paradigm has different access patterns, performance characteristics, and architectural implications. The next three sections break down each one.

---

## 3. Block Storage — What It Is and How It Works

Block storage presents a raw volume of fixed-size blocks (typically 512 bytes or 4KB) to the operating system. The OS does not receive files or directories — it receives numbered blocks that it must organize using a filesystem (ext4, XFS, NTFS). This is identical to how a physical hard drive or SSD works inside your laptop.

### How a block volume operates

When you attach an EBS volume to an EC2 instance, the instance's operating system sees it as a raw disk device — exactly like `/dev/sdf` on Linux or `D:\` on Windows. The OS then:

1. **Formats** the volume with a filesystem (ext4, XFS, etc.)
2. **Mounts** it to a directory (e.g., `/data`)
3. **Reads and writes** individual blocks as needed

The critical characteristic of block storage is that it operates at the **lowest level of abstraction**. The storage system itself has no concept of files, folders, or metadata — it only knows block numbers and the raw bytes at each block. This low-level access is what gives block storage its speed advantage, because there is no overhead from protocol parsing, metadata lookups, or network file locking.

### Why block storage is used for databases

A relational database like PostgreSQL or MySQL manages its own internal file format. It needs to read block 47,293 in 0.2 milliseconds, update 4 bytes at offset 128 within that block, and flush the change to disk immediately. This random I/O pattern — reading and writing small pieces scattered across the volume — is exactly what block storage is designed for.

File storage and object storage add layers of abstraction that introduce latency. A database needs direct control over which blocks are read and written, in what order, with what consistency guarantees. Block storage provides this.

### Key characteristics of block storage

| Property | Detail |
|:---------|:-------|
| Access pattern | Random read/write at block level |
| Latency | Sub-millisecond (SSD-backed) |
| Attachment | Typically one host at a time |
| Filesystem | Required — the OS must format the volume |
| Ideal for | Databases, boot volumes, transaction-heavy applications |
| AWS service | EBS (Elastic Block Store) |

---

## 4. File Storage — What It Is and How It Works

File storage presents a hierarchical directory structure over a network protocol. Multiple machines connect to the same file system simultaneously and see the same directory tree with the same files. This is the model used by Network Attached Storage (NAS) devices in corporate offices and data centers.

### How a file system is shared

The file storage server exposes a directory tree over a network protocol:

- **NFS (Network File System)** — the standard for Linux/Unix systems
- **SMB (Server Message Block)** — the standard for Windows systems

Client machines mount the remote filesystem to a local path. From the application's perspective, the files appear local — you read and write them with normal file operations (`open()`, `read()`, `write()`). The network protocol handles synchronization, locking, and consistency between multiple concurrent clients.

```
Server: EFS (or FSx)
    /shared
    ├── config/
    │   └── app.yaml
    ├── uploads/
    │   ├── report_q1.pdf
    │   └── report_q2.pdf
    └── logs/
        └── combined.log

EC2 Instance A:                EC2 Instance B:               EC2 Instance C:
mount -t nfs efs:/shared       mount -t nfs efs:/shared       mount -t nfs efs:/shared
     │                              │                              │
     └─ sees /shared/*              └─ sees /shared/*              └─ sees /shared/*
        (same files)                   (same files)                   (same files)
```

### Why file storage exists alongside block storage

Block storage is faster, but it attaches to one instance. If you have 50 web servers that all need to read the same configuration file, you have two choices:

1. Copy the file to an EBS volume on each instance and manage synchronization yourself
2. Mount a shared EFS filesystem on all 50 instances and write the file once

Option 2 is what file storage exists for. The trade-off is higher latency per operation (network round-trip) in exchange for shared access across many instances.

### Key characteristics of file storage

| Property | Detail |
|:---------|:-------|
| Access pattern | Hierarchical — directories and files |
| Latency | Low milliseconds (network-dependent) |
| Attachment | Multiple hosts simultaneously |
| Protocol | NFS (Linux), SMB (Windows) |
| Ideal for | Shared content, CMS, home directories, code repositories |
| AWS services | EFS (Linux/NFS), FSx (Windows/SMB, Lustre, ONTAP, OpenZFS) |

---

## 5. Object Storage — What It Is and How It Works

Object storage is fundamentally different from both block and file storage. There is no filesystem, no directory hierarchy, no mounting. Instead, data is stored as **objects** in a flat namespace called a **bucket**. Each object consists of three components:

1. **Key** — a unique identifier string (e.g., `logs/2024/march/app.log`)
2. **Value** — the actual data (bytes), up to 5TB per object
3. **Metadata** — key-value pairs describing the object (content-type, creation date, custom tags)

### The flat namespace illusion

What appears in the AWS console as a folder structure is not actually a hierarchy. S3 has no concept of directories. The `/` character in a key is just a character — the console uses it as a visual delimiter to render a folder-like view, but the underlying storage is completely flat.

```
What the S3 console displays:              What S3 actually stores:
──────────────────────────────              ──────────────────────────────
📁 images/                                  Key: "images/cats/photo1.jpg"
  📁 cats/                                  Key: "images/cats/photo2.jpg"
    📄 photo1.jpg                           Key: "images/dogs/puppy.jpg"
    📄 photo2.jpg                           Key: "docs/report.pdf"
  📁 dogs/
    📄 puppy.jpg
📁 docs/
  📄 report.pdf

These are 4 independent objects in a flat namespace.
There is no "images" directory. There is no nesting.
"images/cats/photo1.jpg" is a single string key, not a path.
```

This distinction matters because operations that are trivial in a filesystem — like renaming a directory — are expensive in object storage. Renaming `images/` to `photos/` means copying every object with a new key prefix and deleting the originals. There is no rename API for prefixes.

### Why object storage exists alongside file and block storage

Object storage sacrifices two things that block and file storage provide — low latency and POSIX semantics — in exchange for three things they cannot match:

1. **Unlimited capacity** — no pre-provisioning, no capacity planning, no volume resizing
2. **11-nines durability** — 99.999999999% — AWS replicates across at least 3 AZs automatically
3. **HTTP accessibility** — any device, any language, any platform can read/write via a REST API

These trade-offs make object storage ideal for data that is written once and read many times (or infrequently), rather than data that requires constant random updates.

### Key characteristics of object storage

| Property | Detail |
|:---------|:-------|
| Access pattern | Key-based — put, get, delete by key |
| Latency | ~100ms (first byte), HTTP overhead |
| Attachment | Not attached to any instance — accessed via HTTP API |
| Capacity | Unlimited |
| Ideal for | Data lakes, backups, static assets, log archives, media files |
| AWS service | S3 (Simple Storage Service) |

---

## 6. AWS Storage Services Overview

AWS maps the three storage paradigms to four primary services:

```
Storage Paradigm          AWS Service              Access Model
────────────────          ───────────              ────────────
Block Storage       →     EBS                      Attached to one EC2 instance*
File Storage        →     EFS                      Mounted by many EC2s (NFS)
File Storage        →     FSx                      Mounted by many EC2s (SMB/Lustre/ONTAP/ZFS)
Object Storage      →     S3                       HTTP API from anywhere

* io1/io2 volumes support multi-attach to up to 16 instances in the same AZ
```

Each service exists because it solves a fundamentally different problem. The rest of this section provides an architectural overview of each — deep dives into individual services follow in Parts 2-5 of this series.

---

## 7. EBS — Elastic Block Store

EBS provides persistent block-level storage volumes that attach to EC2 instances. A volume behaves like an unformatted external hard drive — you attach it, format it with a filesystem, mount it, and use it for any workload that needs disk I/O.

### Scope and attachment model

An EBS volume exists in a **single Availability Zone**. It can only be attached to EC2 instances in that same AZ. You cannot attach an EBS volume in `eu-west-1a` to an instance in `eu-west-1b` — the volume and instance must share an AZ.

This is because EBS operates over a high-speed network within the AZ's physical data center. Cross-AZ network latency would violate the sub-millisecond guarantees that databases and applications depend on.

```
AZ: eu-west-1a                              AZ: eu-west-1b
┌────────────────────────────────────┐       ┌──────────────────────┐
│                                    │       │                      │
│   EC2 Instance                     │       │  EC2 Instance        │
│   ├── /dev/xvda  →  EBS vol-aaa   │       │  ├── /dev/xvda       │
│   └── /dev/xvdf  →  EBS vol-bbb   │       │  │   → EBS vol-ccc   │
│                                    │       │  │                    │
│   EBS vol-aaa (gp3, 100GB)        │       │  Cannot attach        │
│   EBS vol-bbb (io2, 500GB)        │       │  vol-aaa or vol-bbb  │
│                                    │       │  (wrong AZ)          │
└────────────────────────────────────┘       └──────────────────────┘
```

### Volume types

EBS offers six volume types, divided into two categories based on the underlying hardware:

**SSD-backed volumes** — optimized for random I/O (databases, boot volumes):

| Type | Name | Max IOPS | Max Throughput | Use Case |
|:-----|:-----|:---------|:---------------|:---------|
| gp3 | General Purpose SSD | 16,000 | 1,000 MB/s | Default choice. Best price-to-performance ratio. |
| gp2 | General Purpose SSD | 16,000 | 250 MB/s | Older generation. Burst-based. Use gp3 instead. |
| io2 | Provisioned IOPS SSD | 64,000 | 1,000 MB/s | High-performance databases (Oracle, SAP HANA). |
| io2 BE | Block Express | 256,000 | 4,000 MB/s | Extreme workloads. Largest EBS volumes (64TB). |

**HDD-backed volumes** — optimized for sequential I/O (big data, logs, throughput):

| Type | Name | Max IOPS | Max Throughput | Use Case |
|:-----|:-----|:---------|:---------------|:---------|
| st1 | Throughput Optimized HDD | 500 | 500 MB/s | Data warehouses, Hadoop, log processing. |
| sc1 | Cold HDD | 250 | 250 MB/s | Infrequent access, lowest cost. Archival workloads. |

> **gp3 is the default and recommended starting point.** You should only choose io2 when you have measured IOPS requirements that exceed gp3's 16,000 ceiling, and only choose HDD types when your workload is purely sequential (streaming reads/writes of large files) and latency does not matter.

### Key features

**Snapshots** — A point-in-time backup of an EBS volume, stored incrementally in S3. Snapshots can be used to create new volumes in any AZ within the same region, or copied to other regions for disaster recovery. Only the blocks that have changed since the last snapshot are stored, making subsequent snapshots fast and cost-efficient.

**Encryption** — AES-256 encryption at rest and in transit between the instance and the volume. Managed through AWS KMS (Key Management Service). Encryption is transparent — the instance and application see unencrypted data. Once enabled on a volume, all snapshots created from it are also encrypted.

**Multi-attach** — Available only for io1 and io2 volume types. Allows a single volume to be attached to up to 16 EC2 instances simultaneously within the same AZ. All instances must use a **cluster-aware filesystem** (GFS2, OCFS2) — standard filesystems like ext4 or XFS will cause data corruption because they are not designed for concurrent write access from multiple hosts.

**Lifecycle Manager (DLM)** — Automates the creation, retention, and deletion of EBS snapshots on a schedule. You define a policy (e.g., "snapshot this volume every 12 hours, retain the last 7 snapshots") and DLM manages the rest.

### Persistence model

EBS volumes are independent of the EC2 instance lifecycle. When you stop or terminate an instance, the root volume can optionally be preserved (controlled by the `DeleteOnTermination` flag). Additional data volumes persist by default until you explicitly delete them.

---

## 8. EFS — Elastic File System

EFS is a fully-managed NFS file system that can be mounted by hundreds or thousands of EC2 instances simultaneously across multiple Availability Zones. It automatically grows and shrinks as you add and remove files — there is no capacity provisioning.

### How EFS differs from EBS

The fundamental difference is the access model. EBS is a volume attached to one instance. EFS is a filesystem mounted by many instances over NFS.

```
EBS model:                              EFS model:
──────────                              ──────────
  EC2-A ──── vol-aaa                      EC2-A ──┐
  EC2-B ──── vol-bbb                      EC2-B ──┼──── EFS filesystem
  EC2-C ──── vol-ccc                      EC2-C ──┤
                                          EC2-D ──┘
  Each instance has its own               All instances see the same
  isolated volume. No sharing.            files. Changes are visible
                                          to all immediately.
```

### Protocol and OS support

EFS uses **NFSv4.1**. This means it only works with operating systems that support NFS — in practice, this is **Linux only**. Windows EC2 instances cannot mount EFS natively. For Windows file sharing, you use FSx for Windows File Server (covered in the next section).

### Storage classes

| Storage Class | Description | When to Use |
|:-------------|:------------|:------------|
| Standard | Multi-AZ, frequently accessed | Active data that applications read/write regularly |
| Standard-IA | Multi-AZ, infrequent access | Data accessed less than once a month. Lower storage cost, per-access fee. |
| One Zone | Single-AZ, frequently accessed | Non-critical data where multi-AZ durability is unnecessary. 47% cheaper than Standard. |
| One Zone-IA | Single-AZ, infrequent access | Cheapest EFS option. Single AZ, infrequent access. |

**Lifecycle policies** automatically move files between storage classes based on access patterns. You configure a rule like "move files to IA if not accessed for 30 days" and EFS handles the transitions transparently.

### Key features

**Elastic capacity** — EFS has no concept of "volume size." You do not provision 100GB or 500GB. The filesystem starts at 0 bytes and grows as you write files, shrinks as you delete them. You are billed per GB stored per month. This eliminates the capacity planning and over-provisioning that EBS requires.

**Multi-AZ availability** — Standard and Standard-IA storage classes replicate data across all AZs in the region. If an entire AZ fails, the filesystem remains available from other AZs.

**Access Points** — Application-specific entry points into the filesystem. An Access Point enforces a root directory, user ID, group ID, and permissions. This allows multiple applications to share one EFS filesystem while being isolated into their own directory trees with their own permissions.

**Serverless integration** — Lambda functions, ECS containers, and Fargate tasks can mount EFS via Access Points. This makes EFS the primary mechanism for shared persistent storage in serverless architectures.

### Performance modes

| Mode | Characteristic | When to Use |
|:-----|:---------------|:------------|
| General Purpose | Lower latency per operation | Most workloads. Default. |
| Max I/O | Higher aggregate throughput | Thousands of instances accessing the filesystem (big data, media processing) |

### Throughput modes

| Mode | Characteristic | When to Use |
|:-----|:---------------|:------------|
| Bursting | Throughput scales with storage size | Default. Works well when you store enough data to get adequate burst credits. |
| Provisioned | You set the throughput independently of storage | When you need more throughput than your storage size would provide via bursting. |
| Elastic | Automatically scales throughput up/down | Unpredictable workloads. Pay per throughput consumed. |

---

## 9. Amazon FSx

FSx provides fully-managed file systems built on third-party technologies that EFS does not cover. Where EFS is AWS's own NFS implementation for Linux, FSx gives you commercially supported file systems for specialized use cases.

### Why FSx exists

EFS solves the "shared Linux NFS filesystem" use case. But it does not support Windows (SMB protocol), high-performance computing (Lustre parallel filesystem), or hybrid cloud storage (NetApp ONTAP). FSx fills each of these gaps with a dedicated service.

### FSx variants

| FSx Type | Protocol | OS Support | Primary Use Case |
|:---------|:---------|:-----------|:-----------------|
| **FSx for Windows File Server** | SMB | Windows (also Linux via SMB) | Corporate Windows file shares. Active Directory integration. SQL Server storage. .NET applications. |
| **FSx for Lustre** | Lustre | Linux | High-Performance Computing. Machine learning training on large datasets. Video rendering and post-production. Financial modeling and genomics. |
| **FSx for NetApp ONTAP** | NFS, SMB, iSCSI | Linux, Windows | Hybrid cloud workloads migrating from on-premises NetApp. Applications requiring both NFS and SMB simultaneously. Multi-protocol environments. |
| **FSx for OpenZFS** | NFS | Linux | Migrating on-premises ZFS workloads to AWS. Low-latency file serving. Development environments needing fast snapshots and clones. |

### How to choose between EFS and FSx

The decision is straightforward:

```
Need a Linux NFS share?                              → EFS
Need a Windows SMB share or AD integration?          → FSx for Windows
Need massively parallel throughput (HPC/ML)?         → FSx for Lustre
Need both NFS + SMB from one filesystem?             → FSx for NetApp ONTAP
Migrating from on-prem ZFS?                          → FSx for OpenZFS
```

EFS should be the default choice for Linux workloads. Only move to FSx when you need a specific capability that EFS does not provide — Windows support, Lustre parallel I/O, or multi-protocol access.

---

## 10. S3 — Simple Storage Service

S3 is an object storage service with effectively unlimited capacity, 11-nines durability, and HTTP-based access from anywhere. It is the most widely used storage service in AWS and serves as the backbone for data lakes, backups, static websites, log archives, and content distribution.

### Core concepts

**Buckets** — A bucket is a top-level container for objects. Every object in S3 lives inside a bucket. Bucket names are globally unique across all AWS accounts worldwide — if someone else has the name `my-bucket`, you cannot use it. Buckets are created in a specific region, and data does not leave that region unless you explicitly configure replication.

**Objects** — An object is a file stored in S3. Each object consists of:

| Component | Description |
|:----------|:------------|
| Key | The unique identifier within the bucket (e.g., `logs/2024/march/app.log`) |
| Value | The actual data — up to 5TB per object |
| Version ID | A unique string per version (when versioning is enabled) |
| Metadata | System and user-defined key-value pairs (content-type, cache-control, custom tags) |
| Storage Class | Which tier the object is stored in (Standard, IA, Glacier, etc.) |

### Storage classes

S3 provides seven storage classes, arranged from hottest (most accessible, highest cost) to coldest (least accessible, lowest cost):

| Storage Class | Availability | Min Duration | Retrieval | Use Case |
|:-------------|:-------------|:-------------|:----------|:---------|
| S3 Standard | 99.99% | None | Instant | Frequently accessed data. Default. |
| S3 Intelligent-Tiering | 99.9% | None | Instant | Unknown or changing access patterns. AWS auto-moves between tiers. |
| S3 Standard-IA | 99.9% | 30 days | Instant | Infrequent access but needs immediate retrieval. |
| S3 One Zone-IA | 99.5% | 30 days | Instant | Non-critical infrequent data. Single AZ — cheaper but less durable. |
| S3 Glacier Instant | 99.9% | 90 days | Instant | Archive data that rarely needs access but must be retrieved in milliseconds. |
| S3 Glacier Flexible | 99.99% | 90 days | Minutes to hours | Long-term archive. Configurable retrieval speed. |
| S3 Glacier Deep Archive | 99.99% | 180 days | 12 hours | Cheapest storage. Regulatory archives, compliance data retained for years. |

> **Min Duration** means you are billed for at least that many days even if you delete the object sooner. An object stored in Standard-IA and deleted after 5 days is billed for 30 days.

### Key features

**Versioning** — When enabled on a bucket, every overwrite of an object creates a new version instead of replacing the original. All previous versions remain accessible by their Version ID. Deleting a versioned object does not remove it — S3 inserts a **delete marker** that hides the current version. Previous versions still exist and can be recovered by removing the delete marker or requesting a specific Version ID.

**Replication** — Automatically copies objects from a source bucket to a destination bucket. Two types:
- **CRR (Cross-Region Replication)** — replicates to a bucket in a different AWS region. Used for disaster recovery and reducing latency for geographically distributed users.
- **SRR (Same-Region Replication)** — replicates within the same region. Used for log aggregation across accounts, or maintaining a production/test copy.

Both require versioning enabled on source and destination buckets.

**Lifecycle Rules** — Automate transitions between storage classes and object expiration. Example configuration: "Move objects to Standard-IA after 30 days, to Glacier after 90 days, delete after 365 days." This is the primary mechanism for S3 cost optimization.

**Static Website Hosting** — S3 can serve HTML, CSS, JavaScript, and media files directly as a website. You configure an index document (e.g., `index.html`), an error document, and set the bucket policy to allow public reads. For production use, CloudFront (CDN) is placed in front for HTTPS, caching, and custom domains.

**Encryption** — Three server-side encryption options:
- **SSE-S3** — AWS manages the keys entirely. Simplest option.
- **SSE-KMS** — AWS KMS manages the keys. Provides audit trail and key rotation.
- **SSE-C** — Customer provides the encryption key with each request. AWS does not store the key.

**Access control mechanisms:**
1. **IAM Policies** — who (user/role) can do what on which S3 resources
2. **Bucket Policies** — resource-based JSON policies on the bucket itself (supports cross-account access)
3. **ACLs** — legacy per-object/per-bucket access lists (AWS recommends disabling these)
4. **S3 Access Points** — named network endpoints with their own access policy
5. **Pre-signed URLs** — time-limited URLs granting temporary access to a specific object
6. **VPC Endpoints** — restrict access to traffic from specific VPCs only

---

## 11. Storage Services Comparison

| | EBS | EFS | FSx | S3 |
|:--|:-----|:-----|:-----|:-----|
| **Storage type** | Block | File | File | Object |
| **Protocol** | Raw block device | NFSv4.1 | SMB / Lustre / NFS / iSCSI | HTTP REST API |
| **Used with** | EC2 (single instance*) | EC2 (many instances), Lambda, ECS, Fargate | EC2 (depends on filesystem) | Any service — RDS, API Gateway, Lambda, CloudFront, etc. |
| **AZ scope** | Single AZ | Multi-AZ (Standard) or Single-AZ | Multi-AZ or Single-AZ (depends on type) | Multi-AZ (3+ AZs automatically) |
| **Capacity model** | Provisioned (you set the size) | Elastic (grows/shrinks automatically) | Provisioned or elastic (depends on type) | Unlimited (no provisioning) |
| **Max size** | 16TB (gp2/gp3) to 64TB (io2 Block Express) | Unlimited | Up to 65TB (FSx for Windows) | 5TB per object, unlimited total |
| **Durability** | 99.8% – 99.999% (within one AZ) | 99.999999999% (11 nines) | Varies by type | 99.999999999% (11 nines) |
| **Availability** | 99.999% (single AZ) | 99.99% (multi-AZ) | Varies by type | 99.5% – 99.99% (depends on class) |
| **Pricing model** | Pay for provisioned capacity | Pay for what you store | Pay for what you store or provision | Pay for what you store + requests + transfer |
| **Latency** | Sub-millisecond | Low milliseconds | Sub-millisecond to low ms | ~100ms (first byte) |
| **OS support** | Any | Linux only | Depends on filesystem | N/A (API access) |
| **Ideal for** | Databases, boot volumes, transaction workloads | Shared Linux filesystems, CMS, code repos | Windows shares, HPC, hybrid cloud | Data lakes, backups, static sites, archives |

\* *io1/io2 volumes support multi-attach to up to 16 instances within the same AZ*

---

## 12. EBS vs EC2 Instance Store

This distinction comes up constantly in AWS exams and architecture discussions. EC2 Instance Store is **not** EBS — it is a separate storage type with fundamentally different behavior.

| Property | EBS | EC2 Instance Store |
|:---------|:----|:-------------------|
| **Location** | Network-attached — the volume sits on a separate storage server and communicates with the instance over a dedicated network | Physically attached — the storage disk is on the same physical host as the instance |
| **Persistence** | Persists independently of the instance. Survives stop/start. Can be configured to survive termination. | **Ephemeral.** All data is lost when the instance stops, terminates, or the underlying host fails. |
| **Performance** | Good to excellent. gp3: 16k IOPS. io2: 64k IOPS. io2 Block Express: 256k IOPS. | Highest possible — no network latency. Millions of IOPS for NVMe-based stores. |
| **Size** | Configurable at creation. Can be resized. | Fixed by the instance type. Cannot be changed. |
| **Snapshots** | Yes — point-in-time backups to S3 | No snapshot capability |
| **Detach/reattach** | Yes — can move volumes between instances in the same AZ | No — permanently bound to the physical host |
| **Encryption** | Yes — KMS-based encryption at rest and in transit | Yes — hardware-based encryption on supported instance types |
| **Use when** | You need persistent storage. Databases, application data, anything that must survive instance lifecycle events. | You need temporary high-speed storage. Caches, buffers, scratch files, temporary data processing. Data can be regenerated or is replicated elsewhere. |

The rule is simple: if losing the data would be a problem, use EBS. If the data is temporary, reproducible, or replicated at the application layer, Instance Store gives you the best performance at no additional cost (it is included in the instance price).

---

## 13. Questions & Answers

### Q1: Between block, file, object — which AWS services are related to each type?

| Storage Type | AWS Services |
|:------------|:-------------|
| Block | EBS (Elastic Block Store), EC2 Instance Store |
| File | EFS (Elastic File System), FSx (Windows File Server, Lustre, NetApp ONTAP, OpenZFS) |
| Object | S3 (Simple Storage Service), S3 Glacier |

---

### Q2: What's the difference between EBS and EC2 Instance Store?

Covered in full in [Section 12](#12-ebs-vs-ec2-instance-store) above. The essential difference: EBS persists across instance lifecycle events, Instance Store does not. Instance Store is faster because it is physically attached to the host hardware, but all data is lost on stop, terminate, or hardware failure.

---

### Q3: If you need a folder used across a number of instances, what service will you use?

**EFS** for Linux instances. All instances mount the same filesystem via NFS and see identical files. Changes made by one instance are immediately visible to all others.

**FSx for Windows File Server** if the instances run Windows, because EFS only supports NFS (Linux). FSx provides SMB protocol and Active Directory integration.

---

### Q4: How can you replicate an EBS volume?

EBS volumes cannot be live-replicated across AZs. The replication path is through **snapshots**:

1. Create a snapshot of the volume — this is stored incrementally in S3
2. Copy the snapshot to another AZ or region using `aws ec2 copy-snapshot`
3. Create a new EBS volume from the copied snapshot in the target AZ

For automated replication on a schedule, use **EBS Data Lifecycle Manager (DLM)** — it creates snapshots at defined intervals and manages retention (e.g., "snapshot every 12 hours, keep last 14 snapshots, copy to eu-west-1").

For real-time replication at the application level, you would use the database's own replication (e.g., PostgreSQL streaming replication, MySQL binlog replication) rather than EBS-level replication.

---

### Q5: You have static pages you want to expose to the Internet — what will you use?

**S3 Static Website Hosting.** Upload your HTML, CSS, JavaScript, and media files to an S3 bucket. Enable static website hosting in the bucket properties. Configure a bucket policy to allow public `GetObject` access. S3 serves the files directly over HTTP.

For production deployments, place **CloudFront** in front of the S3 bucket. CloudFront provides HTTPS (S3 website endpoints only support HTTP), caching at edge locations worldwide, and custom domain name support via Route 53.

---

### Q6: What are buckets in S3?

A bucket is the top-level container for objects in S3. Every object must live inside a bucket. Key properties:

- **Globally unique name** — no two buckets across all AWS accounts can share the same name
- **Region-scoped** — created in a specific region, data stays there unless replicated
- **Unlimited objects** — no cap on the number or total size of objects in a bucket
- **Flat namespace** — objects are identified by key, there are no real directories
- **Per-bucket configuration** — versioning, encryption, lifecycle rules, access policies, logging, replication are all configured at the bucket level

---

### Q7: What does S3 replication do? What is it for?

S3 replication automatically copies objects from a source bucket to one or more destination buckets. It operates asynchronously — there is a slight delay between the object being written to the source and appearing in the destination.

**Cross-Region Replication (CRR):**
- Copies objects to a bucket in a different AWS region
- Used for disaster recovery, compliance (data must exist in multiple geographies), and reducing latency for users in different regions

**Same-Region Replication (SRR):**
- Copies objects to another bucket in the same region
- Used for log aggregation from multiple source buckets, maintaining production/test copies, or cross-account replication within a region

Both types require **versioning enabled** on both the source and destination buckets. Replication only applies to new objects — existing objects at the time replication is enabled are not retroactively copied (you must use S3 Batch Replication for those).

---

### Q8: What are versions in S3? What does it mean to delete an object when versioning is enabled?

**Versioning** means S3 keeps every version of every object. When you overwrite an object, the previous version is not replaced — a new version is created alongside it. Each version has a unique Version ID.

**Deleting with versioning enabled:**

S3 does not physically remove the object. Instead, it inserts a **delete marker** — a zero-byte placeholder that tells S3 "this object should appear deleted." The result:

- A `GET` request for the object returns a 404 (because the delete marker hides it)
- All previous versions still exist in the bucket and are retrievable by specifying their Version ID
- To restore the object, you delete the delete marker
- To permanently delete a specific version, you must issue a `DELETE` with the explicit Version ID

```
Version history of "report.pdf":
────────────────────────────────────────────────────
  Version ID      Action              Status
  v-001           PUT (original)      Exists, accessible by version ID
  v-002           PUT (overwrite)     Exists, accessible by version ID
  v-003           PUT (overwrite)     Exists, accessible by version ID
  (delete marker) DELETE issued       report.pdf appears "deleted"
                                      but v-001, v-002, v-003 all remain
────────────────────────────────────────────────────
```

This is why versioning is critical for data protection — accidental deletions are recoverable.

---

### Q9: How is it possible to optimize cost of S3 resources?

1. **Lifecycle Rules** — automatically transition objects to cheaper storage classes as they age (Standard → Standard-IA after 30 days → Glacier after 90 days → delete after 365 days)
2. **S3 Intelligent-Tiering** — let AWS automatically move objects between access tiers based on usage patterns, with no retrieval fees
3. **Choose the right storage class upfront** — if you know data will be accessed infrequently, write it directly to Standard-IA or One Zone-IA
4. **Delete incomplete multipart uploads** — failed large uploads leave invisible fragments that accumulate cost. Create a lifecycle rule to abort incomplete multipart uploads after a defined period (e.g., 7 days)
5. **S3 Analytics** — enable storage class analysis to identify which objects could be moved to cheaper tiers
6. **Requester Pays** — shift data transfer costs to the entity downloading the data
7. **Compression** — compress objects (gzip, zstd) before uploading to reduce stored bytes
8. **Object expiration** — set lifecycle rules to automatically delete objects after their useful life
9. **S3 Storage Lens** — organization-wide visibility into storage usage and activity trends

---

### Q10: How are nested file hierarchies represented in S3?

They are not real hierarchies. S3 is a flat key-value store. What appears as folders in the AWS console is the result of using the `/` character as a **delimiter** in object keys.

The key `images/cats/photo1.jpg` is a single string — not a file `photo1.jpg` inside a directory `cats` inside a directory `images`. S3 has no directory concept, no inodes, no directory metadata. The console and CLI parse the `/` characters to present a visual hierarchy for human convenience.

Consequences of this design:
- "Renaming a folder" requires copying every object with the old prefix to a new key and deleting the originals
- "Listing a folder" is actually a `ListObjectsV2` API call with a `Prefix` parameter
- Empty "folders" in the console are zero-byte objects with a key ending in `/`
- There is no performance difference between deeply nested keys and flat keys

---

### Q11: What does S3 store inside its objects?

Each S3 object consists of:

| Component | Description |
|:----------|:------------|
| **Key** | The unique identifier within the bucket — the "name" of the object |
| **Value** | The actual data content — the file bytes, up to 5TB |
| **Version ID** | A system-assigned unique identifier for each version of the object (present when versioning is enabled) |
| **Metadata** | Key-value pairs. System metadata (content-type, content-length, last-modified) and user-defined metadata (prefixed with `x-amz-meta-`) |
| **Access Control** | Permissions defining who can access this specific object (legacy ACLs or bucket/IAM policies) |
| **Storage Class** | Which tier the object resides in (Standard, IA, Glacier, etc.) |

---

### Q12: Why is S3 better than a physically maintained file server?

| Concern | Physical File Server | S3 |
|:--------|:--------------------|:-----|
| **Durability** | Depends on your RAID configuration, backup frequency, and disaster recovery plan. A fire or flood destroys the server and all data. | 99.999999999% durability. Data is automatically replicated across at least 3 physically separated AZs. |
| **Scalability** | Adding capacity means purchasing, racking, and configuring new hard drives. Downtime during expansion. | Unlimited. No capacity planning. Just write data. |
| **Availability** | Single point of failure unless you build redundancy yourself (which multiplies cost and complexity). | Built-in multi-AZ redundancy. 99.99% availability for Standard class. |
| **Maintenance** | Operating system patches, disk replacements, RAID rebuilds, firmware updates, physical security, cooling, power. | Fully managed by AWS. Zero operational overhead. |
| **Cost structure** | Large upfront capital expenditure for hardware. Ongoing costs for power, cooling, staff, data center space. | Pay-per-use. No upfront cost. Scale down instantly by deleting data. |
| **Global access** | Requires VPN or dedicated connections for remote access. | HTTP API accessible from anywhere in the world. |
| **Disaster recovery** | Manual setup — off-site backup tapes, replication to another data center. | One-click cross-region replication. Automated lifecycle policies. |
| **Security** | Physical lock, network firewall, OS-level permissions. | Encryption at rest and in transit, IAM policies, bucket policies, VPC endpoints, access logging, versioning for recovery from accidental deletion. |

---

### Q13: How many ways to allow access to an S3 bucket do you know?

1. **IAM Policies** — identity-based policies attached to users, groups, or roles defining what S3 actions they can perform on which resources
2. **Bucket Policies** — resource-based JSON policies attached to the bucket itself, supporting cross-account access grants
3. **ACLs (Access Control Lists)** — legacy mechanism for per-bucket and per-object access control. AWS now recommends disabling ACLs on all new buckets.
4. **S3 Access Points** — named network endpoints with their own access policy, simplifying access management for shared datasets
5. **Pre-signed URLs** — time-limited URLs that grant temporary access to a specific object without requiring AWS credentials
6. **VPC Endpoints (Gateway type)** — restrict S3 access to traffic originating from a specific VPC, enforced via endpoint policies
7. **AWS Organizations Service Control Policies (SCPs)** — organization-level guardrails that restrict S3 access across all accounts in the organization

---

### Q14: Is it possible to attach an EBS volume to a few instances simultaneously?

Yes, but only under specific conditions:

- **Volume type must be io1 or io2** — general purpose (gp2/gp3) and HDD (st1/sc1) volumes do not support multi-attach
- **Maximum 16 instances** concurrently
- **All instances must be in the same AZ** as the volume
- **Cluster-aware filesystem required** — all attached instances must use a filesystem designed for concurrent multi-host access (GFS2, OCFS2). Using ext4 or XFS with multi-attach will cause data corruption because these filesystems assume exclusive access.

For most use cases requiring shared storage across instances, **EFS** is the appropriate solution. EBS Multi-Attach is designed for niche scenarios like clustered databases with their own distributed locking mechanisms.

---

### Q15: What is the use scenario for each Amazon FSx file system type?

| FSx Type | Scenario | Why Not EFS or Another FSx? |
|:---------|:---------|:----------------------------|
| **FSx for Windows File Server** | Corporate Windows file shares with Active Directory integration. SQL Server storage. .NET applications requiring SMB access. Home directories for Windows users. | EFS does not support SMB/Windows. This is the only AWS-managed option for native Windows file sharing. |
| **FSx for Lustre** | High-Performance Computing (HPC) — genomics, financial modeling, seismic analysis. Machine learning training on datasets of hundreds of TB. Video rendering and post-production pipelines. | Lustre provides orders-of-magnitude higher throughput than EFS for parallel workloads. It can also integrate directly with S3 as a backend, presenting S3 data as a POSIX filesystem. |
| **FSx for NetApp ONTAP** | Hybrid cloud environments migrating from on-premises NetApp storage. Applications that need both NFS and SMB access to the same data simultaneously. Multi-protocol workloads. | ONTAP supports NFS, SMB, and iSCSI on the same filesystem — neither EFS (NFS only) nor FSx for Windows (SMB only) can do this. It also provides features like data deduplication, compression, and SnapMirror replication familiar to NetApp administrators. |
| **FSx for OpenZFS** | Migrating on-premises ZFS-based storage to AWS. Development and test environments needing instant snapshots and clones. Low-latency file serving where ZFS features (checksums, compression, snapshots) are needed. | ZFS-specific features like block-level checksums, copy-on-write snapshots, and transparent compression are not available in EFS or other FSx types. |

---

## What's Next

```
Part 2:  EBS Deep Dive — volume types, snapshots, encryption, migration, monitoring, lifecycle
Part 3:  EFS Deep Dive — storage classes, access points, performance modes, serverless integration
Part 4:  S3 Deep Dive — tiers, lifecycle, replication, versioning, security, website hosting
Part 5:  Amazon FSx — variants, use cases, hands-on labs
```

---

## References

- [AWS EBS Documentation](https://docs.aws.amazon.com/ebs/)
- [AWS EFS Documentation](https://docs.aws.amazon.com/efs/)
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [AWS FSx Documentation](https://docs.aws.amazon.com/fsx/)
- [AWS Storage Services Overview](https://aws.amazon.com/products/storage/)
- [EBS Volume Types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
