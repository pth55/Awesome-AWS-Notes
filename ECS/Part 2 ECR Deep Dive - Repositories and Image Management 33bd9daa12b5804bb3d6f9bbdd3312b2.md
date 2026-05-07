# Part 2: ECR Deep Dive - Repositories and Image Management

---

## Table of Contents

1. [Creating ECR Repositories](#1-creating-ecr-repositories)
2. [Pushing Images to ECR — Step by Step](#2-pushing-images-to-ecr--step-by-step)
3. [Pulling Images from ECR](#3-pulling-images-from-ecr)
4. [ECR Image Scanning](#4-ecr-image-scanning)
5. [Lifecycle Policies — Automated Image Cleanup](#5-lifecycle-policies--automated-image-cleanup)
6. [Cross-Region Replication](#6-cross-region-replication)
7. [Cross-Account Access](#7-cross-account-access)
8. [Repository Policies vs IAM Policies](#8-repository-policies-vs-iam-policies)
9. [ECR VPC Endpoints (PrivateLink)](#9-ecr-vpc-endpoints-privatelink)
10. [Image Tag Mutability](#10-image-tag-mutability)
11. [ECR Cost Optimization](#11-ecr-cost-optimization)
12. [Troubleshooting Common ECR Issues](#12-troubleshooting-common-ecr-issues)

---

## 1. Creating ECR Repositories

### Repository Creation Options

You can create repositories via:
- AWS Console
- AWS CLI
- CloudFormation/Terraform
- Automatically (ECS can create on first push if allowed)

### Using AWS Console

```
AWS Console → ECR → Repositories → Create repository

Settings:
─────────────────────────────────────────────────────────
Visibility:   Private (or Public for ECR Public)
Repository name:  my-web-app
Tag immutability:  Disabled (mutable tags)
Image scanning:   Scan on push (enabled)
Encryption:       AES-256 (default)
─────────────────────────────────────────────────────────
```

**What each setting means:**

**Visibility:**
- **Private** — Accessible only with IAM permissions within your AWS account (or cross-account if you grant access)
- **Public** — Accessible to anyone on the internet via `public.ecr.aws` (separate from account-private ECR)

**Tag Immutability:**
- **Disabled (mutable)** — You can overwrite a tag. `docker push my-app:v1.0` twice overwrites the first push.
- **Enabled (immutable)** — Once a tag is pushed, it cannot be overwritten. Prevents accidental overwrites. Recommended for production.

**Image Scanning:**
- **Scan on push** — Automatically scans for vulnerabilities when a new image is pushed
- **Manual** — Scan only when you trigger it explicitly
- **Enhanced scanning (Inspector)** — Uses Amazon Inspector for deeper OS and language package vulnerability detection

**Encryption:**
- **AES-256** — Encryption at rest using AWS-managed keys (default, no extra cost)
- **KMS** — Encryption using customer-managed KMS keys (for compliance/audit requirements, additional KMS costs)

### Using AWS CLI

```bash
aws ecr create-repository \
  --repository-name my-web-app \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1

# With tag immutability enabled
aws ecr create-repository \
  --repository-name my-api-service \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1

# With KMS encryption
aws ecr create-repository \
  --repository-name secure-app \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123456789012:key/abc-123 \
  --region us-east-1
```

### Repository Naming Rules

```
Valid characters: lowercase a-z, 0-9, hyphens (-), underscores (_), forward slashes (/)
Max length: 256 characters

Valid names:
────────────
my-app
api-service
team-a/web-app
project/backend/auth-service

Invalid names:
────────────
My-App                 (capital letters not allowed)
my app                 (spaces not allowed)
my_app_v2              (no versioning in names — use tags)
123app                 (must start with letter — actually allowed but not best practice)
```

**Namespace pattern with slashes:**

```
team-a/frontend
team-a/backend
team-b/ml-inference
team-b/data-pipeline

Benefits:
─────────
✓ Logical grouping
✓ Easier to manage permissions (IAM policies by namespace)
✓ Clear ownership
```

### Repository Limits

| Limit | Default | Can Increase? |
|:------|:--------|:--------------|
| Repositories per account per region | 10,000 | Yes (service quota increase) |
| Images per repository | Unlimited | N/A |
| Maximum image size | 10 GB per layer | No (hard limit) |
| Maximum layers per image | 127 | No (Docker limitation) |
| Concurrent image pushes | 10 per repository | No |

---

## 2. Pushing Images to ECR — Step by Step

### Prerequisites

```bash
# 1. Install AWS CLI
aws --version

# 2. Configure AWS credentials
aws configure
# Enter: Access Key, Secret Key, Region, Output format

# 3. Install Docker
docker --version
```

### Step 1: Authenticate Docker to ECR

```bash
# Get authentication token and log Docker into ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Expected output:
# Login Succeeded
```

**What happens:**
```
1. aws ecr get-login-password
   → Calls GetAuthorizationToken API
   → Returns base64-encoded token valid for 12 hours

2. docker login
   → Stores credentials in ~/.docker/config.json
   → Used by docker push/pull commands
```

### Step 2: Build Your Docker Image

Create a simple application:

```bash
mkdir my-app && cd my-app
```

**Dockerfile:**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Run application
CMD ["node", "server.js"]
```

**server.js:**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from ECR!\n');
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

**package.json:**
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "Simple Node.js app",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {}
}
```

**Build the image:**

```bash
docker build -t my-app:v1.0.0 .

# Expected output shows each layer being built:
# Step 1/7 : FROM node:18-alpine
# Step 2/7 : WORKDIR /app
# ...
# Successfully built abc123def456
# Successfully tagged my-app:v1.0.0
```

### Step 3: Tag Image for ECR

ECR requires images to be tagged with the full repository URI.

```bash
# Format: [account-id].dkr.ecr.[region].amazonaws.com/[repo]:[tag]

docker tag my-app:v1.0.0 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0

# Verify the tag
docker images | grep my-app

# Output:
# my-app         v1.0.0    abc123def456   2 minutes ago   150MB
# 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app   v1.0.0   abc123def456   2 minutes ago   150MB
```

**Important:** Both tags point to the same image (same IMAGE ID). Tagging does not duplicate the image on disk.

### Step 4: Push to ECR

```bash
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0

# Expected output:
# The push refers to repository [123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app]
# abc123: Preparing
# def456: Preparing
# ...
# abc123: Pushed
# def456: Pushed
# v1.0.0: digest: sha256:789abc... size: 1234
```

### Step 5: Verify in ECR

```bash
# List images in repository
aws ecr describe-images \
  --repository-name my-app \
  --region us-east-1

# Output (JSON):
{
  "imageDetails": [
    {
      "registryId": "123456789012",
      "repositoryName": "my-app",
      "imageDigest": "sha256:789abc...",
      "imageTags": ["v1.0.0"],
      "imageSizeInBytes": 157286400,
      "imagePushedAt": "2026-04-29T10:15:30.000Z",
      "imageScanStatus": {
        "status": "COMPLETE",
        "description": "The scan completed successfully."
      }
    }
  ]
}
```

### Multi-Tagging the Same Image

Push multiple tags for the same image:

```bash
# Tag with semantic version
docker tag my-app:v1.0.0 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0

# Tag with git commit SHA
docker tag my-app:v1.0.0 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:abc123

# Tag as latest
docker tag my-app:v1.0.0 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# Push all tags
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:abc123
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

All three tags point to the same image digest. ECR stores only one copy (content-addressed storage). You are not charged three times for storage.

### Push Optimization

Docker pushes only the layers that don't already exist in ECR. If you push a new version with only code changes:

```
First push (my-app:v1.0.0):
─────────────────────────────
Layer 1: node:18-alpine base (50 MB)   → Pushed
Layer 2: npm dependencies (30 MB)      → Pushed
Layer 3: application code (5 MB)       → Pushed
Total uploaded: 85 MB

Second push (my-app:v1.0.1) — only code changed:
─────────────────────────────────────────────────
Layer 1: node:18-alpine base (50 MB)   → Already exists (skipped)
Layer 2: npm dependencies (30 MB)      → Already exists (skipped)
Layer 3: application code (5 MB)       → Pushed (changed)
Total uploaded: 5 MB  ← 17x faster
```

---

## 3. Pulling Images from ECR

### From Your Local Machine

```bash
# 1. Authenticate (if not already)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# 2. Pull the image
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0

# 3. Run the container
docker run -d -p 3000:3000 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0

# 4. Test
curl localhost:3000
# Output: Hello from ECR!
```

### From ECS (Automatic)

ECS automatically pulls images from ECR using the Task Execution Role. No manual authentication needed.

**Task Definition (JSON snippet):**
```json
{
  "family": "my-app-task",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0",
      "memory": 512,
      "cpu": 256,
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

ECS uses the `executionRoleArn` to:
1. Authenticate to ECR
2. Pull the image
3. Write logs to CloudWatch

The role must have these permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

### From EC2 Instance

```bash
# On EC2 instance (must have IAM role with ECR permissions)

# 1. Install Docker
sudo yum install -y docker
sudo service docker start

# 2. Authenticate to ECR
aws ecr get-login-password --region us-east-1 | \
  sudo docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. Pull and run
sudo docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
sudo docker run -d -p 80:3000 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
```

---

## 4. ECR Image Scanning

ECR provides automated vulnerability scanning for known CVEs (Common Vulnerabilities and Exposures) in your container images.

### Two Scanning Types

**1. Basic Scanning (Clair-based)**
- Uses the open-source Clair scanner
- Scans OS packages and language libraries
- Free for up to 10,000 scans per month per account
- Provides severity ratings: Critical, High, Medium, Low, Informational

**2. Enhanced Scanning (Amazon Inspector)**
- Uses Amazon Inspector integration
- Deeper analysis of OS packages and language dependencies
- Continuous scanning (rescans images automatically as new CVE data is released)
- Additional cost (~$0.09 per image scan)
- Better detection accuracy

### Enabling Scan on Push

```bash
# When creating repository
aws ecr create-repository \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1

# For existing repository
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

### Manual Scan

```bash
# Scan a specific image
aws ecr start-image-scan \
  --repository-name my-app \
  --image-id imageTag=v1.0.0 \
  --region us-east-1

# Check scan status
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=v1.0.0 \
  --region us-east-1
```

### Understanding Scan Results

```json
{
  "imageScanFindings": {
    "findings": [
      {
        "name": "CVE-2023-12345",
        "severity": "HIGH",
        "uri": "https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-12345",
        "attributes": [
          {
            "key": "package_name",
            "value": "openssl"
          },
          {
            "key": "package_version",
            "value": "1.1.1k"
          }
        ]
      }
    ],
    "findingSeverityCounts": {
      "CRITICAL": 2,
      "HIGH": 5,
      "MEDIUM": 10,
      "LOW": 8
    }
  }
}
```

**Severity levels:**

| Severity | Score (CVSS) | Action Required |
|:---------|:-------------|:----------------|
| CRITICAL | 9.0 - 10.0 | Immediate fix — do not deploy to production |
| HIGH | 7.0 - 8.9 | Fix before production deployment |
| MEDIUM | 4.0 - 6.9 | Plan to fix in next release cycle |
| LOW | 0.1 - 3.9 | Monitor, fix if possible |
| INFORMATIONAL | 0.0 | No action needed (not a vulnerability) |

### Blocking Deployments Based on Scan Results

Use EventBridge to trigger actions based on scan findings:

```json
{
  "source": ["aws.ecr"],
  "detail-type": ["ECR Image Scan"],
  "detail": {
    "scan-status": ["COMPLETE"],
    "finding-severity-counts": {
      "CRITICAL": [{"numeric": [">", 0]}]
    }
  }
}
```

Trigger Lambda to:
- Send SNS notification to security team
- Update a tag on the image to mark it as "vulnerable"
- Block ECS deployment if critical vulnerabilities exist

### Best Practices for Scan Findings

```
1. Scan early and often
   ─────────────────────
   ✓ Scan on every push
   ✓ Rescan production images weekly
   ✓ Use enhanced scanning for production

2. Prioritize by severity
   ─────────────────────
   Critical → fix immediately
   High     → fix before prod deployment
   Medium   → plan fix in next sprint
   Low      → backlog

3. Update base images regularly
   ─────────────────────
   FROM node:18-alpine     ← vulnerabilities found in node:18-alpine:v1
   Update to: node:18.16-alpine  ← newer patched version

4. Use minimal base images
   ─────────────────────
   ❌ FROM ubuntu:20.04    (100+ packages, more vulnerabilities)
   ✅ FROM alpine:3.18     (minimal packages, smaller attack surface)
   ✅ FROM scratch         (no OS packages — only your app binary)

5. Don't ignore findings
   ─────────────────────
   ✓ Review all findings
   ✓ Assess applicability (is the vulnerable package actually used?)
   ✓ Document exceptions if a finding is false positive
```

---

## 5. Lifecycle Policies — Automated Image Cleanup

Over time, ECR repositories accumulate hundreds of old/unused images, increasing storage costs. Lifecycle policies automatically delete images based on rules you define.

### How Lifecycle Policies Work

```
Evaluation happens daily (overnight):
─────────────────────────────────────
1. ECR evaluates all images in repository against lifecycle rules
2. Images matching rule conditions are marked for expiration
3. After retention period, images are deleted
4. Storage costs drop
```

### Lifecycle Policy Structure

A lifecycle policy is a JSON document with rules:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

### Common Lifecycle Policy Examples

**Example 1: Keep only the last 10 images**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Retain only 10 most recent images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

**Example 2: Delete untagged images older than 7 days**

Untagged images are created when you overwrite a mutable tag.

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Delete untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

**Example 3: Keep production tags forever, expire dev tags after 30 days**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep production tags indefinitely",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod", "v"],
        "countType": "imageCountMoreThan",
        "countNumber": 100
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Expire dev tags after 30 days",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["dev", "feature"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

### Applying a Lifecycle Policy

```bash
# Save policy to a JSON file
cat > lifecycle-policy.json <<EOF
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
EOF

# Apply the policy
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region us-east-1

# Preview what would be deleted (dry run)
aws ecr start-lifecycle-policy-preview \
  --repository-name my-app \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region us-east-1

# Check preview results
aws ecr get-lifecycle-policy-preview \
  --repository-name my-app \
  --region us-east-1
```

### Rule Priority

If an image matches multiple rules, the **lowest priority number** wins.

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep prod images forever",
      "selection": {
        "tagPrefixList": ["prod"],
        ...
      }
    },
    {
      "rulePriority": 2,
      "description": "Delete all images older than 90 days",
      "selection": {
        "tagStatus": "any",
        ...
      }
    }
  ]
}
```

An image tagged `prod-v1.0` matches both rules. Rule 1 (priority 1) wins → image is kept forever.

---

## 6. Cross-Region Replication

ECR can automatically replicate images to other AWS regions for:
- Disaster recovery
- Low-latency access (pull images from nearest region)
- Multi-region deployments

### Replication Configuration

```bash
# Create replication configuration
cat > replication-config.json <<EOF
{
  "rules": [
    {
      "destinations": [
        {
          "region": "us-west-2",
          "registryId": "123456789012"
        },
        {
          "region": "eu-west-1",
          "registryId": "123456789012"
        }
      ],
      "repositoryFilters": [
        {
          "filter": "my-app",
          "filterType": "PREFIX_MATCH"
        }
      ]
    }
  ]
}
EOF

# Apply replication configuration
aws ecr put-replication-configuration \
  --replication-configuration file://replication-config.json \
  --region us-east-1
```

**What happens:**

```
Image pushed to us-east-1:
  my-app:v1.0.0
    ↓
ECR automatically replicates to:
  ├── us-west-2: 123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app:v1.0.0
  └── eu-west-1: 123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:v1.0.0

Replication happens asynchronously (usually < 1 minute per image).
```

### Costs for Replication

- **Storage** — You pay for storage in each region (duplicate storage costs)
- **Data transfer** — Cross-region data transfer charges apply ($0.02/GB from us-east-1 to us-west-2)
- **No replication fee** — ECR does not charge extra for the replication feature itself

---

## 7. Cross-Account Access

Share ECR images with other AWS accounts without copying them.

### Use Case

```
Account A (123456789012): Builds and stores images
Account B (987654321098): Runs ECS services that pull images from Account A
```

### Method 1: Repository Policy (Recommended)

**In Account A (repository owner):**

```bash
cat > repository-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPullFromAccountB",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:root"
      },
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }
  ]
}
EOF

# Apply policy to repository
aws ecr set-repository-policy \
  --repository-name my-app \
  --policy-text file://repository-policy.json \
  --region us-east-1
```

**In Account B (consuming account):**

ECS task execution role needs permission to get auth token:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/my-app"
    }
  ]
}
```

**ECS Task Definition in Account B:**

```json
{
  "family": "cross-account-task",
  "executionRoleArn": "arn:aws:iam::987654321098:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0",
      ...
    }
  ]
}
```

### Method 2: IAM Role Assumption

Account B assumes a role in Account A:

**In Account A:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Attach ECR permissions to the assumable role.

**In Account B:**

Assume the role, then pull images using temporary credentials.

---

## 8. Repository Policies vs IAM Policies

Two ways to control access to ECR: **Repository Policies** (resource-based) and **IAM Policies** (identity-based).

### Repository Policy

- Attached to the repository itself
- Specifies WHO can access this repository
- Used for cross-account access
- JSON document attached via `set-repository-policy`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccount",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:root"
      },
      "Action": [
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ]
    }
  ]
}
```

### IAM Policy

- Attached to IAM users/roles/groups
- Specifies WHAT this identity can access
- Used for same-account access control

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

### When to Use Which

| Scenario | Use |
|:---------|:----|
| Grant access to users/roles in the same account | IAM Policy |
| Grant access to another AWS account | Repository Policy |
| Grant access to specific repositories only | IAM Policy with Resource ARN |
| Public access (ECR Public) | Repository Policy with Principal: "*" |

---

## 9. ECR VPC Endpoints (PrivateLink)

By default, Docker CLI communicates with ECR over the internet. For security/compliance, you can access ECR entirely through private IPs within your VPC using **VPC Endpoints** (AWS PrivateLink).

### Why Use VPC Endpoints?

- **No internet gateway needed** — traffic never leaves AWS network
- **Security** — data does not traverse the public internet
- **Compliance** — meet requirements for private-only connectivity
- **Cost savings** — avoid NAT Gateway data processing charges

### Required VPC Endpoints for ECR

You need **three** VPC endpoints:

```
1. com.amazonaws.[region].ecr.dkr
   → Docker registry API (docker push/pull commands)

2. com.amazonaws.[region].ecr.api
   → ECR control plane API (aws ecr describe-repositories, etc.)

3. com.amazonaws.[region].s3 (Gateway endpoint)
   → ECR stores image layers in S3 behind the scenes
```

### Creating VPC Endpoints

```bash
# Get your VPC ID and subnet IDs
VPC_ID="vpc-abc123"
SUBNET_IDS="subnet-111,subnet-222"
SECURITY_GROUP_ID="sg-xyz789"

# 1. Create ECR DKR endpoint (Interface endpoint)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --subnet-ids $SUBNET_IDS \
  --security-group-ids $SECURITY_GROUP_ID \
  --region us-east-1

# 2. Create ECR API endpoint (Interface endpoint)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --subnet-ids $SUBNET_IDS \
  --security-group-ids $SECURITY_GROUP_ID \
  --region us-east-1

# 3. Create S3 Gateway endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-abc123 \
  --region us-east-1
```

### Security Group Rules for VPC Endpoints

The security group attached to ECR VPC endpoints must allow:

```
Inbound Rules:
──────────────
Type: HTTPS
Protocol: TCP
Port: 443
Source: VPC CIDR (e.g., 10.0.0.0/16)

This allows instances in your VPC to reach the endpoint.
```

### DNS Configuration

ECR VPC endpoints require **private DNS** enabled:

```bash
# Enable private DNS on the endpoint
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-abc123 \
  --private-dns-enabled \
  --region us-east-1
```

With private DNS enabled, requests to `123456789012.dkr.ecr.us-east-1.amazonaws.com` automatically resolve to the private IP of the VPC endpoint.

---

## 10. Image Tag Mutability

### Mutable Tags (Default)

You can push a new image with the same tag, overwriting the previous one.

```bash
# First push
docker build -t my-app:latest .
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
# Image ABC pushed

# Second push (later, after code change)
docker build -t my-app:latest .
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
# Image XYZ pushed — previous image ABC becomes UNTAGGED

Result:
  my-app:latest → now points to image XYZ
  Previous image ABC still exists but has no tag (untagged)
```

**Problem:**
- You lose track of what was deployed
- Rollbacks are difficult (which image was production?)
- Auditing is harder

### Immutable Tags

Once a tag is pushed, it cannot be changed. Attempts to push the same tag again are rejected.

```bash
# Enable tag immutability
aws ecr put-image-tag-mutability \
  --repository-name my-app \
  --image-tag-mutability IMMUTABLE \
  --region us-east-1

# Now try to push the same tag twice:
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
# SUCCESS

docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
# ERROR: Image tag already exists
```

**Benefits:**
- Cannot accidentally overwrite production tags
- Full audit trail (every push creates a new tag)
- Safer deployments

**Recommendation:**
- Use immutable tags for production repositories
- Use mutable tags for development repositories (convenience of reusing `dev`, `latest`)

---

## 11. ECR Cost Optimization

ECR charges for:
1. **Storage** — $0.10 per GB/month
2. **Data transfer** — Standard AWS data transfer pricing

### Cost Optimization Strategies

**1. Use Lifecycle Policies**

Delete old/unused images automatically:
```
✓ Keep only last 10 images
✓ Delete untagged images after 7 days
✓ Expire dev/feature tags after 30 days
```

Savings example:
```
Before: 500 images × 200 MB avg = 100 GB → $10/month
After:  50 images × 200 MB avg = 10 GB → $1/month
Savings: $9/month per repository
```

**2. Minimize Base Image Size**

```
❌ FROM ubuntu:20.04 → 72 MB
❌ FROM node:18      → 910 MB

✅ FROM alpine:3.18  → 7 MB
✅ FROM node:18-alpine → 170 MB

Savings: 740 MB per image
If you have 100 image tags: 74 GB saved → $7.40/month
```

**3. Use Multi-Stage Builds**

```dockerfile
# Build stage (large)
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci
RUN npm run build

# Production stage (small)
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]

Result:
  Builder stage: 1.2 GB (not pushed to ECR)
  Final image:   180 MB (only this is pushed)
```

**4. Regional Colocation**

Avoid cross-region data transfer:
```
ECS cluster in us-east-1 → Pull from ECR in us-east-1 (free)
ECS cluster in us-west-2 → Pull from ECR in us-east-1 ($0.02/GB)

Solution: Use replication or build images in each region
```

**5. Monitor with CloudWatch**

Track metrics:
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECR \
  --metric-name RepositoryPullCount \
  --dimensions Name=RepositoryName,Value=my-app \
  --start-time 2026-04-01T00:00:00Z \
  --end-time 2026-04-30T23:59:59Z \
  --period 86400 \
  --statistics Sum \
  --region us-east-1
```

Identify unused repositories → delete them.

---

## 12. Troubleshooting Common ECR Issues

### Issue 1: Authentication Failed

**Error:**
```
Error response from daemon: Get "https://123456789012.dkr.ecr.us-east-1.amazonaws.com/v2/": 
no basic auth credentials
```

**Cause:** Not authenticated to ECR.

**Solution:**
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
```

### Issue 2: Permission Denied

**Error:**
```
denied: User: arn:aws:iam::123456789012:user/john is not authorized to perform: 
ecr:PutImage on resource: arn:aws:ecr:us-east-1:123456789012:repository/my-app
```

**Cause:** IAM user/role lacks ECR permissions.

**Solution:** Attach policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "*"
    }
  ]
}
```

### Issue 3: Repository Does Not Exist

**Error:**
```
An error occurred (RepositoryNotFoundException) when calling the PutImage operation: 
The repository with name 'my-app' does not exist
```

**Cause:** Repository not created.

**Solution:**
```bash
aws ecr create-repository --repository-name my-app --region us-east-1
```

### Issue 4: ECS Cannot Pull Image

**Error in ECS task:**
```
CannotPullContainerError: Error response from daemon: pull access denied for 
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app, repository does not exist 
or may require 'docker login'
```

**Cause:** Task execution role lacks ECR pull permissions.

**Solution:** Attach `AmazonECSTaskExecutionRolePolicy` to execution role.

### Issue 5: Image Too Large

**Error:**
```
Error response from daemon: layer size exceeds maximum allowed size
```

**Cause:** ECR has a 10 GB per layer limit.

**Solution:** Reduce image size:
- Use smaller base images
- Split large files across multiple layers
- Remove build artifacts from final image (multi-stage build)

---

**End of Part 2 — ECR Deep Dive**
