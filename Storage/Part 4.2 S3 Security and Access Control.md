# Part 4.2: S3 Security and Access Control

---

## Table of Contents

1. [S3 Security Model Overview](#1-s3-security-model-overview)
2. [IAM Policies for S3](#2-iam-policies-for-s3)
3. [Bucket Policies](#3-bucket-policies)
4. [Access Control Lists (ACLs)](#4-access-control-lists-acls)
5. [S3 Access Points](#5-s3-access-points)
6. [Pre-Signed URLs](#6-pre-signed-urls)
7. [Block Public Access](#7-block-public-access)
8. [S3 Encryption Deep Dive](#8-s3-encryption-deep-dive)
9. [VPC Endpoints for S3](#9-vpc-endpoints-for-s3)
10. [S3 Access Logging and Monitoring](#10-s3-access-logging-and-monitoring)
11. [Hands-On: Securing an S3 Bucket](#11-hands-on-securing-an-s3-bucket)
12. [Key Points to Remember](#12-key-points-to-remember)
13. [Self-Check Questions](#13-self-check-questions)

---

## 1. S3 Security Model Overview

S3 security operates through multiple independent layers. A request must pass ALL applicable layers to succeed. If any layer denies the request, access is blocked regardless of what other layers allow.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      S3 ACCESS EVALUATION                               │
│                                                                         │
│   Request arrives                                                       │
│        │                                                                │
│        ▼                                                                │
│   ┌──────────────────────┐                                              │
│   │ Block Public Access  │ ← Account-level or bucket-level toggle       │
│   │ (overrides everything│   If enabled, public access is DENIED        │
│   │  if it blocks)       │   regardless of policies below               │
│   └──────────┬───────────┘                                              │
│              │                                                          │
│              ▼                                                          │
│   ┌──────────────────────┐                                              │
│   │ IAM Policy           │ ← Identity-based: what can THIS user/role do?│
│   │ (user/role perms)    │                                              │
│   └──────────┬───────────┘                                              │
│              │                                                          │
│              ▼                                                          │
│   ┌──────────────────────┐                                              │
│   │ Bucket Policy        │ ← Resource-based: who can access THIS bucket?│
│   │ (resource perms)     │   Supports cross-account access              │
│   └──────────┬───────────┘                                              │
│              │                                                          │
│              ▼                                                          │
│   ┌──────────────────────┐                                              │
│   │ ACL (legacy)         │ ← Per-bucket or per-object permissions       │
│   │                      │   AWS recommends disabling                   │
│   └──────────┬───────────┘                                              │
│              │                                                          │
│              ▼                                                          │
│        ACCESS GRANTED                                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### The evaluation logic

For **same-account** access (user and bucket in the same AWS account):
- Access is allowed if either the IAM policy OR the bucket policy grants it
- An explicit DENY in either one overrides any ALLOW

For **cross-account** access (user in account A, bucket in account B):
- Access requires BOTH the IAM policy (in account A) AND the bucket policy (in account B) to grant access
- Missing permission in either = denied

```
Same-account:                         Cross-account:
─────────────                         ──────────────
  IAM allows OR Bucket allows         IAM allows AND Bucket allows
  = Access granted                    = Access granted

  Either one is sufficient            BOTH are required
  (unless explicit deny exists)       (missing either = denied)
```

---

## 2. IAM Policies for S3

IAM policies are attached to **users, groups, or roles** and define what S3 actions they can perform on which resources.

### Policy structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadOnlyAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

### Common S3 actions

| Action | What It Allows |
|:-------|:---------------|
| `s3:GetObject` | Download/read objects |
| `s3:PutObject` | Upload/write objects |
| `s3:DeleteObject` | Delete objects |
| `s3:ListBucket` | List objects in a bucket |
| `s3:GetBucketLocation` | Get bucket region |
| `s3:GetBucketPolicy` | Read bucket policy |
| `s3:PutBucketPolicy` | Modify bucket policy |
| `s3:GetObjectVersion` | Read specific versions |
| `s3:DeleteObjectVersion` | Permanently delete a version |
| `s3:PutLifecycleConfiguration` | Set lifecycle rules |
| `s3:GetEncryptionConfiguration` | Read encryption settings |
| `s3:*` | All S3 actions (dangerous — avoid in production) |

### Resource ARN format

Two different ARN patterns are needed because bucket-level and object-level are separate resources:

```
Bucket-level actions (ListBucket, GetBucketPolicy, etc.):
  arn:aws:s3:::my-bucket

Object-level actions (GetObject, PutObject, DeleteObject, etc.):
  arn:aws:s3:::my-bucket/*
  arn:aws:s3:::my-bucket/prefix/*
```

A common mistake is using only `arn:aws:s3:::my-bucket/*` for ListBucket — this fails because ListBucket operates on the bucket resource, not objects.

### Example: Read-only access to a specific prefix

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::company-data",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["reports/*"]
        }
      }
    },
    {
      "Sid": "AllowGetObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::company-data/reports/*"
    }
  ]
}
```

This allows listing and downloading objects only under the `reports/` prefix. Objects under `secrets/` or root level are inaccessible.

### Example: Deny delete for everyone except admins

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteForNonAdmins",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::critical-data/*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::123456789012:role/AdminRole"
        }
      }
    }
  ]
}
```

---

## 3. Bucket Policies

Bucket policies are **resource-based** JSON policies attached to the bucket itself. They define who can access the bucket and what they can do — including users from other AWS accounts.

### When to use bucket policies vs IAM policies

| Use Case | Best Choice | Why |
|:---------|:------------|:----|
| Control what a specific user/role can do | IAM Policy | Attached to the identity |
| Control who can access a specific bucket | Bucket Policy | Attached to the resource |
| Grant cross-account access | Bucket Policy | IAM policies cannot grant access to external accounts |
| Enforce encryption on all uploads | Bucket Policy | Applies to everyone regardless of their IAM policy |
| Restrict access by IP address or VPC | Bucket Policy | Condition keys for network-based restrictions |

### Bucket policy structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StatementIdentifier",
      "Effect": "Allow | Deny",
      "Principal": "Who is affected",
      "Action": "What actions",
      "Resource": "Which objects/bucket",
      "Condition": { "Optional conditions" }
    }
  ]
}
```

The key difference from IAM policies is the **Principal** field — bucket policies specify WHO the policy applies to.

### Example: Allow public read (static website)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-website-bucket/*"
    }
  ]
}
```

`"Principal": "*"` means anyone — authenticated or not. This makes objects publicly readable.

### Example: Cross-account access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPartnerAccountRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:root"
      },
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::shared-data-bucket",
        "arn:aws:s3:::shared-data-bucket/*"
      ]
    }
  ]
}
```

Account `987654321098` can now read from this bucket. Their IAM users/roles still need an IAM policy allowing the same actions (cross-account requires both sides to grant).

### Example: Deny unencrypted uploads

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::secure-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

Any upload without KMS encryption is denied — regardless of who performs it.

### Example: Restrict access to specific VPC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictToVPC",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::internal-bucket",
        "arn:aws:s3:::internal-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpc": "vpc-1a2b3c4d"
        }
      }
    }
  ]
}
```

All access is denied unless the request originates from the specified VPC (requires a VPC endpoint for S3).

### Example: Restrict by IP address

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyOfficeIP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::sensitive-bucket",
        "arn:aws:s3:::sensitive-bucket/*"
      ],
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
        }
      }
    }
  ]
}
```

---

## 4. Access Control Lists (ACLs)

ACLs are the **legacy** access control mechanism for S3. They predate IAM and bucket policies.

### Why ACLs still exist but should not be used

ACLs were the original S3 permission system (2006). They are:
- Limited in granularity (only a few canned permissions)
- Difficult to audit (scattered across individual objects)
- Cannot express complex conditions (no IP restrictions, no MFA requirements)
- Not visible in a single place (unlike bucket policies which are centralized)

**AWS recommendation since April 2023**: All new buckets are created with ACLs disabled (BucketOwnerEnforced) by default. All access control should be done through IAM policies and bucket policies.

### Canned ACLs (if you encounter them in legacy systems)

| Canned ACL | Effect |
|:-----------|:-------|
| `private` | Owner gets full control. No one else has access. (Default) |
| `public-read` | Owner full control + everyone can read |
| `public-read-write` | Owner full control + everyone can read and write (extremely dangerous) |
| `authenticated-read` | Owner full control + any authenticated AWS user can read |
| `bucket-owner-full-control` | Object owner + bucket owner get full control |

### Disabling ACLs (recommended for all buckets)

```bash
aws s3api put-bucket-ownership-controls \
    --bucket my-bucket \
    --ownership-controls '{"Rules": [{"ObjectOwnership": "BucketOwnerEnforced"}]}'
```

When `BucketOwnerEnforced` is set:
- All ACLs are disabled
- Bucket owner automatically owns all objects (even those uploaded by other accounts)
- All access control is done via policies only

---

## 5. S3 Access Points

Access Points are **named network endpoints** attached to a bucket. Each Access Point has its own DNS name, access policy, and network configuration. They simplify managing access to shared buckets with many different consumer applications.

### The problem Access Points solve

Without Access Points, a shared bucket's policy grows into a massive document with dozens of statements for different teams, applications, and access patterns. A single misconfiguration can expose everything.

```
Without Access Points:                    With Access Points:
─────────────────────────                 ────────────────────
                                          
  ┌───────────────────────┐                 ┌─── AP: analytics-team ───┐
  │  Bucket Policy:       │                 │  Policy: read logs/*     │
  │                       │                 │  VPC: vpc-analytics      │
  │  50+ statements for   │                 └──────────────┬───────────┘
  │  different teams,     │                                │
  │  VPCs, conditions     │                 ┌─── AP: ml-pipeline ──────┐
  │                       │                 │  Policy: read/write ml/* │
  │  One mistake =        │                 │  VPC: vpc-ml-prod        │
  │  data breach          │                 └──────────────┬───────────┘
  │                       │                                │
  └───────────────────────┘                 ┌─── AP: public-website ───┐
                                            │  Policy: read public/*   │
  Hard to manage,                           │  Network: internet       │
  hard to audit,                            └──────────────┬───────────┘
  easy to misconfigure                                     │
                                                  ┌────────┴────────┐
                                                  │    S3 Bucket    │
                                                  └─────────────────┘
                                            
                                            Each AP has isolated policy.
                                            Each AP can restrict to a VPC.
                                            Easier to audit and manage.
```

### Creating an Access Point

```bash
aws s3control create-access-point \
    --account-id 123456789012 \
    --name analytics-team-ap \
    --bucket my-shared-bucket \
    --vpc-configuration VpcId=vpc-1a2b3c4d
```

### Access Point policy example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AnalyticsTeamReadOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/AnalyticsRole"
      },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-team-ap",
        "arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-team-ap/object/logs/*"
      ]
    }
  ]
}
```

### Using Access Points in code

Access Points have their own ARN that replaces the bucket name:

```bash
# Instead of:
aws s3 cp s3://my-shared-bucket/logs/app.log ./

# Use the Access Point ARN:
aws s3 cp s3://arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-team-ap/logs/app.log ./
```

---

## 6. Pre-Signed URLs

A pre-signed URL grants **time-limited access** to a specific S3 object without requiring the requester to have AWS credentials. The URL is signed with the credentials of the user who generates it.

### How pre-signed URLs work

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  1. Your server generates a pre-signed URL                           │
│     (using your AWS credentials + expiration time)                   │
│                                                                      │
│  2. You give the URL to the client (browser, mobile app, partner)    │
│                                                                      │
│  3. Client uses the URL to GET or PUT directly to/from S3            │
│     (no AWS credentials needed by the client)                        │
│                                                                      │
│  4. URL expires after the configured time → access revoked           │
│                                                                      │
│  Server                      Client                     S3           │
│    │                           │                         │           │
│    │── Generate pre-signed URL │                         │           │
│    │── Send URL to client ────>│                         │           │
│    │                           │── GET/PUT using URL ───>│           │
│    │                           │<── Object data ─────────│           │
│    │                           │                         │           │
│    │         No AWS creds      │    Validates signature  │           │
│    │         on client         │    and expiration       │           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Generating pre-signed URLs

```bash
# CLI: generate a download URL valid for 1 hour (3600 seconds)
aws s3 presign s3://my-bucket/private-report.pdf --expires-in 3600

# Output:
# https://my-bucket.s3.amazonaws.com/private-report.pdf?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Expires=3600&X-Amz-Signature=...
```

```python
# Python (boto3): generate download URL
import boto3

s3 = boto3.client('s3')
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'private-report.pdf'},
    ExpiresIn=3600
)

# Generate upload URL (allows client to PUT directly)
upload_url = s3.generate_presigned_url(
    'put_object',
    Params={'Bucket': 'my-bucket', 'Key': 'uploads/user-photo.jpg'},
    ExpiresIn=300
)
```

### Use cases

| Scenario | How |
|:---------|:----|
| Let users download private files without login | Generate GET pre-signed URL on your server, redirect user to it |
| Let users upload directly to S3 (bypass your server) | Generate PUT pre-signed URL, client uploads straight to S3 |
| Share temporary access with a partner | Email them a pre-signed URL with 24-hour expiry |
| Serve private content from a web page | Embed pre-signed URL in `<img src="...">` or `<a href="...">` |

### Important constraints

- The URL is only valid as long as the **credentials used to generate it are valid**. If you generate a URL using temporary credentials (STS) that expire in 1 hour, the URL cannot be valid longer than 1 hour regardless of `ExpiresIn`.
- Maximum expiration: 7 days (604,800 seconds) for IAM user credentials, shorter for temporary credentials.
- Anyone with the URL can use it — protect it like a password during its validity period.
- The pre-signed URL inherits the permissions of the identity that generated it. If that identity loses `s3:GetObject` permission, existing URLs stop working.

---

## 7. Block Public Access

Block Public Access (BPA) is an **account-level and bucket-level safety net** that prevents accidental public exposure. It overrides bucket policies and ACLs that would grant public access.

### The four BPA settings

| Setting | What It Blocks |
|:--------|:---------------|
| `BlockPublicAcls` | Rejects PUT requests that include a public ACL |
| `IgnorePublicAcls` | Ignores all existing public ACLs on the bucket and objects |
| `BlockPublicPolicy` | Rejects bucket policies that grant public access |
| `RestrictPublicBuckets` | Restricts access to the bucket to only AWS services and authorized users (ignores any public policy already applied) |

### Account-level vs bucket-level

```
Account-level BPA (applies to ALL buckets in the account):
──────────────────────────────────────────────────────────
  Enable this first. It's the safety net for every bucket.
  Even if someone puts a public policy on a bucket, BPA blocks it.

Bucket-level BPA (override per-bucket):
──────────────────────────────────────────────────────────
  You can disable BPA on specific buckets that genuinely
  need public access (like static website hosting).
  But account-level BPA must also allow it.
```

### Enabling Block Public Access

```bash
# Account-level (recommended: enable on ALL accounts)
aws s3control put-public-access-block \
    --account-id 123456789012 \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Bucket-level
aws s3api put-public-access-block \
    --bucket my-bucket \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### When to disable BPA (rare cases)

- Static website hosting bucket (needs public read)
- Public dataset bucket (research data for anyone to download)
- CloudFront Origin Access Identity (doesn't need public — use OAC instead)

Even in these cases, disable only the specific settings needed, not all four.

---

## 8. S3 Encryption Deep Dive

S3 encryption protects data at rest (stored on disk) and in transit (moving over the network). Since January 2023, **all new objects are encrypted by default** using SSE-S3.

### Encryption at rest — Server-Side Encryption (SSE)

S3 encrypts data before writing it to disk and decrypts it when you read. Three options:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    SERVER-SIDE ENCRYPTION OPTIONS                         │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────┐ │
│  │   SSE-S3       │  │   SSE-KMS      │  │   SSE-C                    │ │
│  │                │  │                │  │                            │ │
│  │ AWS manages    │  │ AWS KMS        │  │ Customer provides          │ │
│  │ everything     │  │ manages keys   │  │ their own key              │ │
│  │                │  │                │  │                            │ │
│  │ • Zero config  │  │ • Audit trail  │  │ • You manage key storage   │ │
│  │ • Default      │  │ • Key rotation │  │ • Key sent with every req  │ │
│  │ • No extra     │  │ • Per-key      │  │ • AWS never stores the key │ │
│  │   cost         │  │   permissions  │  │ • If you lose key = data   │ │
│  │ • No audit     │  │ • KMS costs    │  │   lost forever             │ │
│  │   trail        │  │   apply        │  │ • HTTPS required           │ │
│  │                │  │                │  │                            │ │
│  │ Header:        │  │ Header:        │  │ Header:                    │ │
│  │ x-amz-server-  │  │ x-amz-server-  │  │ x-amz-server-side-        │ │
│  │ side-encryption │  │ side-encryption │  │ encryption-customer-      │ │
│  │ :AES256        │  │ :aws:kms       │  │ algorithm: AES256          │ │
│  └────────────────┘  └────────────────┘  └────────────────────────────┘ │
│                                                                          │
│  Recommended:         Best for:           Use only when:                 │
│  Default choice       Compliance,         Regulatory requirement         │
│  for most data        auditing,           to control keys yourself       │
│                       key control                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

### SSE-S3 (AES-256)

- Default encryption for all S3 objects
- AWS generates, manages, and rotates the keys
- Zero configuration required
- No additional cost
- No audit trail of key usage (you cannot track who decrypted what)

```bash
# Explicitly specify SSE-S3 (optional — it's the default)
aws s3 cp ./file.txt s3://my-bucket/file.txt --sse AES256
```

### SSE-KMS

- Uses AWS Key Management Service to manage encryption keys
- You can use the AWS-managed key (`aws/s3`) or create your own Customer Managed Key (CMK)
- **CloudTrail logs every use of the key** — full audit trail of who decrypted what and when
- Key policies control who can use the key (separate from S3 permissions)
- Automatic key rotation (every year for AWS-managed keys, configurable for CMKs)
- **KMS request charges apply** ($0.03 per 10,000 requests)

```bash
# Upload with SSE-KMS using default aws/s3 key
aws s3 cp ./file.txt s3://my-bucket/file.txt --sse aws:kms

# Upload with SSE-KMS using a specific CMK
aws s3 cp ./file.txt s3://my-bucket/file.txt \
    --sse aws:kms \
    --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012
```

### SSE-C (Customer-Provided Keys)

- You provide the encryption key with every PUT and GET request
- AWS uses your key to encrypt/decrypt but **never stores the key**
- If you lose the key, your data is permanently unrecoverable
- HTTPS is mandatory (key travels in the request header)
- No KMS costs, but operational burden is on you

```bash
# Upload with SSE-C
aws s3api put-object \
    --bucket my-bucket \
    --key secret-file.txt \
    --body ./secret-file.txt \
    --sse-customer-algorithm AES256 \
    --sse-customer-key fileb://my-256bit-key.bin \
    --sse-customer-key-md5 "base64-md5-of-key"
```

### Client-Side Encryption

Not an S3 feature — you encrypt data before uploading and decrypt after downloading. S3 stores opaque encrypted bytes. Useful when you cannot trust the server at all.

### Encryption in transit

All S3 API calls use HTTPS by default. You can enforce this via bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### Default encryption configuration

Set a default encryption method for all objects in a bucket (objects uploaded without specifying encryption use this):

```bash
# Set default encryption to SSE-KMS
aws s3api put-bucket-encryption \
    --bucket my-bucket \
    --server-side-encryption-configuration '{
      "Rules": [{
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/my-key-id"
        },
        "BucketKeyEnabled": true
      }]
    }'
```

### S3 Bucket Keys (cost optimization for SSE-KMS)

Without Bucket Keys, every object generates a separate KMS request (encrypt/decrypt). With 1 million GETs per day, that is $3/day in KMS charges alone.

**Bucket Keys** reduce KMS costs by up to 99%: S3 generates a bucket-level key from KMS and uses it for a time window, reducing individual KMS API calls.

```bash
# Enable Bucket Keys
aws s3api put-bucket-encryption \
    --bucket my-bucket \
    --server-side-encryption-configuration '{
      "Rules": [{
        "ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"},
        "BucketKeyEnabled": true
      }]
    }'
```

---

## 9. VPC Endpoints for S3

By default, EC2 instances access S3 over the public internet (even though both are AWS services). A **VPC Endpoint** creates a private connection between your VPC and S3 — traffic never leaves the AWS network.

### Gateway Endpoint (most common for S3)

```
Without VPC Endpoint:                    With VPC Endpoint:
─────────────────────                    ──────────────────
  EC2 → Internet Gateway                   EC2 → VPC Endpoint → S3
       → Public internet                   (private AWS network)
       → S3 endpoint                       No internet needed
                                           No NAT Gateway needed
  Requires:                                Requires:
  - Internet Gateway                       - VPC Endpoint
  - Public IP or NAT Gateway               - Route table entry
  - Security group allowing outbound       - Endpoint policy (optional)
```

### Why use VPC Endpoints

| Benefit | Detail |
|:--------|:-------|
| **Security** | Traffic stays on AWS private network — no internet exposure |
| **Cost savings** | No NAT Gateway data processing charges ($0.045/GB saved) |
| **Performance** | Lower latency — no internet routing |
| **Compliance** | Data never traverses the public internet |

### Creating a Gateway Endpoint

```bash
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-1a2b3c4d \
    --service-name com.amazonaws.us-east-1.s3 \
    --route-table-ids rtb-11aa22bb
```

This adds an entry to the specified route table that routes S3 traffic through the endpoint.

### Endpoint policy

You can restrict which buckets are accessible through the endpoint:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlySpecificBucket",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::approved-bucket/*"
    }
  ]
}
```

Instances using this endpoint can only access `approved-bucket`. Requests to any other bucket are denied.

### Interface Endpoint (alternative)

For S3, there is also an Interface Endpoint (powered by PrivateLink). Use it when:
- You need to access S3 from on-premises via VPN/Direct Connect
- You need DNS resolution within the VPC (private DNS)
- You need to access S3 from another region via VPC peering

Gateway Endpoints are free; Interface Endpoints have hourly + data charges.

---

## 10. S3 Access Logging and Monitoring

### Server Access Logging

Records all requests made to a bucket in detailed log files stored in a target bucket.

```bash
# Enable server access logging
aws s3api put-bucket-logging \
    --bucket my-bucket \
    --bucket-logging-status '{
      "LoggingEnabled": {
        "TargetBucket": "my-logs-bucket",
        "TargetPrefix": "s3-access-logs/my-bucket/"
      }
    }'
```

**Log format** (each request generates one line):

```
79a59df900b949e55d96a1e698fbacedfd6e09d98eacf8f8d5218e7cd47ef2be
my-bucket [06/Feb/2024:00:00:38 +0000] 192.0.2.3
79a59df900b949e55d96a1e698fbacedfd6e09d98eacf8f8d5218e7cd47ef2be
REST.GET.OBJECT reports/q4.pdf "GET /reports/q4.pdf HTTP/1.1"
200 - 3462 3462 12 11 "-" "aws-cli/2.13.0" - ...
```

Fields include: bucket owner, bucket name, timestamp, remote IP, requester, operation, key, HTTP status, bytes sent, time taken, user agent.

### CloudTrail for S3

CloudTrail records **management events** (bucket creation, policy changes) by default. For **data events** (GetObject, PutObject), you must explicitly enable them:

```bash
aws cloudtrail put-event-selectors \
    --trail-name my-trail \
    --event-selectors '[{
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [{
        "Type": "AWS::S3::Object",
        "Values": ["arn:aws:s3:::my-sensitive-bucket/"]
      }]
    }]'
```

### CloudWatch Metrics for S3

S3 publishes metrics to CloudWatch for monitoring:

| Metric | What It Measures |
|:-------|:-----------------|
| `BucketSizeBytes` | Total size of all objects (daily) |
| `NumberOfObjects` | Total object count (daily) |
| `AllRequests` | Total number of requests |
| `GetRequests` | Number of GET requests |
| `PutRequests` | Number of PUT requests |
| `4xxErrors` | Client error count |
| `5xxErrors` | Server error count |
| `FirstByteLatency` | Time to first byte |
| `TotalRequestLatency` | Total request time |

### S3 Storage Lens

Organization-wide dashboard providing visibility into storage usage and activity across all accounts and buckets. Includes:
- Usage metrics (size, object count, by storage class)
- Activity metrics (requests, bytes downloaded, error rates)
- Cost optimization recommendations
- Drill-down by account, region, bucket, prefix

---

## 11. Hands-On: Securing an S3 Bucket

This lab walks through securing a bucket with multiple layers of protection — the way you would configure a production bucket.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   SECURED S3 BUCKET                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Layer 1: Block Public Access (enabled)             │    │
│  │  ┌─────────────────────────────────────────────┐    │    │
│  │  │  Layer 2: Bucket Policy                     │    │    │
│  │  │  - Deny unencrypted uploads                 │    │    │
│  │  │  - Deny non-HTTPS                           │    │    │
│  │  │  - Restrict to VPC                          │    │    │
│  │  │  ┌─────────────────────────────────────┐    │    │    │
│  │  │  │  Layer 3: Default Encryption (KMS)  │    │    │    │
│  │  │  │  ┌─────────────────────────────┐    │    │    │    │
│  │  │  │  │  Layer 4: Versioning         │    │    │    │    │
│  │  │  │  │  (protect against deletion)  │    │    │    │    │
│  │  │  │  └─────────────────────────────┘    │    │    │    │
│  │  │  └─────────────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  + Access Logging enabled                                   │
│  + CloudTrail data events enabled                           │
│  + Lifecycle rules for cost management                      │
└─────────────────────────────────────────────────────────────┘
```

### Step 1: Create the bucket

```bash
aws s3 mb s3://company-secure-data-2024 --region us-east-1
```

### Step 2: Enable Block Public Access

```bash
aws s3api put-public-access-block \
    --bucket company-secure-data-2024 \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### Step 3: Disable ACLs

```bash
aws s3api put-bucket-ownership-controls \
    --bucket company-secure-data-2024 \
    --ownership-controls '{"Rules": [{"ObjectOwnership": "BucketOwnerEnforced"}]}'
```

### Step 4: Enable versioning

```bash
aws s3api put-bucket-versioning \
    --bucket company-secure-data-2024 \
    --versioning-configuration Status=Enabled
```

### Step 5: Enable default KMS encryption with Bucket Keys

```bash
aws s3api put-bucket-encryption \
    --bucket company-secure-data-2024 \
    --server-side-encryption-configuration '{
      "Rules": [{
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "alias/s3-encryption-key"
        },
        "BucketKeyEnabled": true
      }]
    }'
```

### Step 6: Apply security bucket policy

```bash
aws s3api put-bucket-policy \
    --bucket company-secure-data-2024 \
    --policy '{
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "DenyInsecureTransport",
          "Effect": "Deny",
          "Principal": "*",
          "Action": "s3:*",
          "Resource": [
            "arn:aws:s3:::company-secure-data-2024",
            "arn:aws:s3:::company-secure-data-2024/*"
          ],
          "Condition": {
            "Bool": {"aws:SecureTransport": "false"}
          }
        },
        {
          "Sid": "DenyUnencryptedUploads",
          "Effect": "Deny",
          "Principal": "*",
          "Action": "s3:PutObject",
          "Resource": "arn:aws:s3:::company-secure-data-2024/*",
          "Condition": {
            "StringNotEquals": {
              "s3:x-amz-server-side-encryption": "aws:kms"
            }
          }
        }
      ]
    }'
```

### Step 7: Enable access logging

```bash
# First create the logging bucket
aws s3 mb s3://company-secure-data-2024-logs --region us-east-1

# Enable logging
aws s3api put-bucket-logging \
    --bucket company-secure-data-2024 \
    --bucket-logging-status '{
      "LoggingEnabled": {
        "TargetBucket": "company-secure-data-2024-logs",
        "TargetPrefix": "access-logs/"
      }
    }'
```

### Step 8: Verify the configuration

```bash
# Check Block Public Access
aws s3api get-public-access-block --bucket company-secure-data-2024

# Check versioning
aws s3api get-bucket-versioning --bucket company-secure-data-2024

# Check encryption
aws s3api get-bucket-encryption --bucket company-secure-data-2024

# Check bucket policy
aws s3api get-bucket-policy --bucket company-secure-data-2024

# Test: try uploading without encryption (should fail)
aws s3 cp ./test.txt s3://company-secure-data-2024/test.txt --sse AES256
# Expected: Access Denied (policy requires aws:kms, not AES256)

# Test: upload with correct encryption (should succeed)
aws s3 cp ./test.txt s3://company-secure-data-2024/test.txt --sse aws:kms
```

---

## 12. Key Points to Remember

1. **S3 default = private.** New buckets deny all public access by default. You must explicitly grant access through IAM policies or bucket policies.

2. **Cross-account access requires BOTH sides.** The bucket policy must allow the external account, AND the external account's IAM must allow the S3 actions. Missing either = denied.

3. **Block Public Access overrides everything.** Even if your bucket policy says `"Principal": "*"`, BPA will block it if enabled. Disable BPA only for intentionally public buckets.

4. **ACLs are legacy — disable them.** Use `BucketOwnerEnforced` object ownership on all buckets. Manage access exclusively through IAM and bucket policies.

5. **SSE-S3 is automatic since January 2023.** Every new object is encrypted at rest by default. Use SSE-KMS when you need audit trails or fine-grained key access control.

6. **Bucket Keys reduce KMS costs by 99%.** Always enable Bucket Keys when using SSE-KMS to avoid per-object KMS charges.

7. **Pre-signed URLs have the permissions of the generator.** If the generating identity loses access, existing pre-signed URLs stop working immediately.

8. **VPC Endpoints eliminate NAT Gateway costs for S3.** Gateway Endpoints are free and keep traffic on the AWS network. No reason not to use them.

9. **Always use both `arn:aws:s3:::bucket` and `arn:aws:s3:::bucket/*` in policies.** Bucket-level actions (ListBucket) use the first ARN; object-level actions (GetObject) use the second. Missing one breaks the other.

10. **Enable access logging on all production buckets.** Without logs, you cannot investigate who accessed what or detect unauthorized access patterns.

---

## 13. Self-Check Questions

1. User A in Account 111 wants to read objects from a bucket in Account 222. What is required?
   > Two things: (1) Account 222's bucket policy must grant access to Account 111's user/role, and (2) User A must have an IAM policy in Account 111 allowing `s3:GetObject` on the bucket. Cross-account access requires both the resource policy AND the identity policy to grant access.

2. You enable Block Public Access at the account level. A developer puts a bucket policy with `"Principal": "*"` on a bucket. Does the bucket become public?
   > No. Block Public Access overrides the bucket policy. Even though the policy grants public access, BPA blocks it. The developer would need to disable BPA on that specific bucket AND at the account level for public access to work.

3. What is the difference between SSE-S3 and SSE-KMS? When would you choose one over the other?
   > SSE-S3: AWS manages everything, no audit trail, no extra cost. SSE-KMS: keys managed by KMS, full CloudTrail audit trail of key usage, key rotation, per-key access policies, but incurs KMS request charges. Choose SSE-KMS for compliance/regulated data where you need to prove who accessed what. Choose SSE-S3 for everything else.

4. A pre-signed URL was generated with 24-hour expiry. The generating user's IAM policy is changed to deny S3 access 2 hours later. Can the URL still be used?
   > No. Pre-signed URLs are validated at request time against the current permissions of the signing identity. If the identity loses access, the URL immediately stops working, regardless of its expiration time.

5. Your EC2 instance accesses S3 through a NAT Gateway. You are paying $0.045/GB for NAT data processing. How do you eliminate this cost?
   > Create a Gateway VPC Endpoint for S3 and add it to the route table used by the instance's subnet. S3 traffic will route through the endpoint (free) instead of the NAT Gateway. The instance no longer needs internet access for S3 operations.

6. You want to ensure no one can delete objects from a critical bucket, not even administrators. How?
   > Enable MFA Delete on the bucket (requires versioning). With MFA Delete, permanently deleting object versions or changing versioning state requires MFA authentication — even the root account must provide an MFA code. Additionally, use an S3 Object Lock in Governance or Compliance mode to prevent deletion for a specified retention period.

---

## References

- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [Bucket Policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html)
- [Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [S3 Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html)
- [VPC Endpoints for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/privatelink-interface-endpoints.html)
- [S3 Access Points](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points.html)
- [Pre-Signed URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [S3 Access Logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html)
- [IAM Policies for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-with-s3-actions.html)
