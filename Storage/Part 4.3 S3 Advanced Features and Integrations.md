# Part 4.3: S3 Advanced Features and Integrations

---

## Table of Contents

1. [Versioning Deep Dive](#1-versioning-deep-dive)
2. [Lifecycle Policies](#2-lifecycle-policies)
3. [S3 Replication](#3-s3-replication)
4. [Transfer Acceleration](#4-transfer-acceleration)
5. [Multipart Upload](#5-multipart-upload)
6. [S3 Events and Notifications](#6-s3-events-and-notifications)
7. [Static Website Hosting](#7-static-website-hosting)
8. [S3 with CloudFront](#8-s3-with-cloudfront)
9. [S3 Select and Glacier Select](#9-s3-select-and-glacier-select)
10. [S3 Object Lock and MFA Delete](#10-s3-object-lock-and-mfa-delete)
11. [Integration with Other AWS Services](#11-integration-with-other-aws-services)
12. [Hands-On: Complete S3 Workflow](#12-hands-on-complete-s3-workflow)
13. [Key Points to Remember](#13-key-points-to-remember)
14. [Self-Check Questions](#14-self-check-questions)

---

## 1. Versioning Deep Dive

Versioning keeps every version of every object in a bucket. Every PUT (upload/overwrite) creates a new version instead of replacing the previous one. Every DELETE inserts a delete marker instead of physically removing the object.

### Versioning states

A bucket can be in one of three states:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BUCKET VERSIONING STATES                          │
│                                                                     │
│  1. Unversioned (default)                                           │
│     └── Objects have null version ID                                │
│     └── Overwrite replaces the object                               │
│     └── Delete permanently removes the object                       │
│                                                                     │
│  2. Enabled                                                         │
│     └── Every object gets a unique version ID                       │
│     └── Overwrite creates a new version (old versions remain)       │
│     └── Delete adds a delete marker (old versions remain)           │
│                                                                     │
│  3. Suspended                                                       │
│     └── Existing versions remain (not deleted)                      │
│     └── New objects get version ID = "null"                         │
│     └── Overwriting creates a new "null" version                    │
│     └── Cannot go back to "Unversioned" — only Enabled or Suspended │
└─────────────────────────────────────────────────────────────────────┘
```

Once versioning is enabled on a bucket, it **cannot be returned to unversioned**. You can only suspend it.

### How versioning works in practice

```
Timeline of object "config.json" in a versioned bucket:
═══════════════════════════════════════════════════════════════════════

  Action                  Version ID          What's stored
  ──────                  ──────────          ─────────────
  PUT config.json         v1 (abc123)         {"env": "dev"}
  PUT config.json         v2 (def456)         {"env": "staging"}
  PUT config.json         v3 (ghi789)         {"env": "production"}
  DELETE config.json      (delete marker)     ← hides v3

  GET config.json         → 404 (delete marker is "current")
  GET config.json?versionId=ghi789  → {"env": "production"}
  GET config.json?versionId=abc123  → {"env": "dev"}

  DELETE delete-marker    → config.json is "undeleted" (v3 becomes current)
  GET config.json         → {"env": "production"}

═══════════════════════════════════════════════════════════════════════
```

### Permanently deleting a version

```bash
# This adds a delete marker (recoverable)
aws s3 rm s3://my-bucket/config.json

# This permanently deletes a specific version (IRREVERSIBLE)
aws s3api delete-object \
    --bucket my-bucket \
    --key config.json \
    --version-id abc123
```

### Cost implications of versioning

Every version is a full copy of the object and is billed as stored data. If you overwrite a 1GB file 10 times, you are storing 10GB (not 1GB).

```
Object: database-backup.sql (1GB)
Versions stored: 30 daily backups

Storage cost WITHOUT versioning:  1 GB  × $0.023 = $0.023/month
Storage cost WITH versioning:     30 GB × $0.023 = $0.69/month

Solution: Use lifecycle rules to expire old versions
  (e.g., delete versions older than 30 days)
```

### Listing and managing versions

```bash
# List all versions of all objects
aws s3api list-object-versions --bucket my-bucket

# List versions of a specific object
aws s3api list-object-versions --bucket my-bucket --prefix config.json

# List delete markers
aws s3api list-object-versions --bucket my-bucket --key-marker "" --version-id-marker ""

# Restore a deleted object (remove the delete marker)
aws s3api delete-object \
    --bucket my-bucket \
    --key config.json \
    --version-id "delete-marker-version-id"
```

---

## 2. Lifecycle Policies

Lifecycle policies automate two things: **transitioning** objects between storage classes and **expiring** (deleting) objects. They are the primary tool for S3 cost optimization.

### Rule structure

```
Lifecycle Rule:
────────────────────────────────────────────────────
  Scope:       Which objects (prefix, tags, or entire bucket)
  Transitions: Move to cheaper classes after N days
  Expiration:  Delete objects after N days
  Version cleanup: Delete old versions after N days
  Abort incomplete uploads: Clean up failed multipart uploads
────────────────────────────────────────────────────
```

### Example: Full lifecycle policy

```json
{
  "Rules": [
    {
      "ID": "TransitionAndExpire",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 730
      },
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "STANDARD_IA"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

### Visual timeline of this policy

```
Object uploaded to logs/ prefix:
═══════════════════════════════════════════════════════════════════════════

Day 0        Day 30         Day 90           Day 365         Day 730
  │            │              │                │               │
  ▼            ▼              ▼                ▼               ▼
┌────────┐  ┌──────────┐  ┌─────────────┐  ┌────────────┐  ┌────────┐
│Standard│→ │Standard-IA│→ │   Glacier   │→ │Deep Archive│→ │DELETED │
│$0.023  │  │ $0.0125  │  │  $0.0036    │  │ $0.00099   │  │  $0    │
└────────┘  └──────────┘  └─────────────┘  └────────────┘  └────────┘

Old versions (when versioning is enabled):
  Day 0 (becomes noncurrent)    Day 30         Day 90
  │                              │              │
  ▼                              ▼              ▼
┌────────┐                    ┌──────────┐  ┌────────┐
│Standard│                 →  │Standard-IA│→ │DELETED │
└────────┘                    └──────────┘  └────────┘

═══════════════════════════════════════════════════════════════════════════
```

### Transition constraints

Not all transitions are allowed. Objects can only move from warmer to colder tiers:

```
Standard → Standard-IA → Intelligent-Tiering → One Zone-IA → Glacier Instant
    ↓           ↓              ↓                    ↓              ↓
    └───────────┴──────────────┴────────────────────┴──────────────┘
                                                                   ↓
                                                          Glacier Flexible
                                                                   ↓
                                                          Deep Archive

You CANNOT transition:
  ✗ Glacier → Standard (must restore, then copy)
  ✗ Standard-IA → Standard (not via lifecycle)
  ✗ Any class → Intelligent-Tiering → Standard (IT handles this internally)
```

### Minimum days between transitions

| From | To | Minimum Days |
|:-----|:---|:-------------|
| Standard | Standard-IA | 30 |
| Standard | One Zone-IA | 30 |
| Standard | Glacier Instant | 90 |
| Standard-IA | Glacier Flexible | 30 (from IA creation) |
| Any | Deep Archive | 90 (from previous transition) |

### Filter types

```json
// By prefix (all objects under "logs/")
"Filter": { "Prefix": "logs/" }

// By tag
"Filter": { "Tag": { "Key": "Archive", "Value": "true" } }

// By prefix AND tag (both must match)
"Filter": {
  "And": {
    "Prefix": "data/",
    "Tags": [{ "Key": "Department", "Value": "Finance" }]
  }
}

// By minimum object size (ignore small files)
"Filter": {
  "And": {
    "Prefix": "",
    "ObjectSizeGreaterThan": 131072
  }
}

// Entire bucket (empty filter)
"Filter": {}
```

### Applying lifecycle rules

```bash
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-bucket \
    --lifecycle-configuration file://lifecycle.json
```

### The abort incomplete multipart upload rule

Failed multipart uploads leave invisible parts in your bucket that incur storage charges. This rule automatically cleans them up:

```json
{
  "Rules": [{
    "ID": "CleanupIncompleteUploads",
    "Status": "Enabled",
    "Filter": {},
    "AbortIncompleteMultipartUpload": {
      "DaysAfterInitiation": 7
    }
  }]
}
```

Every bucket should have this rule. There is no downside — incomplete uploads older than 7 days are almost certainly abandoned.

---

## 3. S3 Replication

Replication automatically copies objects from a source bucket to one or more destination buckets. It operates asynchronously — there is a short delay between the object appearing in the source and the replica appearing in the destination.

### Replication types

| Type | Source and Destination | Use Case |
|:-----|:----------------------|:---------|
| **CRR** (Cross-Region Replication) | Different regions | Disaster recovery, latency reduction for global users, compliance |
| **SRR** (Same-Region Replication) | Same region | Log aggregation, prod/test copies, cross-account within region |

### Prerequisites

1. **Versioning must be enabled** on both source and destination buckets
2. **IAM role** with permission to read from source and write to destination
3. Buckets can be in the same or different AWS accounts
4. Source and destination cannot be the same bucket

### Replication architecture

```
Cross-Region Replication (CRR):
══════════════════════════════════════════════════════════════

  Region: us-east-1                    Region: eu-west-1
  ┌──────────────────────┐             ┌──────────────────────┐
  │  Source Bucket       │             │  Destination Bucket  │
  │  (versioning: ON)    │  ──────────>│  (versioning: ON)    │
  │                      │  async copy │                      │
  │  PUT object ─┐       │             │  Object appears      │
  │              │       │             │  (seconds to minutes) │
  └──────────────┼───────┘             └──────────────────────┘
                 │
                 └── S3 Replication reads the object
                     and writes it to the destination
                     using the configured IAM role

══════════════════════════════════════════════════════════════
```

### What IS replicated

- New objects created after replication is enabled
- Object metadata and tags
- Object ACLs (if configured)
- S3 Object Lock settings
- Objects encrypted with SSE-S3 or SSE-KMS (with proper configuration)

### What is NOT replicated

- Objects that existed before replication was enabled (use S3 Batch Replication for these)
- Objects in the Glacier or Deep Archive storage class
- Delete markers (by default — can be enabled)
- Deletions of specific versions (permanent deletes are never replicated for safety)
- Objects that are themselves replicas (no chaining: A→B→C)
- Objects encrypted with SSE-C

### Replication configuration

```json
{
  "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
  "Rules": [
    {
      "ID": "ReplicateAll",
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::destination-bucket",
        "StorageClass": "STANDARD_IA",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": { "Minutes": 15 }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": { "Minutes": 15 }
        }
      },
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      }
    }
  ]
}
```

### Replication Time Control (RTC)

Standard replication has no SLA on timing — most objects replicate in seconds, but some can take hours. **Replication Time Control** guarantees 99.99% of objects replicate within 15 minutes.

- Extra cost (~$0.015/GB replicated)
- Includes CloudWatch metrics for monitoring replication lag
- Use when you have compliance requirements for RPO (Recovery Point Objective)

### S3 Batch Replication

For objects that existed before replication was enabled:

```bash
# Create a batch replication job for existing objects
aws s3control create-job \
    --account-id 123456789012 \
    --operation '{"S3ReplicateObject": {}}' \
    --manifest '{"Spec": {"Format": "S3InventoryReport_CSV_20211130"}, "Location": {"..."}}'
```

---

## 4. Transfer Acceleration

Transfer Acceleration uses **CloudFront edge locations** to speed up uploads and downloads to/from S3 over long distances. Instead of routing your data across the public internet all the way to the S3 bucket's region, the data reaches the nearest edge location first and then travels over AWS's optimized backbone network.

### How it works

```
Without Transfer Acceleration:
══════════════════════════════════════════════════════════════
  User in Sydney → Public internet (many hops) → S3 in us-east-1
  Latency: High, variable, packet loss across ocean links

With Transfer Acceleration:
══════════════════════════════════════════════════════════════
  User in Sydney → Nearest edge (Sydney) → AWS backbone → S3 in us-east-1
  Latency: Lower, consistent, optimized network path

  ┌─────────┐          ┌───────────────┐         ┌────────────┐
  │  Client │ ──────── │  Edge Location │ ═══════ │  S3 Bucket │
  │ (Sydney)│  short   │  (Sydney)     │  AWS     │ (us-east-1)│
  └─────────┘  hop     └───────────────┘  private └────────────┘
                                           network
               Public internet             (optimized, low-loss)
               (short distance)
══════════════════════════════════════════════════════════════
```

### When Transfer Acceleration helps

| Scenario | Benefit |
|:---------|:--------|
| Users uploading from distant regions (e.g., Asia to us-east-1) | 50-500% faster uploads |
| Large file transfers across continents | Consistent throughput, fewer failures |
| Global user base uploading to a single bucket | Nearest edge location gives each user low latency |
| Aggregating data from worldwide sources | All edge locations feed into one bucket efficiently |

### When Transfer Acceleration does NOT help

- Users already in the same region as the bucket (adds overhead, no benefit)
- Small files (< 1MB) where connection setup dominates
- Downloads that should use CloudFront directly (CloudFront is better for repeated reads)

### Enabling Transfer Acceleration

```bash
# Enable on the bucket
aws s3api put-bucket-accelerate-configuration \
    --bucket my-global-uploads \
    --accelerate-configuration Status=Enabled
```

### Using Transfer Acceleration

Once enabled, use the accelerated endpoint instead of the standard one:

```
Standard endpoint:       my-bucket.s3.amazonaws.com
Accelerated endpoint:    my-bucket.s3-accelerate.amazonaws.com
```

```bash
# Upload using accelerated endpoint
aws s3 cp ./large-file.zip s3://my-global-uploads/large-file.zip \
    --endpoint-url https://my-global-uploads.s3-accelerate.amazonaws.com

# Or configure the CLI to use acceleration
aws configure set s3.use_accelerate_endpoint true
aws s3 cp ./large-file.zip s3://my-global-uploads/large-file.zip
```

### Speed comparison tool

AWS provides a tool to test whether acceleration benefits you:
`https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html`

### Pricing

- Additional $0.04-$0.08/GB on top of normal transfer costs
- **You are only charged when acceleration actually improves performance.** If the accelerated path is not faster than the standard path, the transfer uses the standard path and you are not charged the acceleration fee.

---

## 5. Multipart Upload

Multipart Upload is how S3 handles large files. Instead of uploading a 5GB file in one request (which would fail if the network drops for a second), you split it into parts and upload them independently.

### How it works

```
Large file (5GB):
═══════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────┐
  │                5GB file on disk                      │
  └──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬─┘
     │      │      │      │      │      │      │      │
     ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
   Part 1 Part 2 Part 3 Part 4 Part 5 Part 6 Part 7 Part 8
   (100MB each, uploaded independently, can be parallel)
     │      │      │      │      │      │      │      │
     └──────┴──────┴──────┴──────┴──────┴──────┴──────┘
                              │
                              ▼
                     Complete Multipart Upload
                     (S3 assembles parts into final object)

═══════════════════════════════════════════════════════════════════
```

### When to use multipart upload

| File Size | Recommendation |
|:----------|:---------------|
| < 100MB | Single PUT is fine |
| 100MB - 5GB | Multipart recommended (better resilience) |
| > 5GB | Multipart **required** (single PUT limit is 5GB) |

### Key benefits

- **Parallel uploads** — upload 10 parts simultaneously for higher throughput
- **Retry individual parts** — if part 7 fails, re-upload only part 7, not the entire file
- **Pause and resume** — upload some parts now, rest later
- **Begin before file size is known** — start uploading while still generating data

### Part constraints

| Property | Limit |
|:---------|:------|
| Minimum part size | 5MB (except the last part) |
| Maximum part size | 5GB |
| Maximum number of parts | 10,000 |
| Maximum object size | 5TB (10,000 × 5GB theoretically, but 5TB is the hard limit) |

### CLI handles it automatically

The AWS CLI automatically uses multipart upload for files above a threshold:

```bash
# Files > 8MB are automatically uploaded as multipart (default threshold)
aws s3 cp ./10gb-file.zip s3://my-bucket/10gb-file.zip

# Configure multipart threshold and chunk size
aws configure set s3.multipart_threshold 64MB
aws configure set s3.multipart_chunksize 64MB

# Control max concurrent uploads
aws configure set s3.max_concurrent_requests 20
```

### Manual multipart upload (API level)

```bash
# Step 1: Initiate
UPLOAD_ID=$(aws s3api create-multipart-upload \
    --bucket my-bucket \
    --key large-file.zip \
    --query 'UploadId' --output text)

# Step 2: Upload parts
aws s3api upload-part \
    --bucket my-bucket \
    --key large-file.zip \
    --part-number 1 \
    --body ./part1.bin \
    --upload-id $UPLOAD_ID

# Step 3: Complete (after all parts uploaded)
aws s3api complete-multipart-upload \
    --bucket my-bucket \
    --key large-file.zip \
    --upload-id $UPLOAD_ID \
    --multipart-upload '{"Parts": [{"PartNumber": 1, "ETag": "\"etag1\""}, ...]}'

# Or abort (cancel and clean up)
aws s3api abort-multipart-upload \
    --bucket my-bucket \
    --key large-file.zip \
    --upload-id $UPLOAD_ID
```

---

## 6. S3 Events and Notifications

S3 can trigger actions automatically when events occur in a bucket. This enables event-driven architectures where uploads, deletions, or modifications kick off processing pipelines without polling.

### Supported event types

| Event Type | Triggers When |
|:-----------|:--------------|
| `s3:ObjectCreated:*` | Any object is created (PUT, POST, COPY, multipart complete) |
| `s3:ObjectCreated:Put` | Specifically a PUT request |
| `s3:ObjectCreated:Post` | Specifically a POST request |
| `s3:ObjectCreated:Copy` | Specifically a COPY |
| `s3:ObjectCreated:CompleteMultipartUpload` | Multipart upload completes |
| `s3:ObjectRemoved:*` | Any deletion occurs |
| `s3:ObjectRemoved:Delete` | Object is deleted |
| `s3:ObjectRemoved:DeleteMarkerCreated` | Delete marker added (versioned bucket) |
| `s3:ObjectRestore:Post` | Glacier restore initiated |
| `s3:ObjectRestore:Completed` | Glacier restore completed |
| `s3:Replication:*` | Replication events |
| `s3:LifecycleTransition` | Object transitions to another storage class |
| `s3:ObjectTagging:*` | Tags are added or deleted |

### Notification destinations

```
S3 Event occurs
      │
      ├──────────────────────── Amazon SNS (Simple Notification Service)
      │                         → Email, SMS, HTTP endpoints, other subscribers
      │
      ├──────────────────────── Amazon SQS (Simple Queue Service)
      │                         → Queue for asynchronous processing
      │
      ├──────────────────────── AWS Lambda
      │                         → Run code in response to the event
      │
      └──────────────────────── Amazon EventBridge
                                → Route to 20+ AWS services, apply rules/filters
```

### Common use cases

```
┌─────────────────────────────────────────────────────────────────────────┐
│  USE CASE: Image Processing Pipeline                                    │
│                                                                         │
│  User uploads image → S3 event → Lambda function → creates thumbnail   │
│                                    → stores thumbnail in another prefix  │
│                                    → updates DynamoDB metadata           │
│                                                                         │
│  ┌────────┐    s3:ObjectCreated    ┌──────────┐    PUT thumbnail       │
│  │  User  │ ── PUT image ──> S3 ──>│  Lambda  │ ── ──────────────> S3  │
│  └────────┘                        └────┬─────┘                        │
│                                         │                               │
│                                         └── Update metadata ──> DynamoDB│
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  USE CASE: Log Processing                                               │
│                                                                         │
│  Application writes logs to S3 → Event → SQS → Processing service      │
│                                                                         │
│  Benefit: Decoupled. Log producers don't know about consumers.          │
│  The queue buffers bursts. Multiple consumers can process in parallel.  │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  USE CASE: Data Lake Ingestion                                          │
│                                                                         │
│  New data file lands in S3 → EventBridge → Step Functions workflow      │
│  → Validate schema → Transform → Load to Redshift                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Configuring S3 event notifications

```bash
aws s3api put-bucket-notification-configuration \
    --bucket my-uploads-bucket \
    --notification-configuration '{
      "LambdaFunctionConfigurations": [
        {
          "Id": "ProcessNewImages",
          "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:ImageProcessor",
          "Events": ["s3:ObjectCreated:*"],
          "Filter": {
            "Key": {
              "FilterRules": [
                {"Name": "prefix", "Value": "uploads/images/"},
                {"Name": "suffix", "Value": ".jpg"}
              ]
            }
          }
        }
      ],
      "QueueConfigurations": [
        {
          "Id": "LogProcessingQueue",
          "QueueArn": "arn:aws:sqs:us-east-1:123456789012:log-processing",
          "Events": ["s3:ObjectCreated:*"],
          "Filter": {
            "Key": {
              "FilterRules": [
                {"Name": "prefix", "Value": "logs/"},
                {"Name": "suffix", "Value": ".gz"}
              ]
            }
          }
        }
      ]
    }'
```

### EventBridge integration (recommended for new setups)

EventBridge offers more flexibility than direct S3 notifications:
- Route events to 20+ targets (not just SNS/SQS/Lambda)
- Apply content-based filtering rules
- Archive and replay events
- Cross-account event routing

```bash
# Enable EventBridge notifications on the bucket
aws s3api put-bucket-notification-configuration \
    --bucket my-bucket \
    --notification-configuration '{"EventBridgeConfiguration": {}}'
```

Then create EventBridge rules to route events:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["my-bucket"] },
    "object": { "key": [{ "prefix": "data/" }] }
  }
}
```

---

## 7. Static Website Hosting

S3 can serve static content (HTML, CSS, JavaScript, images) directly as a website. No web server (Apache, Nginx) is needed.

### What S3 static hosting provides

- HTTP endpoint for serving content
- Index document support (`index.html`)
- Custom error document (`404.html`)
- Redirect rules (e.g., redirect `/old-page` to `/new-page`)
- Hosting at bucket-level domain: `http://bucket-name.s3-website-region.amazonaws.com`

### What S3 static hosting does NOT provide

- HTTPS (HTTP only — use CloudFront for HTTPS)
- Server-side scripting (no PHP, Python, Node.js — static files only)
- Custom domain without Route 53 + CloudFront
- Authentication (public content only, or use CloudFront + Lambda@Edge)

### Setting up static website hosting

```bash
# Step 1: Enable website hosting
aws s3 website s3://my-website-bucket/ \
    --index-document index.html \
    --error-document error.html

# Step 2: Upload content
aws s3 sync ./website/ s3://my-website-bucket/ \
    --content-type "text/html" \
    --exclude "*.css" --exclude "*.js"
aws s3 sync ./website/ s3://my-website-bucket/ \
    --include "*.css" --content-type "text/css"
aws s3 sync ./website/ s3://my-website-bucket/ \
    --include "*.js" --content-type "application/javascript"

# Step 3: Disable Block Public Access (required for public website)
aws s3api put-public-access-block \
    --bucket my-website-bucket \
    --public-access-block-configuration \
    "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Step 4: Add bucket policy for public read
aws s3api put-bucket-policy \
    --bucket my-website-bucket \
    --policy '{
      "Version": "2012-10-17",
      "Statement": [{
        "Sid": "PublicReadGetObject",
        "Effect": "Allow",
        "Principal": "*",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::my-website-bucket/*"
      }]
    }'
```

### Website endpoint format

```
Endpoint format:
  http://<bucket-name>.s3-website-<region>.amazonaws.com
  http://<bucket-name>.s3-website.<region>.amazonaws.com

Examples:
  http://my-website.s3-website-us-east-1.amazonaws.com
  http://my-website.s3-website.eu-west-1.amazonaws.com
```

### Redirect rules

```json
{
  "RoutingRules": [
    {
      "Condition": {
        "KeyPrefixEquals": "old-docs/"
      },
      "Redirect": {
        "ReplaceKeyPrefixWith": "new-docs/",
        "HttpRedirectCode": "301"
      }
    },
    {
      "Condition": {
        "HttpErrorCodeReturnedEquals": "404"
      },
      "Redirect": {
        "HostName": "www.example.com",
        "Protocol": "https"
      }
    }
  ]
}
```

---

## 8. S3 with CloudFront

CloudFront is AWS's Content Delivery Network (CDN). Placing CloudFront in front of S3 is the standard production pattern for serving content.

### Why CloudFront + S3

| Without CloudFront | With CloudFront |
|:-------------------|:----------------|
| All requests go to S3 in one region | Requests served from nearest edge (400+ locations) |
| HTTP only (S3 website endpoint) | HTTPS with custom SSL certificate |
| No caching — every request hits S3 | Edge caching — S3 only hit on cache miss |
| S3 data transfer rates ($0.09/GB) | CloudFront rates ($0.085/GB, cheaper at scale) |
| Bucket name as domain | Custom domain (www.example.com) |
| No DDoS protection | AWS Shield Standard included |
| No request manipulation | Lambda@Edge for headers, auth, redirects |

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Users worldwide                                                        │
│  ┌──────┐  ┌──────┐  ┌──────┐                                         │
│  │Tokyo │  │London│  │ NYC  │                                          │
│  └──┬───┘  └──┬───┘  └──┬───┘                                         │
│     │         │         │                                               │
│     ▼         ▼         ▼                                               │
│  ┌──────┐  ┌──────┐  ┌──────┐     Cache miss only                     │
│  │Edge  │  │Edge  │  │Edge  │  ──────────────────────>  ┌───────────┐ │
│  │Tokyo │  │London│  │ NYC  │                           │ S3 Bucket │ │
│  └──────┘  └──────┘  └──────┘  <──────────────────────  │ (origin)  │ │
│     │         │         │          Fetches from S3       └───────────┘ │
│     │         │         │          and caches at edge                   │
│     ▼         ▼         ▼                                               │
│  Users get content from nearest edge location                           │
│  Subsequent requests served from cache (no S3 hit)                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Origin Access Control (OAC)

OAC is the modern way to restrict S3 access to only CloudFront (replacing the older OAI method). The bucket stays private — only CloudFront can read from it.

```
Without OAC: Bucket must be public → anyone can bypass CloudFront
With OAC:    Bucket is private → only CloudFront (via OAC) can access it
             Users MUST go through CloudFront
```

**Bucket policy for OAC:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

### Creating a CloudFront distribution for S3

```bash
aws cloudfront create-distribution \
    --origin-domain-name my-bucket.s3.us-east-1.amazonaws.com \
    --default-root-object index.html
```

In practice, you would configure via the console or CloudFormation with:
- Origin: S3 bucket (not the website endpoint)
- Origin Access Control: create and attach
- Viewer Protocol Policy: Redirect HTTP to HTTPS
- Cache Policy: CachingOptimized (recommended default)
- Default Root Object: `index.html`
- Custom domain + ACM certificate (for HTTPS on your domain)

### Cache invalidation

When you update content in S3, CloudFront edges may still serve the old cached version until TTL expires. Force an update:

```bash
# Invalidate specific files
aws cloudfront create-invalidation \
    --distribution-id EDFDVBD6EXAMPLE \
    --paths "/index.html" "/css/style.css"

# Invalidate everything (use sparingly — costs $0.005 per path after 1000/month free)
aws cloudfront create-invalidation \
    --distribution-id EDFDVBD6EXAMPLE \
    --paths "/*"
```

---

## 9. S3 Select and Glacier Select

S3 Select lets you retrieve a **subset** of an object's data using SQL expressions, instead of downloading the entire object and filtering client-side.

### The problem it solves

```
Without S3 Select:                       With S3 Select:
─────────────────                        ───────────────
  5GB CSV file in S3                       5GB CSV file in S3
  Download all 5GB → filter locally        Send SQL query to S3
  Transfer cost: 5GB × $0.09 = $0.45      S3 returns only matching rows
  Time: minutes to download                Transfer cost: maybe $0.001
                                           Time: seconds
```

### Supported formats

| Input Format | SQL Support |
|:-------------|:------------|
| CSV | Full SQL (SELECT, WHERE, LIMIT) |
| JSON | Full SQL with JSON path expressions |
| Parquet | Full SQL (columnar — very efficient for column selection) |

### Example query

```bash
aws s3api select-object-content \
    --bucket my-data-bucket \
    --key sales/2024/all-transactions.csv \
    --expression "SELECT s.product, s.revenue FROM s3object s WHERE s.region = 'EU' AND CAST(s.revenue AS FLOAT) > 1000" \
    --expression-type SQL \
    --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
    --output-serialization '{"CSV": {}}' \
    output.csv
```

### When to use S3 Select

| Scenario | Use S3 Select? |
|:---------|:--------------|
| Need a few columns from a large CSV | Yes — reduces data transferred |
| Need rows matching a filter from a large file | Yes — server-side filtering |
| Need the entire file | No — just download it |
| Complex joins or aggregations | No — use Athena instead |
| Frequent queries on the same data | No — use Athena or Redshift |

### Glacier Select

Same concept applied to Glacier archives — run SQL queries on archived data without restoring the entire archive. Results are written to an S3 bucket.

---

## 10. S3 Object Lock and MFA Delete

These features protect objects from deletion or modification — even by the root account.

### S3 Object Lock

Object Lock prevents objects from being deleted or overwritten for a specified retention period. Designed for regulatory compliance (SEC 17a-4, FINRA, HIPAA).

**Two retention modes:**

| Mode | Who Can Override | Use Case |
|:-----|:-----------------|:---------|
| **Governance** | Users with `s3:BypassGovernanceRetention` permission | Internal protection — admins can override in emergencies |
| **Compliance** | Nobody — not even the root account | Regulatory compliance — once set, cannot be shortened or removed |

```
Governance Mode:                          Compliance Mode:
────────────────                          ─────────────────
  Object locked for 365 days              Object locked for 365 days
  Admin with special permission           NO ONE can delete or modify
  can override and delete                 Not root, not AWS support
  Normal users cannot delete              Cannot shorten retention
                                          Must wait for period to expire
```

### Legal Hold

Independent of retention period. A Legal Hold prevents deletion indefinitely until explicitly removed. Used when objects are subject to legal proceedings.

```bash
# Put legal hold on an object
aws s3api put-object-legal-hold \
    --bucket my-bucket \
    --key evidence/document.pdf \
    --legal-hold '{"Status": "ON"}'

# Remove legal hold
aws s3api put-object-legal-hold \
    --bucket my-bucket \
    --key evidence/document.pdf \
    --legal-hold '{"Status": "OFF"}'
```

### MFA Delete

Requires Multi-Factor Authentication to:
- Permanently delete object versions
- Change the versioning state of a bucket

Can only be enabled by the **root account** using the CLI (not the console).

```bash
# Enable MFA Delete (must use root credentials)
aws s3api put-bucket-versioning \
    --bucket my-critical-bucket \
    --versioning-configuration Status=Enabled,MFADelete=Enabled \
    --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 123456"
```

---

## 11. Integration with Other AWS Services

S3 serves as a central data hub that connects to nearly every AWS service.

### Compute integrations

| Service | Integration | Example |
|:--------|:------------|:--------|
| **Lambda** | Event-driven processing | Image upload triggers thumbnail generation |
| **EC2** | Data source/destination | Application reads config, writes logs to S3 |
| **ECS/Fargate** | Input/output storage | Container reads training data, writes results |
| **EMR** | EMRFS direct access | Spark/Hadoop jobs process S3 data directly |
| **Batch** | Job input/output | Batch processing reads from and writes to S3 |

### Analytics integrations

| Service | Integration | Example |
|:--------|:------------|:--------|
| **Athena** | Query S3 data with SQL directly | Run SQL on CSV/Parquet files without loading into a database |
| **Redshift Spectrum** | Query S3 from Redshift | Extend Redshift queries to include S3 data lake |
| **Glue** | ETL and data catalog | Crawl S3 data, build schemas, transform data |
| **QuickSight** | Visualization | Create dashboards from S3 data via Athena |
| **Kinesis Firehose** | Streaming data delivery | Stream real-time data directly into S3 |

### Database integrations

| Service | Integration | Example |
|:--------|:------------|:--------|
| **RDS** | Export/import | Export database snapshots to S3 in Parquet format |
| **DynamoDB** | Export/import | Export DynamoDB tables to S3 for analytics |
| **Aurora** | Direct S3 access | `SELECT INTO OUTFILE S3` / `LOAD DATA FROM S3` |

### Security and logging integrations

| Service | Integration | Example |
|:--------|:------------|:--------|
| **CloudTrail** | Log storage | All API activity logs stored in S3 |
| **VPC Flow Logs** | Log destination | Network traffic logs delivered to S3 |
| **GuardDuty** | Threat detection | Monitors S3 data events for suspicious activity |
| **Macie** | Data classification | Scans S3 for sensitive data (PII, credentials) |
| **Config** | Configuration history | Stores configuration snapshots in S3 |

### Content delivery and transfer

| Service | Integration | Example |
|:--------|:------------|:--------|
| **CloudFront** | CDN origin | Serve S3 content globally with caching |
| **DataSync** | Automated transfer | Sync on-premises storage to S3 |
| **Transfer Family** | SFTP/FTPS access | Legacy systems upload to S3 via SFTP |
| **Snow Family** | Physical data transfer | Snowball Edge for petabyte-scale migration |

### Architecture: S3 as the center of a data lake

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATA LAKE ON S3                                  │
│                                                                         │
│  Ingestion:          Storage:              Processing:        Serving:  │
│  ──────────          ────────              ───────────        ────────  │
│                                                                         │
│  Kinesis ────┐       ┌─────────────────┐   ┌─ Athena (SQL)             │
│  IoT Core ──┐│      │                 │   │                            │
│  App logs ──┼┼─────>│   S3 BUCKET     │──>├─ EMR (Spark)    ──> QuickSight│
│  Databases ─┼│      │   (Data Lake)   │   │                            │
│  APIs ──────┘│      │                 │   ├─ Glue (ETL)               │
│  Files ──────┘      │  Raw / Curated  │   │                            │
│                      │  / Processed    │   └─ Redshift Spectrum        │
│                      └─────────────────┘                                │
│                              │                                          │
│                              └── Lifecycle rules move old data          │
│                                  to Glacier automatically               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Hands-On: Complete S3 Workflow

This lab combines multiple S3 features into a realistic workflow: versioned bucket with lifecycle rules, event notifications triggering Lambda, and CloudFront distribution.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  User uploads image                                                 │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────────────────────────────────┐                       │
│  │  S3 Bucket: my-app-uploads               │                       │
│  │  - Versioning: ON                        │                       │
│  │  - Encryption: SSE-S3                    │                       │
│  │  - Lifecycle: IA after 30d, Glacier 90d  │                       │
│  └─────────────────────┬────────────────────┘                       │
│                        │ s3:ObjectCreated event                      │
│                        ▼                                            │
│  ┌─────────────────────────────────┐                                │
│  │  Lambda: process-image          │                                │
│  │  - Generates thumbnail          │                                │
│  │  - Stores in thumbnails/ prefix │                                │
│  └─────────────────────────────────┘                                │
│                        │                                            │
│                        ▼                                            │
│  ┌──────────────────────────────────────────┐                       │
│  │  CloudFront Distribution                 │                       │
│  │  - Origin: S3 bucket (OAC)              │                       │
│  │  - Serves thumbnails to users           │                       │
│  │  - HTTPS with custom domain             │                       │
│  └──────────────────────────────────────────┘                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 1: Create and configure the bucket

```bash
# Create bucket
aws s3 mb s3://my-app-uploads-demo --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket my-app-uploads-demo \
    --versioning-configuration Status=Enabled
```

### Step 2: Add lifecycle rules

```bash
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-app-uploads-demo \
    --lifecycle-configuration '{
      "Rules": [
        {
          "ID": "TransitionUploads",
          "Status": "Enabled",
          "Filter": {"Prefix": "uploads/"},
          "Transitions": [
            {"Days": 30, "StorageClass": "STANDARD_IA"},
            {"Days": 90, "StorageClass": "GLACIER"}
          ],
          "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
        },
        {
          "ID": "CleanupFailedUploads",
          "Status": "Enabled",
          "Filter": {},
          "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
        }
      ]
    }'
```

### Step 3: Configure event notification for Lambda

```bash
aws s3api put-bucket-notification-configuration \
    --bucket my-app-uploads-demo \
    --notification-configuration '{
      "LambdaFunctionConfigurations": [{
        "Id": "ThumbnailGenerator",
        "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:process-image",
        "Events": ["s3:ObjectCreated:*"],
        "Filter": {
          "Key": {
            "FilterRules": [
              {"Name": "prefix", "Value": "uploads/"},
              {"Name": "suffix", "Value": ".jpg"}
            ]
          }
        }
      }]
    }'
```

### Step 4: Test the workflow

```bash
# Upload an image
aws s3 cp ./photo.jpg s3://my-app-uploads-demo/uploads/photo.jpg

# Verify thumbnail was created (by Lambda)
aws s3 ls s3://my-app-uploads-demo/thumbnails/

# Upload another version (versioning creates new version)
aws s3 cp ./photo-v2.jpg s3://my-app-uploads-demo/uploads/photo.jpg

# List versions
aws s3api list-object-versions \
    --bucket my-app-uploads-demo \
    --prefix uploads/photo.jpg
```

### Step 5: Verify lifecycle rules

```bash
# Check lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket my-app-uploads-demo

# After 30 days, check storage class changed
aws s3api head-object --bucket my-app-uploads-demo --key uploads/photo.jpg
# Should show: "StorageClass": "STANDARD_IA"
```

---

## 13. Key Points to Remember

1. **Versioning cannot be disabled, only suspended.** Once enabled, the bucket cannot go back to "unversioned." Old versions remain and are billed.

2. **DELETE in a versioned bucket adds a delete marker.** The object is not removed. To permanently delete, you must delete by version ID.

3. **Lifecycle rules are the #1 cost tool.** Every production bucket should have: transition to IA/Glacier + expire old versions + abort incomplete multipart uploads.

4. **Replication requires versioning on both buckets.** Existing objects are not replicated retroactively — use Batch Replication for those.

5. **Transfer Acceleration only charges when it helps.** If the accelerated path is not faster, you use the normal path and pay nothing extra.

6. **Multipart upload is automatic in the CLI above 8MB.** For SDK/API, implement it yourself for files > 100MB. Always include a lifecycle rule to clean up incomplete uploads.

7. **S3 events + Lambda = serverless processing pipeline.** No servers to manage. S3 triggers Lambda on upload, Lambda processes and writes results back to S3.

8. **CloudFront + S3 with OAC is the production pattern.** Never expose S3 buckets directly to the internet in production. Use CloudFront for HTTPS, caching, custom domains, and DDoS protection.

9. **Object Lock in Compliance mode is irreversible.** Not even AWS support can delete the object before retention expires. Test in Governance mode first.

10. **S3 integrates with almost everything in AWS.** It is the universal data layer — logs go there, backups go there, data lakes live there, analytics query it directly.

---

## 14. Self-Check Questions

1. You accidentally deleted an important file from a versioned bucket 3 days ago. How do you recover it?
   > The delete operation only added a delete marker. The previous versions still exist. List versions (`aws s3api list-object-versions`), find the delete marker's version ID, and delete the delete marker itself. The most recent real version becomes current again, and the object reappears.

2. You have a lifecycle rule that transitions objects to Standard-IA after 30 days and to Glacier after 60 days. An object is uploaded and accessed daily. Does it still get moved?
   > Yes. Lifecycle rules are based on object **creation date** (or last modified date), NOT last access date. The object transitions regardless of how often it is accessed. To avoid this, use Intelligent-Tiering instead, which moves based on access patterns.

3. You enabled CRR (Cross-Region Replication) on a bucket that already contains 1 million objects. Are they replicated?
   > No. Replication only applies to new objects created after the rule is enabled. To replicate existing objects, use S3 Batch Replication.

4. Your users in Asia experience slow uploads to a bucket in us-east-1. What feature should you enable?
   > Transfer Acceleration. Users will upload to the nearest CloudFront edge location, and data travels over AWS's optimized backbone to us-east-1. This significantly reduces latency and improves throughput for long-distance transfers.

5. You want to automatically generate a PDF report whenever a CSV file is uploaded to S3. How?
   > Configure an S3 event notification for `s3:ObjectCreated:*` with a suffix filter of `.csv`, pointing to a Lambda function. The Lambda reads the CSV from S3, generates the PDF, and writes it back to S3 (or sends it via SES/SNS).

6. Your static website hosted on S3 works over HTTP but you need HTTPS. What do you do?
   > S3 website endpoints do not support HTTPS. Create a CloudFront distribution with the S3 bucket as the origin, request an ACM (Certificate Manager) certificate, and configure CloudFront to use it. Point your custom domain to CloudFront via Route 53.

7. You need to ensure that a compliance document cannot be deleted by anyone for 7 years. How?
   > Enable S3 Object Lock on the bucket (must be done at bucket creation), then apply a Compliance mode retention policy of 7 years to the object. In Compliance mode, no one — not even the root account — can delete or modify the object until the retention period expires.

8. A 50GB file upload keeps failing at 80%. What should you do?
   > Use Multipart Upload. Split the file into parts (e.g., 100MB each), upload parts independently (with parallelism), and if any part fails, retry only that part. The AWS CLI does this automatically for files > 8MB. For manual control, increase `multipart_chunksize` and `max_concurrent_requests`.

---

## References

- [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [S3 Lifecycle Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [S3 Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [S3 Transfer Acceleration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transfer-acceleration.html)
- [Multipart Upload Overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)
- [Hosting a Static Website on S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [Using CloudFront with S3](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [S3 Select](https://docs.aws.amazon.com/AmazonS3/latest/userguide/selecting-content-from-objects.html)
- [S3 and EventBridge](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventBridge.html)
