# Part 6 — DynamoDB Security, Backup & Recovery

## Table of Contents

1. [IAM Authentication and Authorization](#1-iam-authentication-and-authorization)
2. [Fine-Grained Access Control](#2-fine-grained-access-control)
3. [VPC Endpoints](#3-vpc-endpoints)
4. [Encryption at Rest](#4-encryption-at-rest)
5. [Encryption in Transit](#5-encryption-in-transit)
6. [Point-in-Time Recovery (PITR)](#6-point-in-time-recovery-pitr)
7. [On-Demand Backups](#7-on-demand-backups)
8. [Export to S3](#8-export-to-s3)
9. [AWS Backup Integration](#9-aws-backup-integration)
10. [Compliance and Auditing](#10-compliance-and-auditing)
11. [Security Checklist](#11-security-checklist)

---

## 1. IAM Authentication and Authorization

DynamoDB has no concept of database users. All access is controlled through **IAM**. Every API call to DynamoDB must include AWS credentials (access key + secret, IAM role, or web identity token) that are authorized by an IAM policy.

### Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrderTableReadWrite",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:BatchGetItem",
        "dynamodb:BatchWriteItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:123456789012:table/Orders",
        "arn:aws:dynamodb:us-east-1:123456789012:table/Orders/index/*"
      ]
    }
  ]
}
```

The `/index/*` suffix is required to allow `Query` on GSIs. Without it, queries against GSIs will return `AccessDeniedException`.

### Commonly Used DynamoDB Actions

| Category | Actions |
|---|---|
| Item operations | `GetItem`, `PutItem`, `UpdateItem`, `DeleteItem` |
| Batch operations | `BatchGetItem`, `BatchWriteItem` |
| Query and scan | `Query`, `Scan` |
| Table management | `CreateTable`, `DeleteTable`, `UpdateTable`, `DescribeTable` |
| Index management | `UpdateTable` (includes GSI creation/deletion) |
| Stream access | `ListStreams`, `DescribeStream`, `GetShardIterator`, `GetRecords` |
| Backup | `CreateBackup`, `DeleteBackup`, `RestoreTableFromBackup`, `ListBackups` |
| Export | `ExportTableToPointInTime` |

### Least-Privilege Role Pattern

Applications should have read-only or scoped roles. Never give an application `dynamodb:*` on all resources.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:Query", "dynamodb:GetItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/UserProfiles",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:Attributes": ["UserId", "Email", "DisplayName"]
        },
        "StringEquals": {
          "dynamodb:Select": "SPECIFIC_ATTRIBUTES"
        }
      }
    }
  ]
}
```

---

## 2. Fine-Grained Access Control

DynamoDB supports IAM condition keys that let you restrict access based on item-level attribute values. This is useful for multi-tenant tables where different users should only see their own data.

### Condition Keys

| Condition Key | Description |
|---|---|
| `dynamodb:LeadingKeys` | Restricts access to items where the partition key matches a specific value |
| `dynamodb:Attributes` | Restricts which attributes a caller can read or write |
| `dynamodb:Select` | Restricts the `Select` parameter value (`ALL_ATTRIBUTES`, `SPECIFIC_ATTRIBUTES`, etc.) |
| `dynamodb:ReturnValues` | Restricts which return value types are allowed |

### Per-User Data Isolation

This pattern is common in web applications backed by Amazon Cognito. The IAM policy uses `${aws:userid}` or `${cognito-identity.amazonaws.com:sub}` to enforce that a user can only access their own partition key:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/UserData",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
        }
      }
    }
  ]
}
```

With this policy, a user authenticated through Cognito Identity Pools can only read and write items whose partition key equals their own Cognito identity ID. Attempting to access another user's partition key returns `AccessDeniedException`.

### Attribute-Level Restrictions

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:Query"],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Employees",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:Attributes": ["EmployeeId", "Name", "Department", "Email"]
    }
  }
}
```

This blocks access to sensitive fields like `Salary` or `SocialSecurityNumber` at the policy level, even if the caller requests `ALL_ATTRIBUTES`.

---

## 3. VPC Endpoints

By default, DynamoDB is a public AWS service — your EC2 instances or Lambda functions connect to it over the public internet (though through AWS's network backbone). A **VPC Gateway Endpoint** for DynamoDB keeps traffic within the AWS network and removes the need for NAT Gateway or internet connectivity.

### Creating a VPC Gateway Endpoint

```bash
# Get your VPC and route table IDs
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=false" \
  --query "Vpcs[0].VpcId" --output text)

ROUTE_TABLE_IDS=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "RouteTables[*].RouteTableId" --output text)

# Create the Gateway Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --route-table-ids $ROUTE_TABLE_IDS \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": "dynamodb:*",
      "Resource": "*"
    }]
  }'
```

The endpoint policy on the VPC endpoint controls which DynamoDB resources are accessible through that endpoint. You can restrict it to specific table ARNs to prevent exfiltration to DynamoDB tables in other AWS accounts.

### Restrictive Endpoint Policy (Data Exfiltration Prevention)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "dynamodb:*",
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "dynamodb:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:ResourceAccount": "123456789012"
        }
      }
    }
  ]
}
```

This allows access only to DynamoDB tables in account `123456789012`, preventing compromised workloads from writing data to an attacker's DynamoDB table in another account.

---

## 4. Encryption at Rest

All DynamoDB tables are encrypted at rest. You choose the key management model when you create the table.

### Encryption Key Types

| Type | Key Management | Cost | Use Case |
|---|---|---|---|
| **AWS-owned key** (default) | AWS manages entirely, not visible in your account | Free | Standard workloads |
| **AWS-managed key** (`aws/dynamodb`) | KMS key in your account, AWS manages rotation | ~$1/month for the key | Audit visibility required |
| **Customer-managed key (CMK)** | You create and control key in KMS | ~$1/month key + $0.03/10K API calls | BYOK, custom rotation, cross-account sharing |

The default (AWS-owned key) provides strong encryption without any cost or management. Use CMK only when regulatory requirements mandate customer-controlled keys or when you need to revoke access by disabling the key.

### Specifying Encryption on Table Creation

```bash
# Default (AWS-owned key) — no SSESpecification needed
aws dynamodb create-table --table-name Orders ...

# AWS-managed key
aws dynamodb create-table --table-name Orders ... \
  --sse-specification Enabled=true,SSEType=KMS

# Customer-managed key
aws dynamodb create-table --table-name Orders ... \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/my-dynamodb-key

# Update an existing table to use a CMK
aws dynamodb update-table \
  --table-name Orders \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/my-dynamodb-key
```

### KMS Key Policy for DynamoDB

When using a CMK, the KMS key policy must allow DynamoDB to use the key:

```json
{
  "Sid": "AllowDynamoDB",
  "Effect": "Allow",
  "Principal": {
    "Service": "dynamodb.amazonaws.com"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt*",
    "kms:GenerateDataKey*",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

---

## 5. Encryption in Transit

All communication between your application and DynamoDB endpoints is encrypted with **TLS 1.2+** by default. AWS SDKs enforce this automatically. There are no configuration options to disable it for DynamoDB (unlike some other AWS services). Ensure that your application does not pin to old TLS versions.

---

## 6. Point-in-Time Recovery (PITR)

PITR continuously backs up your table to S3 (managed by DynamoDB, not directly accessible). You can restore to any second within the past **35 days**.

### Key Characteristics

- PITR runs continuously — there is no scheduled backup window.
- The backup is stored in DynamoDB-managed S3 buckets in the same region. You cannot access these buckets directly.
- Restoring from PITR creates a **new table**. It does not overwrite the source table.
- The new table does not inherit Streams settings, Auto Scaling policies, TTL configuration, or tags from the source — these must be reconfigured after restoration.
- PITR does not protect against Global Tables replication of accidental deletes. If you delete an item, the deletion replicates to all regions. PITR must be enabled and restored per-region.

### Enabling and Managing PITR

```bash
# Enable PITR
aws dynamodb update-continuous-backups \
  --table-name Orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# Check status — shows EarliestRestorableDateTime and LatestRestorableDateTime
aws dynamodb describe-continuous-backups --table-name Orders

# Restore to a specific point in time
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name Orders-Restored-20241201 \
  --restore-date-time 2024-12-01T10:00:00Z \
  --use-latest-restorable-time  # Use this flag to restore to the most recent point
```

### Restore Options

You can modify certain properties during restore:

```bash
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name Orders-Restored \
  --restore-date-time 2024-12-01T10:00:00Z \
  --billing-mode-override PAY_PER_REQUEST \
  --sse-specification-override Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/new-key \
  --global-secondary-index-override '[]'  # Restore without GSIs to speed up restore
```

Restoring without GSIs and then adding them after restore can significantly reduce restore time for large tables.

### PITR Cost

PITR storage cost: **$0.20 per GB-month** for the backup data stored. The cost scales with table size and change frequency (more writes = more backup data).

---

## 7. On-Demand Backups

On-demand backups create a full table snapshot at a specific point in time. Unlike PITR, these are discrete snapshots you initiate manually or via AWS Backup. They do not expire automatically — you manage their lifecycle.

### Creating and Restoring On-Demand Backups

```bash
# Create a backup
aws dynamodb create-backup \
  --table-name Orders \
  --backup-name Orders-Backup-20241201

# List all backups for a table
aws dynamodb list-backups \
  --table-name Orders \
  --query "BackupSummaries[*].{Name:BackupName,ARN:BackupArn,Status:BackupStatus}"

# Restore from a backup (creates a new table)
aws dynamodb restore-table-from-backup \
  --target-table-name Orders-Restored \
  --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders/backup/...

# Delete a backup
aws dynamodb delete-backup \
  --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders/backup/...
```

### On-Demand Backup Characteristics

- Backups complete in seconds for any table size (no performance impact on the live table).
- Retained until explicitly deleted.
- Can be restored to a different AWS region (cross-region restore).
- The restored table does not inherit provisioned throughput, Auto Scaling, or Streams settings.
- Cost: **$0.10 per GB-month** for backup storage.

---

## 8. Export to S3

The Export to S3 feature exports a full table snapshot (or a PITR point) to an S3 bucket in **DynamoDB JSON** or **Amazon Ion** format. Unlike on-demand backups, the export lands in your own S3 bucket, making it accessible for Athena queries, Spark jobs, or data archival.

### Running an Export

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --s3-bucket my-dynamodb-exports \
  --s3-prefix orders-export/ \
  --export-time 2024-12-01T00:00:00Z \
  --export-format DYNAMODB_JSON  # or ION

# Check export status
aws dynamodb describe-export \
  --export-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders/export/...
```

### Export Output Format

The export creates files in S3 at this path structure:

```
s3://my-dynamodb-exports/orders-export/AWSDynamoDB/<export-id>/
  ├── manifest-files.json       — list of all data files
  ├── manifest-summary.json     — metadata (item count, size, export time)
  └── data/
      ├── <uuid>.json.gz        — compressed DynamoDB JSON data
      └── <uuid>.json.gz
```

Each line in the data files is a JSON object with an `Item` key:

```json
{"Item":{"OrderId":{"S":"ORD-001"},"Status":{"S":"SHIPPED"},"Amount":{"N":"49.99"}}}
```

### Querying Exports with Athena

```sql
CREATE EXTERNAL TABLE orders_export (
  Item STRUCT<
    OrderId: STRUCT<S: STRING>,
    Status: STRUCT<S: STRING>,
    Amount: STRUCT<N: STRING>
  >
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://my-dynamodb-exports/orders-export/AWSDynamoDB/<export-id>/data/'
TBLPROPERTIES ('ignore.malformed.json' = 'true');

SELECT Item.OrderId.S AS order_id,
       Item.Status.S AS status,
       CAST(Item.Amount.N AS DECIMAL(10,2)) AS amount
FROM orders_export
WHERE Item.Status.S = 'SHIPPED';
```

### Export Cost

- No additional DynamoDB charge for the export operation.
- S3 storage charged at standard S3 rates for the exported data.
- S3 data transfer charges if exporting to a different region.

---

## 9. AWS Backup Integration

AWS Backup provides centralized backup management across DynamoDB, RDS, EFS, EBS, and other services. Use it when you need:
- Consistent backup policies across multiple services.
- Cross-region or cross-account copy of DynamoDB backups.
- Compliance reporting (HIPAA, PCI-DSS) via the AWS Backup audit reports.
- Automated backup retention lifecycle management.

```bash
# Create a backup plan that backs up DynamoDB daily and retains for 30 days
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "DynamoDB-Daily-30Days",
  "Rules": [{
    "RuleName": "DailyBackup",
    "TargetBackupVaultName": "Default",
    "ScheduleExpression": "cron(0 3 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 180,
    "Lifecycle": {
      "DeleteAfterDays": 30
    },
    "CopyActions": [{
      "DestinationBackupVaultArn": "arn:aws:backup:eu-west-1:123456789012:backup-vault:Default",
      "Lifecycle": {"DeleteAfterDays": 30}
    }]
  }]
}'

# Assign DynamoDB tables to this plan by tag
aws backup create-backup-selection \
  --backup-plan-id <plan-id> \
  --backup-selection '{
    "SelectionName": "AllTaggedTables",
    "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole",
    "ListOfTags": [{"ConditionType": "STRINGEQUALS",
                    "ConditionKey": "Backup",
                    "ConditionValue": "true"}]
  }'
```

---

## 10. Compliance and Auditing

### CloudTrail

All DynamoDB management API calls (create table, update table, enable PITR, etc.) are logged in **AWS CloudTrail**. Data plane operations (`GetItem`, `PutItem`, etc.) are not logged by default — enabling CloudTrail data events for DynamoDB adds cost but provides item-level audit trails.

```bash
# Enable CloudTrail data events for a specific table
aws cloudtrail put-event-selectors \
  --trail-name my-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::DynamoDB::Table",
      "Values": ["arn:aws:dynamodb:us-east-1:123456789012:table/Orders"]
    }]
  }]'
```

### Useful CloudWatch Metrics for Security

| Metric | Alarm Condition | Meaning |
|---|---|---|
| `SystemErrors` | > 0 | Internal DynamoDB errors — contact AWS Support |
| `UserErrors` | Sudden spike | Misconfigured application making invalid requests |
| `ConditionalCheckFailedRequests` | Unusually high | Possible concurrent modification bugs or attack attempts |
| `SuccessfulRequestLatency` | P99 > 20ms | Latency degradation |
| `ThrottledRequests` | > 0 sustained | Capacity under-provisioned or hot partition |

---

## 11. Security Checklist

| Control | How to Implement |
|---|---|
| Use IAM roles, not long-lived access keys | Attach IAM roles to EC2/Lambda; rotate keys if used |
| Least-privilege IAM policies | Scope to specific table ARNs + `/index/*` for GSIs |
| Fine-grained access for multi-tenant tables | `dynamodb:LeadingKeys` condition tied to Cognito identity |
| Prevent cross-account exfiltration | VPC endpoint policy restricting to your account's resources |
| Encryption with customer-controlled key | CMK via KMS when compliance requires BYOK |
| Enable PITR | Always on for production tables |
| Enable on-demand backups or AWS Backup | For compliance retention schedules |
| CloudTrail data events | For SOC 2 / PCI-DSS audit requirements |
| Tag tables with environment and owner | Enables backup policy targeting and cost attribution |
| Block public access (no internet connectivity for EC2) | Use VPC Gateway Endpoint for DynamoDB |

---

## Key Takeaways

- DynamoDB access is entirely IAM-based — there are no database users, passwords, or connection strings.
- Use `dynamodb:LeadingKeys` for per-user data isolation in multi-tenant applications.
- VPC Gateway Endpoints for DynamoDB are free and keep traffic within the AWS network.
- All tables are encrypted at rest by default. Use CMK only when regulatory requirements mandate customer-controlled keys.
- Enable PITR on all production tables. It costs $0.20/GB-month and allows recovery to any second within 35 days.
- Export to S3 makes DynamoDB data available for Athena SQL queries — useful for analytics without impacting live table performance.
