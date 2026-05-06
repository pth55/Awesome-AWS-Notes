# Part 3 — DocumentDB Transactions, Change Streams, Backup & Security

## Table of Contents

1. [Multi-Document Transactions](#1-multi-document-transactions)
2. [Change Streams](#2-change-streams)
3. [Point-in-Time Recovery and Backups](#3-point-in-time-recovery-and-backups)
4. [Security — Network and Authentication](#4-security--network-and-authentication)
5. [Security — Encryption](#5-security--encryption)
6. [IAM Authentication](#6-iam-authentication)
7. [Audit Logging](#7-audit-logging)
8. [DocumentDB Elastic Clusters](#8-documentdb-elastic-clusters)
9. [Migration to DocumentDB](#9-migration-to-documentdb)

---

## 1. Multi-Document Transactions

DocumentDB supports **multi-document ACID transactions** using the MongoDB 4.0 session-based transactions API. Multiple read/write operations across different collections can be grouped into a single atomic unit.

### Transaction Properties

- **Atomicity**: All operations succeed, or none are applied.
- **Consistency**: The database moves from one valid state to another.
- **Isolation**: Transactions use snapshot isolation — each transaction sees a consistent snapshot of the data as of the transaction start.
- **Durability**: Committed transactions are persisted to the storage layer (6-way replication).

### Transaction Pattern

```python
from pymongo import MongoClient
import certifi

client = MongoClient(
    'mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com',
    port=27017,
    username='admin', password='password',
    tls=True, tlsCAFile=certifi.where(),
    retryWrites=False
)

db = client['ecommerce']
orders = db['orders']
inventory = db['inventory']
wallets = db['wallets']

def place_order(customer_id, items, total_amount):
    with client.start_session() as session:
        with session.start_transaction():
            # Step 1: Deduct inventory for each item
            for item in items:
                result = inventory.update_one(
                    {'productId': item['productId'], 'stock': {'$gte': item['qty']}},
                    {'$inc': {'stock': -item['qty']}},
                    session=session
                )
                if result.modified_count == 0:
                    # Stock insufficient — abort the transaction
                    session.abort_transaction()
                    raise ValueError(f"Insufficient stock for {item['productId']}")

            # Step 2: Deduct from customer wallet
            result = wallets.update_one(
                {'customerId': customer_id, 'balance': {'$gte': total_amount}},
                {'$inc': {'balance': -total_amount}},
                session=session
            )
            if result.modified_count == 0:
                session.abort_transaction()
                raise ValueError("Insufficient wallet balance")

            # Step 3: Create the order record
            orders.insert_one({
                'customerId': customer_id,
                'items': items,
                'total': total_amount,
                'status': 'confirmed',
                'createdAt': '2024-12-01T10:00:00Z'
            }, session=session)

        # Transaction auto-commits when exiting the with block without exception
        # If an exception is raised, it auto-aborts

try:
    place_order('CUST-123', [{'productId': 'PROD-A', 'qty': 2}], 59.98)
    print("Order placed successfully")
except ValueError as e:
    print(f"Order failed: {e}")
```

### Transaction Limitations

- Maximum transaction duration: **60 seconds**. Transactions running longer are automatically aborted.
- Maximum transaction size: **4 MB** of write data per transaction.
- DocumentDB does not support transactions that span multiple clusters.
- Avoid holding transactions open for extended periods — they hold locks and block concurrent writes to the same documents.

---

## 2. Change Streams

Change Streams provide a real-time feed of changes to a collection, database, or cluster. They are similar to DynamoDB Streams in concept but use a cursor-based pull model rather than a Lambda event source.

### What Change Streams Provide

Every document-level change generates a change event with:
- `operationType`: `insert`, `update`, `replace`, `delete`
- `fullDocument`: The document after the change (for insert, replace, update with `fullDocument: 'updateLookup'`)
- `documentKey`: The `_id` of the changed document
- `updateDescription`: Fields that were changed and fields that were removed (for update operations)
- `ns`: The namespace (database + collection)
- `clusterTime`: When the change occurred

### Watching a Collection

```python
import time
from pymongo import MongoClient
import certifi

client = MongoClient(
    'mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com',
    port=27017,
    username='admin', password='password',
    tls=True, tlsCAFile=certifi.where(),
    retryWrites=False
)

db = client['ecommerce']
orders = db['orders']

# Open a change stream on the orders collection
pipeline = [
    {'$match': {'operationType': {'$in': ['insert', 'update']}}}
]

with orders.watch(pipeline=pipeline, full_document='updateLookup') as stream:
    for change in stream:
        op = change['operationType']
        doc = change.get('fullDocument', {})
        order_id = change['documentKey']['_id']

        if op == 'insert':
            print(f"New order: {order_id}, total: {doc.get('total')}")
        elif op == 'update':
            updated_fields = change.get('updateDescription', {}).get('updatedFields', {})
            if 'status' in updated_fields:
                print(f"Order {order_id} status changed to: {updated_fields['status']}")
```

### Resuming After Interruption

Change Streams return a `resume_token` with each event. Store this token and use it to resume the stream from where you left off after a restart:

```python
import json

last_token_file = '/tmp/change_stream_token.json'

def load_token():
    try:
        with open(last_token_file) as f:
            return json.load(f)
    except FileNotFoundError:
        return None

def save_token(token):
    with open(last_token_file, 'w') as f:
        json.dump(token, f, default=str)

resume_token = load_token()

with orders.watch(resume_after=resume_token) as stream:
    for change in stream:
        process_change(change)
        save_token(change['_id'])  # _id is the resume token
```

DocumentDB retains change stream history for a configurable retention period (default 3 hours, configurable up to 7 days via the `change_stream_log_retention_duration` cluster parameter).

### Change Stream vs DynamoDB Streams

| Aspect | DocumentDB Change Streams | DynamoDB Streams |
|---|---|---|
| Consumption model | Pull (cursor-based) | Push (Lambda event source) |
| Retention | 3h–7 days (configurable) | 24 hours (fixed) |
| Filter | Aggregation pipeline (flexible) | EventName filter only |
| Resume | Resume token (any point in retention window) | Shard iterator |
| Fan-out | Multiple consumers can watch independently | One Lambda function per shard |

---

## 3. Point-in-Time Recovery and Backups

### Continuous Backups (PITR)

DocumentDB continuously backs up data to S3 (AWS-managed). PITR is always enabled and retains backups for **1 to 35 days** (configurable).

```bash
# Set backup retention to 14 days
aws docdb modify-db-cluster \
  --db-cluster-identifier my-docdb-cluster \
  --backup-retention-period 14 \
  --preferred-backup-window "03:00-04:00" \
  --apply-immediately

# Restore to a specific point in time (creates a new cluster)
aws docdb restore-db-cluster-to-point-in-time \
  --db-cluster-identifier my-docdb-cluster-restored \
  --source-db-cluster-identifier my-docdb-cluster \
  --restore-to-time 2024-12-01T10:30:00Z \
  --db-subnet-group-name docdb-subnet-group \
  --vpc-security-group-ids sg-xxxxxxxx

# After restore, add an instance to the new cluster
aws docdb create-db-instance \
  --db-instance-identifier my-restored-instance \
  --db-cluster-identifier my-docdb-cluster-restored \
  --db-instance-class db.r6g.large \
  --engine docdb
```

Restoration creates a new cluster — it does not overwrite the source. You must add instances to the restored cluster before it can serve traffic.

### Manual Snapshots

```bash
# Create a manual snapshot
aws docdb create-db-cluster-snapshot \
  --db-cluster-identifier my-docdb-cluster \
  --db-cluster-snapshot-identifier my-snapshot-20241201

# List snapshots
aws docdb describe-db-cluster-snapshots \
  --db-cluster-identifier my-docdb-cluster

# Restore from snapshot
aws docdb restore-db-cluster-from-snapshot \
  --db-cluster-identifier my-docdb-cluster-restored \
  --snapshot-identifier my-snapshot-20241201 \
  --engine docdb \
  --db-subnet-group-name docdb-subnet-group

# Copy snapshot to another region
aws docdb copy-db-cluster-snapshot \
  --source-db-cluster-snapshot-identifier arn:aws:rds:us-east-1:123456789012:cluster-snapshot:my-snapshot-20241201 \
  --target-db-cluster-snapshot-identifier my-snapshot-eu \
  --region eu-west-1
```

Manual snapshots are retained until explicitly deleted. Automated backups are deleted when the cluster is deleted (unless you enable the `--no-skip-final-snapshot` flag).

---

## 4. Security — Network and Authentication

### VPC and Security Groups

DocumentDB clusters must reside in a VPC. The cluster's instances are not publicly accessible by default — they require VPC connectivity.

```bash
# Create a security group for DocumentDB
aws ec2 create-security-group \
  --group-name docdb-sg \
  --description "DocumentDB access" \
  --vpc-id vpc-xxxxxxxx

DOCDB_SG=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=docdb-sg \
  --query "SecurityGroups[0].GroupId" --output text)

# Allow inbound MongoDB port (27017) only from application security group
aws ec2 authorize-security-group-ingress \
  --group-id $DOCDB_SG \
  --protocol tcp \
  --port 27017 \
  --source-group sg-app-security-group-id
```

DocumentDB does not support a VPC Gateway Endpoint (unlike DynamoDB). Traffic routes through the VPC's internal networking.

### User Management

DocumentDB uses role-based access control similar to MongoDB. Users are created within the database and have roles that define their permissions.

```javascript
// Create a read-only user for the ecommerce database
db.createUser({
  user: "reporting-user",
  pwd: "ReportingPass123!",
  roles: [{ role: "read", db: "ecommerce" }]
})

// Create an application user with read/write on specific database
db.createUser({
  user: "app-user",
  pwd: "AppUserPass456!",
  roles: [{ role: "readWrite", db: "ecommerce" }]
})

// Create an admin user
db.createUser({
  user: "dba-user",
  pwd: "DBAPass789!",
  roles: [{ role: "dbAdmin", db: "ecommerce" }, { role: "readWrite", db: "ecommerce" }]
})
```

### Built-in Roles

| Role | Permissions |
|---|---|
| `read` | Read all collections in the database |
| `readWrite` | Read and write all collections |
| `dbAdmin` | Create/drop indexes, view stats, no data access |
| `dbOwner` | Combines `readWrite` + `dbAdmin` |
| `userAdmin` | Create and manage users |
| `clusterAdmin` | Manage the cluster (admin database) |

---

## 5. Security — Encryption

### Encryption at Rest

DocumentDB encryption at rest uses KMS and must be enabled at cluster creation time. It cannot be enabled on an existing unencrypted cluster.

```bash
# Create an encrypted cluster
aws docdb create-db-cluster \
  --db-cluster-identifier my-encrypted-cluster \
  --engine docdb \
  --master-username admin \
  --master-user-password "Password123!" \
  --storage-encrypted \
  --kms-key-id alias/my-docdb-key \
  --db-subnet-group-name docdb-subnet-group

# To encrypt an existing cluster: take a snapshot, encrypt the snapshot copy, restore
aws docdb copy-db-cluster-snapshot \
  --source-db-cluster-snapshot-identifier unencrypted-snapshot \
  --target-db-cluster-snapshot-identifier encrypted-snapshot \
  --kms-key-id alias/my-docdb-key

aws docdb restore-db-cluster-from-snapshot \
  --db-cluster-identifier encrypted-cluster \
  --snapshot-identifier encrypted-snapshot \
  --engine docdb
```

### Encryption in Transit (TLS)

DocumentDB requires TLS by default. Connections without TLS are rejected. The `tls` cluster parameter controls this:

```bash
# Check TLS setting (default: enabled)
aws docdb describe-db-clusters \
  --db-cluster-identifier my-docdb-cluster \
  --query "DBClusters[0].DBClusterParameterGroup"

# Download the CA certificate bundle
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

TLS can be disabled by setting the `tls` parameter to `disabled` in the cluster parameter group, but this is strongly discouraged. Keep TLS enabled in all environments.

---

## 6. IAM Authentication

DocumentDB supports IAM database authentication as an alternative to password-based authentication. IAM auth generates a short-lived authentication token valid for **15 minutes** using the `generate-db-auth-token` API.

```bash
# Generate an authentication token
aws docdb generate-db-auth-token \
  --hostname mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com \
  --port 27017 \
  --username iam-user
```

```python
import boto3
import pymongo

def get_docdb_token(hostname, port, username, region):
    client = boto3.client('rds', region_name=region)
    token = client.generate_db_auth_token(
        DBHostname=hostname,
        Port=port,
        DBUsername=username
    )
    return token

hostname = 'mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com'
token = get_docdb_token(hostname, 27017, 'iam-user', 'us-east-1')

connection = pymongo.MongoClient(
    host=hostname,
    port=27017,
    username='iam-user',
    password=token,
    authSource='$external',
    authMechanism='MONGODB-AWS',
    tls=True,
    tlsCAFile='rds-combined-ca-bundle.pem',
    retryWrites=False
)
```

IAM authentication requires an IAM policy granting the `rds-db:connect` action. The benefit: no long-lived passwords stored in application code or secrets managers (though AWS Secrets Manager with automatic rotation is equally viable and simpler to implement).

---

## 7. Audit Logging

DocumentDB can log all authentication and authorization events, as well as DDL and DML operations, to **CloudWatch Logs**.

```bash
# Enable audit logging by updating the cluster parameter group
aws docdb create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name docdb-audit-params \
  --db-parameter-group-family docdb4.0 \
  --description "Enable audit logging"

aws docdb modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name docdb-audit-params \
  --parameters ParameterName=audit_logs,ParameterValue=enabled,ApplyMethod=immediate

# Apply to cluster
aws docdb modify-db-cluster \
  --db-cluster-identifier my-docdb-cluster \
  --db-cluster-parameter-group-name docdb-audit-params \
  --enable-cloudwatch-logs-exports audit \
  --apply-immediately

# View audit logs in CloudWatch
aws logs describe-log-groups --log-group-name-prefix /aws/docdb/my-docdb-cluster
```

Log entries contain: `atype` (authentication, createCollection, dropCollection, find, insert, etc.), timestamp, user, client IP, and query details.

---

## 8. DocumentDB Elastic Clusters

DocumentDB Elastic Clusters is a newer deployment model where DocumentDB manages sharding automatically. Unlike the standard cluster model where you manage instance count and size, Elastic Clusters scale compute and storage independently based on workload.

### Key Differences from Standard Clusters

| Aspect | Standard Cluster | Elastic Cluster |
|---|---|---|
| Sharding | Single shard (no sharding) | Automatic sharding |
| Scale model | Vertical (change instance type) + read replicas | Horizontal (adds shards automatically) |
| Max storage | 128 TiB | No documented limit |
| Shard count | 1 | 2–32 shards |
| Replication | Primary + up to 15 replicas | Built-in per shard |
| Use case | Up to ~128 TiB, moderate writes | Multi-TB, high write throughput |
| MongoDB compatibility | MongoDB 4.0 | MongoDB 5.0 |

### Creating an Elastic Cluster

```bash
aws docdb-elastic create-cluster \
  --cluster-name my-elastic-cluster \
  --admin-user-name admin \
  --admin-user-password "Password123!" \
  --auth-type PLAIN_TEXT \
  --shard-capacity 2 \    # vCPUs per shard
  --shard-count 4         # Number of shards
```

Elastic Clusters are suitable for workloads that exceed what a single large instance can handle or require horizontal write scaling. For most migrations from MongoDB, standard clusters are sufficient.

---

## 9. Migration to DocumentDB

### Using AWS DMS

AWS Database Migration Service can migrate data from a self-managed MongoDB instance to DocumentDB. Refer to the `Database Migration Service/` section for the full DMS walkthrough. Key DMS settings for DocumentDB:

```json
{
  "NestingLevel": "one",
  "ExtractDocId": "true",
  "DocsToInvestigate": "1000"
}
```

`NestingLevel=one` flattens the BSON document one level, which is required for DMS to understand the document structure. Without it, DMS treats the entire document as a blob.

### Using mongodump / mongorestore

For small to medium migrations (< 100 GB):

```bash
# Dump from source MongoDB
mongodump \
  --host source-mongodb:27017 \
  --db myapp \
  --out /backup/myapp-dump \
  --ssl

# Restore to DocumentDB
mongorestore \
  --host mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com:27017 \
  --db myapp \
  --dir /backup/myapp-dump/myapp \
  --ssl --sslCAFile rds-combined-ca-bundle.pem \
  --username admin --password password
```

For live migrations with minimal downtime, combine mongodump (initial load) with change streams or oplog tailing to replay changes that occurred during the migration.

### Pre-Migration Checklist

| Check | Action |
|---|---|
| MongoDB version | DocumentDB is compatible with MongoDB 4.0. Review compatibility for apps using 4.2+ features. |
| Unsupported operators | Test aggregation pipelines — not all stages are supported. |
| `retryWrites` | Set to `false` in connection strings. |
| Index count | DocumentDB supports up to 64 indexes per collection. |
| Document size | Max document size is 16 MB (same as MongoDB). |
| Driver version | Use a MongoDB 4.0-compatible driver version. |
| `$graphLookup` | Replace with Neptune or restructure if used. |
| Full-text search | Replace with OpenSearch if `$text` search is used heavily. |

---

## Key Takeaways

- Multi-document transactions are supported and use snapshot isolation. Keep transactions short (under 60 seconds) to avoid automatic abort.
- Change Streams provide a pull-based real-time change feed with a configurable retention window (3h–7d). Store the resume token to resume after failures.
- Backups (PITR) are always active. Restoration creates a new cluster — never overwrites the source. Always add instances after restoring.
- Encryption at rest and TLS in transit are both available. Encryption at rest must be enabled at creation time; TLS is on by default.
- Audit logging via CloudWatch Logs is available for compliance. Enable it on production clusters.
- For migrations, use DMS for large or ongoing migrations. Use mongodump/mongorestore for one-time migrations of smaller datasets.
