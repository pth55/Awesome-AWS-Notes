# Part 5 — DynamoDB Streams, Global Tables, DAX, Transactions & TTL

## Table of Contents

1. [DynamoDB Streams](#1-dynamodb-streams)
2. [Lambda Integration with Streams](#2-lambda-integration-with-streams)
3. [Global Tables](#3-global-tables)
4. [DAX — DynamoDB Accelerator](#4-dax--dynamodb-accelerator)
5. [Transactions](#5-transactions)
6. [TTL — Time to Live](#6-ttl--time-to-live)
7. [Feature Combination Patterns](#7-feature-combination-patterns)
8. [Cost Reference](#8-cost-reference)

---

## 1. DynamoDB Streams

DynamoDB Streams is an ordered, time-stamped log of every item-level change in a table. Each change appears as a stream record. Records are available for **24 hours** after they are written, then automatically deleted.

### Stream Record Types

When you enable Streams, you choose a **StreamViewType** that controls what data each record contains:

| StreamViewType | What It Contains |
|---|---|
| `KEYS_ONLY` | Only the partition key and sort key of the modified item |
| `NEW_IMAGE` | The entire item as it appears after the change |
| `OLD_IMAGE` | The entire item as it appeared before the change |
| `NEW_AND_OLD_IMAGES` | Both the before and after states of the item |

`NEW_AND_OLD_IMAGES` is the most common choice for audit trails and change detection. `KEYS_ONLY` is cheapest when you only need to know which item changed.

### Internal Structure

A Stream is divided into **shards**. Each shard contains a sequence of stream records for a subset of the table's partitions. Shards are created and retired automatically as the table scales; you never manage them directly. The shard count is roughly proportional to the number of table partitions.

Each stream record contains:
- `eventID` — unique identifier
- `eventName` — `INSERT`, `MODIFY`, or `REMOVE`
- `eventVersion` — always `1.1`
- `awsRegion` — region the change occurred in
- `dynamodb` block — the actual data (keys, old image, new image, sequence number, timestamp)

### Enabling Streams

```bash
# Enable Streams on a new table
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions AttributeName=OrderId,AttributeType=S \
  --key-schema AttributeName=OrderId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# Enable Streams on an existing table
aws dynamodb update-table \
  --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# Get the stream ARN
aws dynamodb describe-table --table-name Orders \
  --query "Table.LatestStreamArn" --output text
```

### Reading from Streams Directly

```python
import boto3

dynamo_streams = boto3.client('dynamodbstreams')

# List shards for the stream
response = dynamo_streams.describe_stream(
    StreamArn='arn:aws:dynamodb:us-east-1:123456789012:table/Orders/stream/...'
)
shards = response['StreamDescription']['Shards']

for shard in shards:
    shard_id = shard['ShardId']
    # Get shard iterator
    iterator_response = dynamo_streams.get_shard_iterator(
        StreamArn='arn:aws:dynamodb:...',
        ShardId=shard_id,
        ShardIteratorType='TRIM_HORIZON'  # Start from oldest available record
    )
    shard_iterator = iterator_response['ShardIterator']

    # Read records
    records_response = dynamo_streams.get_records(ShardIterator=shard_iterator)
    for record in records_response['Records']:
        event_name = record['eventName']  # INSERT, MODIFY, REMOVE
        new_image = record['dynamodb'].get('NewImage', {})
        old_image = record['dynamodb'].get('OldImage', {})
        print(f"{event_name}: {new_image}")
```

Direct shard reading is rarely used in production. Lambda event source mapping is the standard approach.

---

## 2. Lambda Integration with Streams

Lambda is the primary consumer of DynamoDB Streams. The service polls the stream shards and invokes your function in batches.

### Event Source Mapping Configuration

```bash
aws lambda create-event-source-mapping \
  --function-name ProcessOrderChanges \
  --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders/stream/... \
  --batch-size 100 \
  --starting-position LATEST \
  --filter-criteria '{"Filters": [{"Pattern": "{\"eventName\": [\"INSERT\"]}"}]}'
```

Key parameters:
- `--batch-size`: 1–10,000 records per invocation (default 100). Higher values improve throughput but increase retry blast radius.
- `--starting-position`: `LATEST` (new records only) or `TRIM_HORIZON` (all available records, up to 24h back).
- `--filter-criteria`: Filter at the Lambda service level to avoid invoking the function for events you don't care about.
- `--bisect-batch-on-function-error`: When enabled, on error Lambda splits the batch in half and retries each half, helping isolate poison-pill records.
- `--destination-config`: Send failed batches to SQS or SNS for dead-letter handling.

### Lambda Handler Pattern

```python
import boto3
from boto3.dynamodb.types import TypeDeserializer

deserializer = TypeDeserializer()

def lambda_handler(event, context):
    for record in event['Records']:
        event_name = record['eventName']  # INSERT | MODIFY | REMOVE
        
        # DynamoDB stream images use DynamoDB typed format: {"S": "value"}
        # TypeDeserializer converts to plain Python types
        if event_name == 'INSERT':
            new_item = {k: deserializer.deserialize(v)
                        for k, v in record['dynamodb']['NewImage'].items()}
            handle_new_order(new_item)

        elif event_name == 'MODIFY':
            old_item = {k: deserializer.deserialize(v)
                        for k, v in record['dynamodb']['OldImage'].items()}
            new_item = {k: deserializer.deserialize(v)
                        for k, v in record['dynamodb']['NewImage'].items()}
            handle_order_update(old_item, new_item)

        elif event_name == 'REMOVE':
            old_item = {k: deserializer.deserialize(v)
                        for k, v in record['dynamodb']['OldImage'].items()}
            handle_order_deletion(old_item)

def handle_new_order(item):
    # E.g., publish to EventBridge, notify downstream systems
    pass
```

### Common Stream + Lambda Patterns

| Use Case | StreamViewType | What Lambda Does |
|---|---|---|
| Audit trail | `NEW_AND_OLD_IMAGES` | Write change record to S3 or another DynamoDB table |
| Cross-service event | `NEW_IMAGE` | Publish to EventBridge or SNS |
| Search index sync | `NEW_AND_OLD_IMAGES` | PUT/DELETE document in OpenSearch |
| Aggregation/counters | `KEYS_ONLY` | Increment a counter in another table |
| Cache invalidation | `KEYS_ONLY` | Delete item from ElastiCache on change |
| Replication to Redshift | `NEW_IMAGE` | Kinesis Firehose → S3 → Redshift COPY |

---

## 3. Global Tables

Global Tables provide **active-active multi-region replication**. Every replica table accepts both reads and writes. Changes propagate to all other replicas within typically **under 1 second**.

### How Replication Works

1. You designate a table as a Global Table and add replica regions.
2. DynamoDB uses Streams internally to replicate changes. The stream is not the same as a user-visible stream — it is a system-level replication channel.
3. Each replica is a full, independent copy of the data in that region. It can serve reads and writes independently even if other regions become unreachable.
4. Writes in one region propagate to all other regions asynchronously.

### Conflict Resolution

Two clients in different regions can write the same item simultaneously. DynamoDB resolves this with **last-writer-wins** using the write timestamp (wall clock). The write with the most recent timestamp wins. There is no application-level conflict resolution hook.

Design implication: avoid concurrent writes to the same item from multiple regions when ordering matters. Use single-region routing for those items, or design keys so the same item is only ever written from one region.

### Requirements for Global Tables

- Table must use **PAY_PER_REQUEST** billing, or **provisioned with Auto Scaling enabled** in all regions.
- All replica tables must have identical primary key schema, LSIs, and TTL attribute (if used).
- DynamoDB Streams must be enabled (the system does this automatically when you create a Global Table).
- The table must not have existing data before adding replicas (for new tables). Existing tables can be converted using the console.

### Creating Global Tables

```bash
# Create a new table in us-east-1 (first region)
aws dynamodb create-table \
  --table-name UserSessions \
  --attribute-definitions AttributeName=UserId,AttributeType=S \
  --key-schema AttributeName=UserId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Add replica in eu-west-1
aws dynamodb update-table \
  --table-name UserSessions \
  --replica-updates '[{"Create": {"RegionName": "eu-west-1"}}]' \
  --region us-east-1

# Add replica in ap-southeast-1
aws dynamodb update-table \
  --table-name UserSessions \
  --replica-updates '[{"Create": {"RegionName": "ap-southeast-1"}}]' \
  --region us-east-1

# Check replica status
aws dynamodb describe-table --table-name UserSessions --region us-east-1 \
  --query "Table.Replicas"
```

### Reading from the Nearest Region

Use the SDK's regional endpoint to route reads to the nearest replica:

```python
import boto3

# Route to nearest region automatically using latency-based routing
# in Route 53, or configure explicitly per request
dynamodb = boto3.resource('dynamodb', region_name='eu-west-1')
table = dynamodb.Table('UserSessions')

# Strongly consistent reads only serve from the local replica in Global Tables
# You cannot read from another region with strong consistency
response = table.get_item(
    Key={'UserId': 'user-123'},
    ConsistentRead=True  # Consistent within eu-west-1 replica only
)
```

### Global Tables Pricing

Global Tables adds a replicated write request unit (rWRU) charge for each region beyond the source:

- Every write to a Global Table creates **rWRUs** in each destination replica region.
- Cost: approximately **$1.875 per million rWRUs** (vs $1.25 per million standard WRUs).
- Storage is charged separately in each region.

---

## 4. DAX — DynamoDB Accelerator

DAX is a fully managed, in-memory cache purpose-built for DynamoDB. It is a write-through cache that sits between your application and DynamoDB. The DAX client intercepts DynamoDB API calls and routes them through the cache.

### What DAX Caches

| Cache Type | What Is Stored | TTL Control |
|---|---|---|
| **Item cache** | Results of `GetItem` and `BatchGetItem` calls | `ItemTTL` (default 5 min) |
| **Query cache** | Results of `Query` and `Scan` calls | `QueryTTL` (default 5 min) |

`PutItem`, `UpdateItem`, `DeleteItem`, and `TransactWriteItems` are written through to DynamoDB first, then the item cache is updated. Query cache entries are invalidated on writes that affect the same key.

### DAX Architecture

A DAX cluster consists of a **primary node** (handles writes) and up to **9 read replicas** (serve reads). Nodes are distributed across Availability Zones. The DAX client connects to the cluster endpoint and handles routing internally.

```
Application → DAX Cluster Endpoint
                 ├── Primary Node (writes + reads)   →  DynamoDB
                 ├── Read Replica (AZ-b)
                 └── Read Replica (AZ-c)
```

### When to Use DAX

Use DAX when:
- Read latency must be in **microseconds** (DAX delivers sub-millisecond vs DynamoDB's single-digit millisecond).
- You have **hot keys** — the same small set of items read by many requests simultaneously.
- Your application already uses `GetItem`, `BatchGetItem`, `Query`, or `Scan` — the DAX client is a drop-in replacement.

Do NOT use DAX when:
- Your workload is **write-heavy** — DAX only caches reads.
- You need **strongly consistent reads** — DAX item cache serves eventually consistent reads only. Pass-through to DynamoDB for consistent reads.
- Your application already uses ElastiCache for a more flexible caching strategy.
- You need to cache **aggregated or computed results** — DAX only caches raw DynamoDB API responses.
- You need **fine-grained cache control** per key — DAX uses a single TTL for all items.

### Creating a DAX Cluster

```bash
# Create subnet group for DAX
aws dax create-subnet-group \
  --subnet-group-name dax-subnet-group \
  --subnet-ids subnet-aaa111 subnet-bbb222 subnet-ccc333

# Create parameter group (optional, use defaults initially)
aws dax create-parameter-group \
  --parameter-group-name dax-params

# Create the cluster
aws dax create-cluster \
  --cluster-name orders-dax \
  --node-type dax.r6g.large \
  --replication-factor 3 \
  --iam-role-arn arn:aws:iam::123456789012:role/DAXRole \
  --subnet-group-name dax-subnet-group \
  --parameter-group-name dax-params \
  --sse-specification Enabled=true

# Check cluster status
aws dax describe-clusters --cluster-names orders-dax \
  --query "Clusters[0].Status"
```

### Connecting via DAX Client

```python
import amazon.ion.simpleion as ion
import amazondax
import boto3

# The DAX client is a drop-in replacement for the DynamoDB client
dax = amazondax.AmazonDaxClient.resource(
    endpoint_url='daxs://orders-dax.xxxxxx.dax-clusters.us-east-1.amazonaws.com',
    region_name='us-east-1'
)

table = dax.Table('Orders')

# This GetItem is served from DAX cache if available, DynamoDB otherwise
response = table.get_item(Key={'OrderId': 'ORD-001'})

# For strongly consistent reads, bypass DAX entirely using standard DynamoDB client
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
consistent_table = dynamodb.Table('Orders')
consistent_response = consistent_table.get_item(
    Key={'OrderId': 'ORD-001'},
    ConsistentRead=True
)
```

### DAX Node Types and Pricing

| Node Type | Memory | vCPU | Requests/sec (approx) |
|---|---|---|---|
| `dax.t3.small` | 2 GB | 2 | ~1,000 |
| `dax.r6g.large` | 16 GB | 2 | ~50,000 |
| `dax.r6g.xlarge` | 32 GB | 4 | ~100,000 |
| `dax.r6g.4xlarge` | 128 GB | 16 | ~400,000+ |

Pricing: **~$0.269/hr** for `dax.r6g.large` (us-east-1). A 3-node cluster costs ~$0.81/hr (~$590/month). DAX is charged only for cluster running time — no per-request fees.

---

## 5. Transactions

DynamoDB Transactions allow up to **100 items** across multiple tables to be read or written atomically in a single request. All items succeed together or all fail together — true ACID semantics.

### Two Transaction APIs

| API | Direction | Idempotency |
|---|---|---|
| `TransactWriteItems` | Write (Put, Update, Delete, ConditionCheck) | Yes (via `ClientRequestToken`) |
| `TransactGetItems` | Read (Get) | N/A — reads are always consistent snapshots |

### TransactWriteItems

```python
import boto3

dynamodb = boto3.client('dynamodb')

# Atomic transfer: debit one account, credit another
response = dynamodb.transact_write_items(
    TransactItems=[
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'AccountId': {'S': 'ACC-001'}},
                'UpdateExpression': 'SET Balance = Balance - :amount',
                'ConditionExpression': 'Balance >= :amount',
                'ExpressionAttributeValues': {
                    ':amount': {'N': '100'}
                }
            }
        },
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'AccountId': {'S': 'ACC-002'}},
                'UpdateExpression': 'SET Balance = Balance + :amount',
                'ExpressionAttributeValues': {
                    ':amount': {'N': '100'}
                }
            }
        },
        {
            'Put': {
                'TableName': 'TransactionLog',
                'Item': {
                    'TxnId': {'S': 'TXN-20241201-001'},
                    'FromAccount': {'S': 'ACC-001'},
                    'ToAccount': {'S': 'ACC-002'},
                    'Amount': {'N': '100'},
                    'Timestamp': {'S': '2024-12-01T10:00:00Z'}
                },
                'ConditionExpression': 'attribute_not_exists(TxnId)'  # Idempotency guard
            }
        }
    ],
    ClientRequestToken='unique-client-token-for-idempotency'
)
```

The `ConditionExpression` on the debit ensures the account has sufficient funds. If the condition fails, the entire transaction is cancelled and a `TransactionCanceledException` is raised with a `CancellationReasons` list showing which items failed and why.

### ConditionCheck

`ConditionCheck` lets you validate an item without writing to it:

```python
{
    'ConditionCheck': {
        'TableName': 'Orders',
        'Key': {'OrderId': {'S': 'ORD-001'}},
        'ConditionExpression': '#status = :pending',
        'ExpressionAttributeNames': {'#status': 'Status'},
        'ExpressionAttributeValues': {':pending': {'S': 'PENDING'}}
    }
}
```

### TransactGetItems

```python
response = dynamodb.transact_get_items(
    TransactItems=[
        {'Get': {'TableName': 'Accounts', 'Key': {'AccountId': {'S': 'ACC-001'}}}},
        {'Get': {'TableName': 'Accounts', 'Key': {'AccountId': {'S': 'ACC-002'}}}}
    ]
)

items = [r.get('Item') for r in response['Responses']]
```

`TransactGetItems` returns a consistent snapshot of all requested items at a single point in time.

### Transaction Cost

Transactions cost **2× the normal read/write units**:
- A transactional write of a 2 KB item consumes **4 WCUs** (2× the normal 2 WCUs).
- A transactional read of a 4 KB item consumes **4 RCUs** (2× the normal 2 RCUs for strongly consistent reads).

The 2× cost reflects the two-phase commit protocol DynamoDB uses internally to guarantee atomicity.

### Transaction Limits and Constraints

- Maximum **100 items per transaction** across all tables.
- Maximum request size: **4 MB**.
- Items in a transaction cannot span more than **25 unique tables**.
- A transaction cannot include items from Global Tables replicas in different regions simultaneously.
- Concurrent transactions on the same item will conflict. One will succeed; the other raises `TransactionConflictException`. The failed transaction should be retried with exponential backoff.
- Transactions cannot be used inside Lambda functions triggered by DynamoDB Streams (circular dependency risk).

---

## 6. TTL — Time to Live

TTL allows DynamoDB to automatically delete items after a specified expiry time. It is free — deletions from TTL do not consume any WCUs.

### How TTL Works

1. You designate a table attribute as the TTL attribute (any name, must store a **Unix epoch timestamp in seconds**).
2. DynamoDB runs a background scanner that identifies items where the TTL attribute value is in the past.
3. Expired items are typically deleted within **48 hours** of the expiry time (not instantaneous).
4. TTL deletions appear in DynamoDB Streams as `REMOVE` events with `userIdentity.type = "Service"` and `userIdentity.principalId = "dynamodb.amazonaws.com"`.

Items whose TTL timestamp is in the past but have not yet been deleted will still be returned by reads and queries. Filter them in application code if you need strict expiry behavior:

```python
import time

response = table.query(
    KeyConditionExpression=Key('UserId').eq('user-123'),
    FilterExpression=Attr('ExpiresAt').gt(int(time.time()))
)
```

### Enabling TTL

```bash
# Enable TTL on the table using attribute "ExpiresAt"
aws dynamodb update-time-to-live \
  --table-name UserSessions \
  --time-to-live-specification Enabled=true,AttributeName=ExpiresAt

# Verify
aws dynamodb describe-time-to-live --table-name UserSessions
```

### Writing Items with TTL

```python
import time

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('UserSessions')

SESSION_DURATION_SECONDS = 30 * 60  # 30 minutes

table.put_item(Item={
    'UserId': 'user-123',
    'SessionToken': 'tok-abc',
    'CreatedAt': int(time.time()),
    'ExpiresAt': int(time.time()) + SESSION_DURATION_SECONDS,  # TTL attribute
    'Data': {'theme': 'dark', 'locale': 'en-US'}
})
```

### TTL + Streams: Archival Pattern

Use TTL deletions captured in Streams to archive data before it disappears:

```
DynamoDB Table
    │  (item expires)
    ▼
DynamoDB Stream (REMOVE event, userIdentity = dynamodb service)
    │
    ▼
Lambda Function
    │
    ├──► S3 (long-term archive)
    └──► Redshift / Athena (analytics)
```

Lambda filter to catch only TTL deletions (not manual deletes):

```python
def lambda_handler(event, context):
    for record in event['Records']:
        if record['eventName'] != 'REMOVE':
            continue
        # Check if this was a TTL deletion (not a manual delete)
        user_identity = record.get('userIdentity', {})
        if user_identity.get('type') == 'Service' and \
           'dynamodb.amazonaws.com' in user_identity.get('principalId', ''):
            # Archive the expired item
            archive_item(record['dynamodb']['OldImage'])
```

---

## 7. Feature Combination Patterns

| Pattern | Features Used | Use Case |
|---|---|---|
| Real-time search sync | Streams + Lambda + OpenSearch | Auto-index DynamoDB changes into OpenSearch for full-text search |
| Global session store | Global Tables + TTL | User sessions replicated to nearest region, auto-deleted on expiry |
| Microsecond cache | DAX + Streams + Lambda | DAX for reads, Streams + Lambda to invalidate DAX cache on writes from other services |
| Idempotent order processing | Transactions + ConditionCheck | Debit inventory, create order, log event — all atomic, no duplicate processing |
| Compliance audit | Streams + `NEW_AND_OLD_IMAGES` + S3 | Every item change archived to immutable S3 bucket |
| Tiered storage | TTL + Streams + Lambda + S3 IA | Keep 90 days hot in DynamoDB, archive to S3 Intelligent-Tiering on expiry |

---

## 8. Cost Reference

| Feature | Cost |
|---|---|
| DynamoDB Streams reads | Free (first 2.5 million stream read request units/month free, then $0.02/100K) |
| Global Tables (rWRU) | ~$1.875 per million replicated write request units |
| Global Tables storage | Standard table storage rate per region |
| DAX node (`dax.r6g.large`) | ~$0.269/hr per node |
| Transactions (write) | 2× normal WCU cost |
| Transactions (read) | 2× normal strongly consistent RCU cost |
| TTL deletions | Free |
| Lambda invocations (Streams) | Standard Lambda pricing (~$0.20/million requests) |

---

## Key Takeaways

- **Streams** provide a 24-hour change log; Lambda is the standard consumer. Always configure a DLQ (`--destination-config`) for failure handling.
- **Global Tables** use last-writer-wins conflict resolution. Design to avoid concurrent writes to the same item from multiple regions.
- **DAX** is for microsecond read latency on hot keys. It only helps read-heavy workloads; strongly consistent reads bypass the cache.
- **Transactions** are 2× the cost of non-transactional operations. Use them only when atomicity is genuinely required.
- **TTL** deletions are free but can be delayed up to 48 hours. Always filter in application code when strict expiry is needed. Capture TTL deletions via Streams for archival.
