# Part 4.1: S3 Fundamentals and Storage Classes

---

## Table of Contents

1. [What S3 Actually Is](#1-what-s3-actually-is)
2. [Core Architecture — Buckets and Objects](#2-core-architecture--buckets-and-objects)
3. [S3 Availability and Durability](#3-s3-availability-and-durability)
4. [Storage Classes Deep Dive](#4-storage-classes-deep-dive)
5. [S3 Intelligent-Tiering](#5-s3-intelligent-tiering)
6. [S3 Pricing Model and Cost Calculator](#6-s3-pricing-model-and-cost-calculator)
7. [Requestor Pays](#7-requestor-pays)
8. [Object Tagging](#8-object-tagging)
9. [S3 CLI Commands](#9-s3-cli-commands)
10. [Key Points to Remember](#10-key-points-to-remember)
11. [Self-Check Questions](#11-self-check-questions)

---

## 1. What S3 Actually Is

Amazon S3 (Simple Storage Service) is a fully managed **object storage** service with virtually unlimited capacity. Unlike EBS (a disk attached to one instance) or EFS (a shared filesystem mounted by many instances), S3 is accessed via **HTTP API** — not mounted, not attached. Any device that can make HTTPS requests can read or write objects to S3.

S3 is not a filesystem. There are no directories, no file handles, no mount points. You put an object in, you get it back by its key. It is a massive key-value store where the key is a string and the value is up to 5TB of data.

```
Traditional filesystem:                    S3:
─────────────────────────                  ───
  mount /dev/sdf /data                     PUT https://my-bucket.s3.amazonaws.com/logs/app.log
  open("/data/logs/app.log")               GET https://my-bucket.s3.amazonaws.com/logs/app.log
  read(fd, buffer, size)                   DELETE https://my-bucket.s3.amazonaws.com/logs/app.log
  close(fd)

  Requires: attached volume, mount,        Requires: network, credentials
  OS support, filesystem driver             Works from: anywhere (EC2, Lambda,
  Works from: that one instance               laptop, phone, another cloud)
```

### Why S3 dominates AWS storage

- **11 nines of durability** (99.999999999%) — designed to sustain the concurrent loss of data in two facilities
- **Unlimited capacity** — no provisioning, no resizing, no capacity planning
- **Global accessibility** — HTTP API means any service, language, or device can interact with it
- **Deep AWS integration** — serves as the backbone for CloudFront, Athena, EMR, Glue, Lambda, CloudTrail, and dozens more services
- **Cost effective** — starting at $0.023/GB/month for Standard, dropping to $0.00099/GB/month for Deep Archive

---

## 2. Core Architecture — Buckets and Objects

### Buckets

A bucket is the top-level container for objects. Think of it as a namespace — every object must belong to exactly one bucket.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Account                                  │
│                                                                     │
│  ┌─── Bucket: company-app-logs ───┐   ┌─── Bucket: company-assets ─┐│
│  │                                 │   │                            ││
│  │  Object: 2024/01/app.log       │   │  Object: css/style.css     ││
│  │  Object: 2024/01/error.log     │   │  Object: js/app.js         ││
│  │  Object: 2024/02/app.log       │   │  Object: img/logo.png      ││
│  │                                 │   │                            ││
│  │  Region: eu-west-1             │   │  Region: us-east-1         ││
│  └─────────────────────────────────┘   └────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

**Bucket rules:**

| Rule | Detail |
|:-----|:-------|
| Globally unique name | No two buckets across all AWS accounts can share a name |
| Region-scoped | Created in a specific region. Data stays there unless replicated. |
| 3-63 characters | Lowercase letters, numbers, hyphens only. Must start with letter or number. |
| 100 buckets per account (default) | Soft limit — can be increased via support request to 1,000 |
| Cannot be renamed | You must create a new bucket, copy data, delete the old one |
| Cannot be nested | Buckets cannot contain other buckets |

### Objects

An object is the fundamental entity stored in S3. Each object has:

| Component | Description | Example |
|:----------|:------------|:--------|
| **Key** | Unique identifier within the bucket | `logs/2024/march/app.log` |
| **Value** | The actual data (0 bytes to 5TB) | Binary content of the file |
| **Version ID** | Unique per version (when versioning enabled) | `3sL4kqtJlcpXroDTDmJ+rmSpXd3dIbrHY` |
| **Metadata** | System + user-defined key-value pairs | `Content-Type: application/json` |
| **Storage Class** | Which tier the object resides in | `STANDARD`, `GLACIER`, etc. |
| **Tags** | Up to 10 key-value pairs for categorization | `Environment=Production` |

### The flat namespace

S3 has no directories. The `/` in keys is a convention, not a filesystem separator.

```
What you see in the console:          What S3 actually stores:
────────────────────────────          ──────────────────────────
📁 images/                            Key: "images/cats/photo1.jpg"  → Value: <binary>
  📁 cats/                            Key: "images/dogs/puppy.jpg"   → Value: <binary>
    📄 photo1.jpg                     Key: "documents/report.pdf"    → Value: <binary>
  📁 dogs/
    📄 puppy.jpg                      These are 3 independent objects.
📁 documents/                         No hierarchy. No directories.
  📄 report.pdf                       "images/cats/photo1.jpg" is ONE string.
```

**Practical implications:**
- "Renaming a folder" = copy all objects with new prefix + delete originals (expensive for thousands of objects)
- "Listing a folder" = `ListObjectsV2` with a `Prefix` parameter
- Empty "folders" = zero-byte objects with keys ending in `/`
- No performance difference between deep or flat key structures

### Object size limits

| Upload Type | Max Size | When to Use |
|:------------|:---------|:------------|
| Single PUT | 5GB | Small files |
| Multipart Upload | 5TB (10,000 parts × 5GB each) | Files > 100MB (recommended), required > 5GB |

---

## 3. S3 Availability and Durability

These are two different guarantees that people often confuse.

### Durability — Will my data survive?

**99.999999999% (11 nines)** for all Standard storage classes. This means if you store 10,000,000 objects in S3, you can statistically expect to lose a single object once every 10,000 years.

How AWS achieves this:
- Every object is automatically replicated across **a minimum of 3 physically separated Availability Zones** within the region
- Each AZ is in a distinct facility with independent power, cooling, and networking
- S3 uses checksums to detect and repair any bit-level corruption automatically
- Data loss in one AZ is transparent — S3 re-replicates from surviving copies

```
When you PUT an object to S3:
─────────────────────────────────────────────────────────────────
                        PUT object
                           │
                           ▼
               ┌───────────────────────┐
               │   S3 receives object  │
               └───────────┬───────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         ┌────────┐  ┌────────┐  ┌────────┐
         │  AZ-1  │  │  AZ-2  │  │  AZ-3  │
         │ Copy 1 │  │ Copy 2 │  │ Copy 3 │
         └────────┘  └────────┘  └────────┘

S3 returns HTTP 200 ONLY after the object is
durably stored across multiple AZs.
─────────────────────────────────────────────────────────────────
```

### Availability — Can I access my data right now?

Availability varies by storage class because cheaper classes trade access guarantee for lower cost:

| Storage Class | Availability SLA | Designed For |
|:-------------|:-----------------|:-------------|
| S3 Standard | 99.99% | ~53 minutes of downtime per year |
| S3 Standard-IA | 99.9% | ~8.7 hours of downtime per year |
| S3 One Zone-IA | 99.5% | ~1.8 days of downtime per year |
| S3 Intelligent-Tiering | 99.9% | Varies by access tier |
| S3 Glacier Instant | 99.9% | Archive with instant access |

### Durability exception: One Zone classes

**S3 One Zone-IA** stores data in a **single AZ only**. Durability is still 99.999999999% within that AZ, but if the entire AZ is destroyed (fire, flood, earthquake), the data is gone. This is the only S3 class where physical AZ loss means data loss.

```
Standard (Multi-AZ):                    One Zone-IA:
──────────────────────                  ─────────────
  AZ-1: ✓ copy                           AZ-2: ✓ copy (only copy)
  AZ-2: ✓ copy                           AZ-1: ✗ no copy
  AZ-3: ✓ copy                           AZ-3: ✗ no copy

  AZ-1 destroyed → data survives        AZ-2 destroyed → DATA LOST
```

Use One Zone-IA only for data you can regenerate (thumbnails, transcoded media, replicated copies).

### Consistency Model

S3 provides **strong read-after-write consistency** for all operations since December 2020:
- PUT a new object → immediately readable
- Overwrite an object → immediately returns the new version
- DELETE an object → immediately returns 404
- List operations → immediately reflect changes

There is no eventual consistency lag. This simplifies application logic significantly.

---

## 4. Storage Classes Deep Dive

S3 offers seven storage classes. Choosing the right one is the single most impactful decision for S3 cost.

### Overview table

| Storage Class | AZs | Min Storage Duration | Retrieval Fee | Retrieval Time | Storage Cost (us-east-1) |
|:-------------|:----|:--------------------|:-------------|:--------------|:------------------------|
| **Standard** | ≥3 | None | None | Instant (ms) | $0.023/GB |
| **Intelligent-Tiering** | ≥3 | None | None | Instant (ms) | $0.023/GB (frequent tier) |
| **Standard-IA** | ≥3 | 30 days | Per-GB fee | Instant (ms) | $0.0125/GB |
| **One Zone-IA** | 1 | 30 days | Per-GB fee | Instant (ms) | $0.01/GB |
| **Glacier Instant** | ≥3 | 90 days | Per-GB fee | Instant (ms) | $0.004/GB |
| **Glacier Flexible** | ≥3 | 90 days | Per-GB fee | 1 min – 12 hrs | $0.0036/GB |
| **Glacier Deep Archive** | ≥3 | 180 days | Per-GB fee | 12 – 48 hrs | $0.00099/GB |

### When to use each class

```
Decision flow for choosing a storage class:
═══════════════════════════════════════════════════════════════════════

  Is the data accessed frequently (multiple times per month)?
  │
  ├── YES → S3 Standard
  │
  └── NO → How often is it accessed?
            │
            ├── Once a month or less, but needs instant access
            │   │
            │   ├── Critical data (multi-AZ needed)? → Standard-IA
            │   └── Non-critical (single AZ OK)?     → One Zone-IA
            │
            ├── Once a quarter, needs instant access → Glacier Instant Retrieval
            │
            ├── Once or twice a year, can wait minutes/hours → Glacier Flexible
            │
            └── Rarely/never, compliance/regulatory hold → Glacier Deep Archive
```

### Minimum storage duration charges

This is the cost trap most people miss. If you store an object in Standard-IA and delete it after 5 days, you are billed as if it existed for 30 days. The minimum duration charges:

```
Storage Class              Min Duration    What happens if you delete on Day 5?
─────────────────────      ────────────    ────────────────────────────────────
Standard                   None            Pay for 5 days only
Standard-IA               30 days         Pay for 30 days (25 days wasted)
One Zone-IA               30 days         Pay for 30 days
Glacier Instant           90 days         Pay for 90 days (85 days wasted!)
Glacier Flexible          90 days         Pay for 90 days
Glacier Deep Archive      180 days        Pay for 180 days (175 days wasted!)
```

### Minimum object size charges

Standard-IA, One Zone-IA, and Glacier classes charge for a minimum of **128KB** per object, even if the actual object is smaller. A 1KB file in Standard-IA is billed as 128KB.

This makes IA classes cost-ineffective for large numbers of tiny files (log lines, small JSON documents). Keep small frequently changing objects in Standard.

### Retrieval fees

Standard has no retrieval fee — you pay only for storage and requests. All other classes charge per-GB when you read data:

```
Reading 100GB from each class (approximate):
──────────────────────────────────────────────
  Standard:              $0.00  (free retrieval)
  Standard-IA:           $1.00  ($0.01/GB)
  One Zone-IA:           $1.00  ($0.01/GB)
  Glacier Instant:       $3.00  ($0.03/GB)
  Glacier Flexible:      $1.00 - $3.00 (depends on speed)
  Deep Archive:          $2.00 - $10.00 (depends on speed)
```

### Glacier Flexible retrieval options

When you request an object from Glacier Flexible, you choose a retrieval speed:

| Retrieval Tier | Time | Cost | When to Use |
|:---------------|:-----|:-----|:------------|
| Expedited | 1-5 minutes | Highest | Urgent need — disaster recovery, audit request |
| Standard | 3-5 hours | Moderate | Normal planned retrieval |
| Bulk | 5-12 hours | Lowest | Large-scale retrieval, not time-sensitive |

### Glacier Deep Archive retrieval options

| Retrieval Tier | Time | Cost |
|:---------------|:-----|:-----|
| Standard | 12 hours | Moderate |
| Bulk | 48 hours | Lowest |

No expedited option exists for Deep Archive.

---

## 5. S3 Intelligent-Tiering

Intelligent-Tiering deserves its own section because it is the **only storage class that requires no human decision-making about access patterns**. AWS monitors each object's access frequency and automatically moves it between tiers.

### How it works

```
Object uploaded to Intelligent-Tiering:
════════════════════════════════════════════════════════════════════

Day 1:   Object stored in Frequent Access tier ($0.023/GB)
         ↓ (no access for 30 days)
Day 31:  Auto-moved to Infrequent Access tier ($0.0125/GB)
         ↓ (no access for 90 days)
Day 121: Auto-moved to Archive Instant Access tier ($0.004/GB)
         ↓ (no access for 180 days)
Day 301: Auto-moved to Archive Access tier ($0.0036/GB)  [opt-in]
         ↓ (no access for 180 more days)
Day 481: Auto-moved to Deep Archive Access tier ($0.00099/GB) [opt-in]

At ANY point: object is accessed → immediately moved back to
Frequent Access tier. No retrieval fees. No delays.
════════════════════════════════════════════════════════════════════
```

### Tiers within Intelligent-Tiering

| Tier | Activation | Access Time | Cost |
|:-----|:-----------|:------------|:-----|
| Frequent Access | Automatic (default) | Instant | Same as Standard |
| Infrequent Access | Automatic (after 30 days) | Instant | Same as Standard-IA |
| Archive Instant Access | Automatic (after 90 days) | Instant | Same as Glacier Instant |
| Archive Access | **Opt-in** (after 90+ days) | 3-5 hours | Same as Glacier Flexible |
| Deep Archive Access | **Opt-in** (after 180+ days) | 12 hours | Same as Deep Archive |

### Costs

- No retrieval fees (unlike Standard-IA or Glacier)
- Small monitoring fee: $0.0025 per 1,000 objects per month
- No minimum storage duration charge

### When to use Intelligent-Tiering

| Scenario | Use Intelligent-Tiering? |
|:---------|:------------------------|
| Unknown access patterns | Yes — perfect fit |
| Mix of hot and cold data you cannot predict | Yes |
| Millions of tiny objects (< 128KB) | No — monitoring fee per object adds up, and objects < 128KB are always stored in Frequent tier |
| Data you know is always hot | No — Standard is simpler and identical cost |
| Data you know is always cold | No — choose the specific Glacier class directly to avoid monitoring fees |

---

## 6. S3 Pricing Model and Cost Calculator

### What you pay for

S3 pricing has four components. Most people only think about the first one and get surprised by the others.

| Component | What it Covers | How to Reduce |
|:----------|:--------------|:-------------|
| **Storage** | Per-GB/month for data at rest | Use lifecycle rules, choose cheaper classes |
| **Requests** | Per-request charges (PUT, GET, LIST, etc.) | Batch operations, reduce LIST calls |
| **Data Transfer** | Data leaving S3 to the internet or another region | Use CloudFront, keep traffic in same region |
| **Management** | Inventory, analytics, object tagging, replication | Enable only what you need |

### Request pricing (us-east-1)

| Operation | Standard | Standard-IA | Glacier Flexible |
|:----------|:---------|:------------|:-----------------|
| PUT, COPY, POST, LIST | $0.005 per 1,000 | $0.01 per 1,000 | $0.05 per 1,000 |
| GET, SELECT | $0.0004 per 1,000 | $0.001 per 1,000 | $0.0004 per 1,000 |
| DELETE | Free | Free | Free |
| Lifecycle transition | $0.01 per 1,000 | — | — |

### Data transfer pricing

| Direction | Cost |
|:----------|:-----|
| Data IN to S3 (upload) | Free |
| Data OUT to internet (first 100GB/month) | Free |
| Data OUT to internet (next 10TB) | $0.09/GB |
| Data OUT to same region (EC2, Lambda, etc.) | Free (via VPC endpoint or same region) |
| Data OUT to different region | $0.02/GB |
| Data OUT via CloudFront | Cheaper than direct S3 ($0.085/GB or less) |

### Cost calculation example

```
Scenario: Photo hosting service
─────────────────────────────────────────────────────────
  Storage: 5TB in Standard = 5,000 GB × $0.023     = $115.00/month
  Uploads: 500,000 PUTs = 500 × $0.005             = $2.50/month
  Downloads: 2,000,000 GETs = 2,000 × $0.0004      = $0.80/month
  Transfer out: 500 GB to internet = 500 × $0.09    = $45.00/month
                                                      ─────────────
  Total:                                              $163.30/month

  With lifecycle (move 3TB to Standard-IA after 30 days):
  Storage: 2TB Standard + 3TB Standard-IA
           = (2,000 × $0.023) + (3,000 × $0.0125)  = $83.50/month
  Savings: $31.50/month on storage alone
─────────────────────────────────────────────────────────
```

### AWS Pricing Calculator

AWS provides a free tool to estimate S3 costs before committing:
- URL: https://calculator.aws/
- Select S3 → enter your expected storage, requests, and transfer
- Compare storage class options side by side
- Export estimates as PDF for management approval

---

## 7. Requestor Pays

By default, the bucket owner pays for all storage and data transfer costs. **Requestor Pays** shifts the data transfer cost to the requester (the person downloading the data).

### How it works

```
Normal bucket:                          Requestor Pays bucket:
──────────────                          ──────────────────────
  Requester downloads 100GB             Requester downloads 100GB
  Bucket owner pays:                    Bucket owner pays:
    Storage: owner                        Storage: owner
    Transfer: owner ($9.00)               Transfer: requester ($9.00)

  Requester pays: $0                    Requester pays: $9.00
```

### When to use Requestor Pays

| Scenario | Why |
|:---------|:----|
| Sharing large datasets (genomics, satellite imagery, public research) | You cannot afford $10,000/month in transfer for data others download |
| Cross-account data sharing | Partner accounts should pay for their own usage |
| Open data programs | Make data available without unlimited cost exposure |

### How to enable

```bash
aws s3api put-bucket-request-payment \
    --bucket my-dataset-bucket \
    --request-payment-configuration Payer=Requester
```

### Requester requirements

- Requests must include `x-amz-request-payer: requester` header
- Anonymous (unauthenticated) access is not allowed on Requestor Pays buckets
- The requester must have an AWS account (charges need somewhere to go)

```bash
# Downloading from a Requestor Pays bucket:
aws s3 cp s3://public-dataset-bucket/data.csv ./data.csv --request-payer requester
```

---

## 8. Object Tagging

Tags are key-value pairs attached to individual objects. They enable fine-grained control over lifecycle, access, and cost allocation that key prefixes alone cannot provide.

### Tag basics

| Property | Detail |
|:---------|:-------|
| Max tags per object | 10 |
| Key max length | 128 characters |
| Value max length | 256 characters |
| Case sensitive | Yes — `Environment` ≠ `environment` |
| Can be added/modified after upload | Yes — without re-uploading the object |

### Use cases for tags

```
Object: reports/quarterly/q4-2024-financial.pdf
Tags:
────────────────────────────────────
  Department = Finance
  Classification = Confidential
  RetentionPolicy = 7years
  CostCenter = CC-4501
  Project = AnnualReport
────────────────────────────────────
```

**1. Lifecycle rules based on tags:**

```
Rule: Delete objects tagged "RetentionPolicy=90days" after 90 days
Rule: Transition objects tagged "Archive=true" to Glacier after 30 days
```

This is more flexible than prefix-based rules. Objects in the same "folder" can have different lifecycle behaviors.

**2. Access control based on tags:**

```json
{
  "Effect": "Deny",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringEquals": {
      "s3:ExistingObjectTag/Classification": "Confidential"
    }
  }
}
```

This denies access to any object tagged `Classification=Confidential`, regardless of its key/prefix.

**3. Cost allocation:**

Tags flow into AWS Cost Explorer and billing reports, letting you answer "how much S3 storage is the Marketing team using?" without maintaining separate buckets per team.

### Tagging via CLI

```bash
# Add tags during upload
aws s3api put-object \
    --bucket my-bucket \
    --key reports/q4.pdf \
    --body ./q4.pdf \
    --tagging "Department=Finance&Classification=Internal"

# Add/modify tags on existing object
aws s3api put-object-tagging \
    --bucket my-bucket \
    --key reports/q4.pdf \
    --tagging '{"TagSet": [{"Key": "Department", "Value": "Finance"}, {"Key": "Archive", "Value": "true"}]}'

# Get tags
aws s3api get-object-tagging --bucket my-bucket --key reports/q4.pdf

# Delete all tags
aws s3api delete-object-tagging --bucket my-bucket --key reports/q4.pdf
```

---

## 9. S3 CLI Commands

The AWS CLI provides two command groups for S3:
- **`aws s3`** — high-level commands (simplified, user-friendly)
- **`aws s3api`** — low-level API commands (full control, maps 1:1 to S3 API)

### High-level commands (`aws s3`)

These are the commands you use 90% of the time for daily operations.

#### Bucket operations

```bash
# List all buckets
aws s3 ls

# Create a bucket
aws s3 mb s3://my-new-bucket-unique-name

# Create a bucket in a specific region
aws s3 mb s3://my-eu-bucket --region eu-west-1

# Delete an empty bucket
aws s3 rb s3://my-empty-bucket

# Delete a bucket and ALL its contents (dangerous!)
aws s3 rb s3://my-bucket --force
```

#### Object operations

```bash
# List objects in a bucket
aws s3 ls s3://my-bucket/

# List objects recursively (all "subdirectories")
aws s3 ls s3://my-bucket/ --recursive

# List with human-readable sizes
aws s3 ls s3://my-bucket/ --recursive --human-readable --summarize

# Upload a file
aws s3 cp ./local-file.txt s3://my-bucket/path/file.txt

# Download a file
aws s3 cp s3://my-bucket/path/file.txt ./local-file.txt

# Copy between buckets
aws s3 cp s3://source-bucket/file.txt s3://dest-bucket/file.txt

# Upload with specific storage class
aws s3 cp ./archive.zip s3://my-bucket/archive.zip --storage-class STANDARD_IA

# Upload with encryption
aws s3 cp ./secret.txt s3://my-bucket/secret.txt --sse aws:kms --sse-kms-key-id alias/my-key
```

#### Sync operations

```bash
# Sync local directory to S3 (upload only changes)
aws s3 sync ./my-website/ s3://my-bucket/

# Sync S3 to local (download only changes)
aws s3 sync s3://my-bucket/ ./local-backup/

# Sync and delete files that no longer exist in source
aws s3 sync ./my-website/ s3://my-bucket/ --delete

# Sync with exclusions
aws s3 sync ./project/ s3://my-bucket/ --exclude "*.tmp" --exclude ".git/*"

# Sync only specific file types
aws s3 sync ./images/ s3://my-bucket/images/ --include "*.jpg" --exclude "*"

# Dry run (see what would happen without making changes)
aws s3 sync ./my-website/ s3://my-bucket/ --dryrun
```

#### Move and delete

```bash
# Move (rename) an object
aws s3 mv s3://my-bucket/old-name.txt s3://my-bucket/new-name.txt

# Move between buckets
aws s3 mv s3://source-bucket/file.txt s3://dest-bucket/file.txt

# Delete a single object
aws s3 rm s3://my-bucket/file.txt

# Delete all objects with a prefix
aws s3 rm s3://my-bucket/logs/ --recursive

# Delete all objects in a bucket
aws s3 rm s3://my-bucket/ --recursive
```

### Low-level commands (`aws s3api`)

Use these when you need full API control — metadata, tags, versions, policies.

```bash
# Get bucket location
aws s3api get-bucket-location --bucket my-bucket

# Get bucket versioning status
aws s3api get-bucket-versioning --bucket my-bucket

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket my-bucket \
    --versioning-configuration Status=Enabled

# Head object (metadata without downloading)
aws s3api head-object --bucket my-bucket --key path/file.txt

# List object versions
aws s3api list-object-versions --bucket my-bucket --prefix path/

# Get bucket policy
aws s3api get-bucket-policy --bucket my-bucket

# Put bucket policy
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json

# Generate pre-signed URL (expires in 1 hour)
aws s3 presign s3://my-bucket/private-file.pdf --expires-in 3600

# Create multipart upload (for large files)
aws s3api create-multipart-upload --bucket my-bucket --key large-file.zip
```

### Useful options

| Option | Purpose | Example |
|:-------|:--------|:--------|
| `--recursive` | Apply to all objects under prefix | `aws s3 rm s3://bucket/logs/ --recursive` |
| `--dryrun` | Show what would happen without executing | `aws s3 sync ./ s3://bucket/ --dryrun` |
| `--exclude` | Skip files matching pattern | `--exclude "*.log"` |
| `--include` | Include files matching pattern | `--include "*.jpg"` |
| `--storage-class` | Set storage class on upload | `--storage-class GLACIER` |
| `--sse` | Enable server-side encryption | `--sse AES256` or `--sse aws:kms` |
| `--acl` | Set canned ACL | `--acl public-read` |
| `--content-type` | Set content-type metadata | `--content-type "text/html"` |
| `--expires` | Set expiration date | `--expires 2025-01-01T00:00:00Z` |

---

## 10. Key Points to Remember

1. **S3 is object storage, not a filesystem.** No mounting, no directories. HTTP API access only. The `/` in keys is a convention, not a path separator.

2. **11 nines durability = practically indestructible.** Data is replicated across ≥3 AZs automatically. The only exception is One Zone-IA (single AZ).

3. **Strong read-after-write consistency.** No eventual consistency delays. PUT → immediately readable. DELETE → immediately returns 404.

4. **Minimum storage duration charges are real.** Deleting a Glacier object after 1 day still costs you for 90 days. Factor this into lifecycle rules.

5. **Minimum object size charges (128KB) for IA/Glacier.** Thousands of tiny files in Standard-IA will cost more than keeping them in Standard.

6. **Data transfer IN is free. Data transfer OUT costs money.** The biggest surprise on S3 bills is usually egress charges, not storage.

7. **Intelligent-Tiering eliminates guesswork** but has a per-object monitoring fee. Best for unpredictable access patterns; unnecessary for data you already understand.

8. **Bucket names are globally unique.** Once someone takes `my-app-logs`, no one else in any AWS account can use it. Choose descriptive, organization-prefixed names.

9. **`aws s3 sync` is the workhorse command.** It only transfers what changed (delta sync), handles retries, and supports exclusions. Use it for deployments and backups.

10. **Requestor Pays shifts transfer costs to downloaders.** Essential when sharing large datasets publicly without absorbing unlimited egress bills.

---

## 11. Self-Check Questions

1. You store 1 million objects in S3 Standard. How many objects would you statistically lose over 10,000 years?
   > One. 11 nines durability (99.999999999%) means for 10 million objects, you expect to lose 1 object per 10,000 years.

2. You upload an object to Standard-IA and delete it after 10 days. How many days are you billed for?
   > 30 days. Standard-IA has a minimum storage duration of 30 days. You pay for the full 30 days regardless of when you delete.

3. You have 50,000 objects averaging 500 bytes each. Should you use Standard-IA to save money?
   > No. Standard-IA has a minimum billable object size of 128KB. Each 500-byte object would be billed as 128KB, making the effective cost far higher than Standard. Keep small objects in Standard.

4. Your S3 bill shows $500 in data transfer but only $50 in storage. What should you investigate?
   > You are likely serving data directly from S3 to the internet (egress at $0.09/GB). Placing CloudFront in front of S3 reduces transfer costs (CloudFront egress is cheaper), adds caching (fewer S3 requests), and improves latency. Also check for unnecessary cross-region transfers.

5. Can two different AWS accounts have a bucket with the same name?
   > No. Bucket names are globally unique across all AWS accounts worldwide. If account A owns `data-lake`, no other account can create a bucket with that name.

6. What is the difference between `aws s3 cp` and `aws s3 sync`?
   > `cp` copies the specified file(s) unconditionally every time. `sync` compares source and destination and only transfers files that are new or modified (delta sync). For recurring uploads (like deploying a website), `sync` is far more efficient.

7. You need to share a 10TB genomics dataset publicly. You cannot afford the transfer costs. What feature helps?
   > Enable Requestor Pays on the bucket. Users downloading the data pay the transfer costs from their own AWS accounts. Anonymous access is not allowed — requesters must have AWS accounts.

---

## References

- [What is Amazon S3?](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [S3 Pricing](https://aws.amazon.com/s3/pricing/)
- [S3 Intelligent-Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/)
- [AWS CLI S3 Commands](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [AWS CLI S3API Commands](https://docs.aws.amazon.com/cli/latest/reference/s3api/)
- [S3 Data Consistency Model](https://aws.amazon.com/s3/consistency/)
- [S3 Object Tagging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-tagging.html)
- [AWS Pricing Calculator](https://calculator.aws/)
