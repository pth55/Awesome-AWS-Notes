# Part 3: Capacity, Pricing & Auto Scaling

---

## Table of Contents

1. [Read and Write Capacity Units Explained](#1-read-and-write-capacity-units-explained)
2. [How DynamoDB Calculates RCU and WCU](#2-how-dynamodb-calculates-rcu-and-wcu)
3. [Burst Capacity and Adaptive Capacity](#3-burst-capacity-and-adaptive-capacity)
4. [Provisioned Capacity: Sizing and Management](#4-provisioned-capacity-sizing-and-management)
5. [Auto Scaling Configuration](#5-auto-scaling-configuration)
6. [On-Demand Mode: Behavior and Limits](#6-on-demand-mode-behavior-and-limits)
7. [CloudWatch Metrics for Capacity Monitoring](#7-cloudwatch-metrics-for-capacity-monitoring)
8. [Capacity Alarm Setup](#8-capacity-alarm-setup)
9. [Reserved Capacity](#9-reserved-capacity)
10. [Cost Optimization Strategies](#10-cost-optimization-strategies)
11. [Pricing Reference](#11-pricing-reference)

---

## 1. Read and Write Capacity Units Explained

DynamoDB measures throughput in capacity units. Understanding these is required for capacity planning and cost management.

### Read Capacity Unit (RCU)

**1 RCU = 1 strongly consistent read of an item up to 4 KB per second**

- **Strongly consistent read:** Reads from the primary node. Returns the latest committed data. Costs 1 RCU per 4 KB.
- **Eventually consistent read:** Reads from a replica node. May return data up to a few hundred milliseconds old. Costs **0.5 RCU** per 4 KB (half the cost).
- **Transactional read:** Used with `TransactGetItems`. Costs **2 RCU** per 4 KB.

Items larger than 4 KB round up to the next 4 KB boundary:

| Item Size | Strongly Consistent | Eventually Consistent |
|---|---|---|
| 1 KB | 1 RCU | 0.5 RCU |
| 4 KB | 1 RCU | 0.5 RCU |
| 4.1 KB | 2 RCU | 1 RCU |
| 8 KB | 2 RCU | 1 RCU |
| 10 KB | 3 RCU | 1.5 RCU |

### Write Capacity Unit (WCU)

**1 WCU = 1 write of an item up to 1 KB per second**

- **Standard write** (`PutItem`, `UpdateItem`, `DeleteItem`): 1 WCU per 1 KB
- **Transactional write** (`TransactWriteItems`): **2 WCU** per 1 KB
- Writes larger than 1 KB round up to the next 1 KB boundary

| Item Size | Standard Write | Transactional Write |
|---|---|---|
| 0.5 KB | 1 WCU | 2 WCU |
| 1 KB | 1 WCU | 2 WCU |
| 1.1 KB | 2 WCU | 4 WCU |
| 5 KB | 5 WCU | 10 WCU |

### GSI Capacity

Each Global Secondary Index has its own RCU and WCU allocation (for provisioned mode). Every write to the base table that affects a GSI attribute triggers a write to the GSI as well.

If your table has 3 GSIs and every write touches all 3, effective WCU consumption per write is:
```
base table WCU + GSI1 WCU + GSI2 WCU + GSI3 WCU = 4× base WCU per write
```

---

## 2. How DynamoDB Calculates RCU and WCU

### Read Operations and RCU Consumption

| Operation | Consistency | RCU Formula |
|---|---|---|
| `GetItem` | Strongly consistent (opt-in) | `ceil(item_size / 4 KB)` |
| `GetItem` | Eventually consistent (default) | `ceil(item_size / 4 KB) × 0.5` |
| `Query` | Eventually consistent (default) | Sum of `ceil(size / 4 KB) × 0.5` for each returned item |
| `Query` | Strongly consistent (opt-in) | Sum of `ceil(size / 4 KB)` for each returned item |
| `Scan` | Eventually consistent (default) | Sum of `ceil(size / 4 KB) × 0.5` for every item **examined** (not just returned) |
| `BatchGetItem` | Per item, same as `GetItem` | Sum across all items |
| `TransactGetItems` | Strongly consistent always | `ceil(item_size / 4 KB) × 2` |

**Key point for Scans:** RCU is charged for every item examined by the scan, not just the items returned after `FilterExpression` is applied. A scan that examines 10,000 items but returns 50 still costs 10,000 items worth of RCU. This is why scans are expensive at scale.

### Write Operations and WCU Consumption

| Operation | WCU Formula |
|---|---|
| `PutItem` | `ceil(item_size / 1 KB)` |
| `UpdateItem` | `ceil(max(old_item_size, new_item_size) / 1 KB)` (charged for the larger of before/after) |
| `DeleteItem` | `ceil(item_size / 1 KB)` (item size before deletion) |
| `BatchWriteItem` | Sum of individual write WCU |
| `TransactWriteItems` | `ceil(item_size / 1 KB) × 2` |

**Conditional writes:** If a condition expression causes the write to fail, DynamoDB still charges 1 WCU for the conditional check.

---

## 3. Burst Capacity and Adaptive Capacity

### Burst Capacity

DynamoDB reserves unused capacity for burst absorption. Each table retains up to **5 minutes of unused capacity** as a burst pool.

If your table is provisioned at 100 WCU and averages 60 WCU, the unused 40 WCU accumulates in the burst pool. A sudden spike can use this burst pool to temporarily exceed the provisioned limit for a short period.

**Important limitations:**
- Burst capacity is best-effort — AWS does not guarantee it will always be available
- It is not an alternative to proper capacity planning
- The burst pool is per-partition, not per-table

### Adaptive Capacity (Automatic)

DynamoDB has a feature called **Adaptive Capacity** that automatically redistributes uneven traffic across partitions. If one partition is receiving more traffic than its share of the provisioned capacity while other partitions are idle, DynamoDB will shift more capacity to the hot partition.

Adaptive Capacity is enabled automatically for all tables with no configuration required. It handles gradual and moderate hot key scenarios, but it will not save you from an extreme hot partition (e.g., 100% of all traffic to a single key).

---

## 4. Provisioned Capacity: Sizing and Management

### Sizing Based on Traffic Analysis

To size provisioned capacity, analyze your expected access patterns:

```
Required RCU = (reads_per_second × average_item_size_KB) / 4 KB
Required WCU = writes_per_second × ceil(average_item_size_KB / 1 KB)

Example:
  500 reads/sec, average item 2 KB, eventually consistent:
  → RCU = (500 × 2) / 4 × 0.5 = 125 RCU

  100 writes/sec, average item 0.5 KB:
  → WCU = 100 × 1 = 100 WCU
```

Apply a buffer of 20–30% for variance: provision 150–160 RCU and 120–130 WCU in this example.

### Updating Provisioned Capacity

```bash
# Increase or decrease provisioned capacity
aws dynamodb update-table \
  --table-name Orders \
  --provisioned-throughput ReadCapacityUnits=200,WriteCapacityUnits=100 \
  --region us-east-1

# Note: Decreases are limited — you can only decrease capacity
# up to 4 times per calendar day (UTC)
```

**Important constraint:** Provisioned capacity can only be **decreased 4 times per calendar day**. Increases are unlimited. Plan decreases carefully — if you reduce too aggressively and need to reduce again within the same day, you may not be able to.

### Checking Current Capacity

```bash
aws dynamodb describe-table --table-name Orders \
  --query "Table.{
    TableStatus: TableStatus,
    ProvisionedRCU: ProvisionedThroughput.ReadCapacityUnits,
    ProvisionedWCU: ProvisionedThroughput.WriteCapacityUnits,
    ConsumedRCU: ConsumedThroughput.ReadCapacityUnits,
    ConsumedWCU: ConsumedThroughput.WriteCapacityUnits
  }" \
  --region us-east-1
```

---

## 5. Auto Scaling Configuration

DynamoDB Auto Scaling uses AWS Application Auto Scaling to automatically adjust provisioned capacity based on actual usage. It targets a utilization percentage and scales up or down to maintain it.

### How Auto Scaling Works

1. A CloudWatch alarm monitors `ConsumedReadCapacityUnits` or `ConsumedWriteCapacityUnits`
2. When utilization exceeds the target (e.g., 70%), Auto Scaling increases capacity
3. When utilization drops below a scale-in threshold, it decreases capacity
4. Scale-out happens within 1–3 minutes. Scale-in has a default cooldown of 5 minutes.

**Limitation:** Auto Scaling cannot respond faster than the time it takes CloudWatch to register sustained demand (~1 minute). A traffic spike that lasts less than 1 minute may cause throttling before Auto Scaling responds.

### Configuring Auto Scaling via CLI

```bash
# Register the table's RCU as a scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --min-capacity 10 \
  --max-capacity 1000 \
  --region us-east-1

# Register WCU as a scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 5 \
  --max-capacity 500 \
  --region us-east-1

# Set target tracking policy for reads (target: 70% utilization)
aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --policy-name "ReadAutoScaling" \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "DynamoDBReadCapacityUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }' \
  --region us-east-1

# Auto scaling for GSI (same pattern, different resource-id)
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders/index/Status-CreatedAt-index" \
  --scalable-dimension "dynamodb:index:ReadCapacityUnits" \
  --min-capacity 5 \
  --max-capacity 200 \
  --region us-east-1
```

### Auto Scaling via Console

```
AWS Console → DynamoDB → Tables → Orders → Additional settings tab
→ Read/Write capacity → Auto Scaling
→ Set min/max capacity and target utilization
→ Apply same settings to GSIs
```

---

## 6. On-Demand Mode: Behavior and Limits

### Traffic Scaling Behavior

On-Demand mode scales to accommodate traffic automatically with no capacity planning. However, there are practical limits:

- **Previous peak doubling:** DynamoDB allows instant traffic to double the previous peak throughput for the table. If your table's peak in the last 30 days was 10,000 WRU/second, it can handle 20,000 WRU/second without throttling.
- **New tables:** Newly created on-demand tables start with a default limit. Tables with no traffic history have a limit of 4,000 WCU and 12,000 RCU. Requests exceeding these limits will be throttled.

```bash
# Check on-demand table limits
aws dynamodb describe-table --table-name NewTable \
  --query "Table.BillingModeSummary" \
  --region us-east-1
```

### Switching Between Modes

```bash
# Switch from provisioned to on-demand
aws dynamodb update-table \
  --table-name Orders \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Switch from on-demand to provisioned
aws dynamodb update-table \
  --table-name Orders \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50 \
  --region us-east-1
```

**Rule:** You can switch modes once every 24 hours per table.

---

## 7. CloudWatch Metrics for Capacity Monitoring

Monitor these metrics in CloudWatch to understand table performance:

| Metric | Namespace | What it tells you |
|---|---|---|
| `ConsumedReadCapacityUnits` | `AWS/DynamoDB` | Actual RCU used per minute |
| `ConsumedWriteCapacityUnits` | `AWS/DynamoDB` | Actual WCU used per minute |
| `ProvisionedReadCapacityUnits` | `AWS/DynamoDB` | Configured RCU (provisioned mode) |
| `ProvisionedWriteCapacityUnits` | `AWS/DynamoDB` | Configured WCU (provisioned mode) |
| `ReadThrottleEvents` | `AWS/DynamoDB` | Number of throttled reads |
| `WriteThrottleEvents` | `AWS/DynamoDB` | Number of throttled writes |
| `SystemErrors` | `AWS/DynamoDB` | Internal DynamoDB errors (usually transient) |
| `SuccessfulRequestLatency` | `AWS/DynamoDB` | p50/p99 latency per operation type |
| `ReturnedItemCount` | `AWS/DynamoDB` | Items returned per Query or Scan |
| `TransactionConflict` | `AWS/DynamoDB` | Conflicting transactions (optimistic lock contention) |

### Querying Metrics via CLI

```bash
# Get read throttle events for last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ReadThrottleEvents \
  --dimensions Name=TableName,Value=Orders \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Sum \
  --region us-east-1

# Get GSI-specific throttle events (specify index dimension)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name WriteThrottleEvents \
  --dimensions \
      Name=TableName,Value=Orders \
      Name=GlobalSecondaryIndexName,Value=Status-CreatedAt-index \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum \
  --region us-east-1
```

---

## 8. Capacity Alarm Setup

These are the critical alarms to configure for any production DynamoDB table.

```bash
# Alarm: table-level read throttling
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-Orders-ReadThrottle" \
  --metric-name ReadThrottleEvents \
  --namespace AWS/DynamoDB \
  --dimensions Name=TableName,Value=Orders \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:oncall-alerts \
  --region us-east-1

# Alarm: GSI write throttling (critical — GSI throttling blocks base table writes)
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-Orders-GSI-WriteThrottle" \
  --metric-name WriteThrottleEvents \
  --namespace AWS/DynamoDB \
  --dimensions \
      Name=TableName,Value=Orders \
      Name=GlobalSecondaryIndexName,Value=Status-CreatedAt-index \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:oncall-alerts \
  --region us-east-1

# Alarm: high read latency (p99 over 50ms indicates a problem)
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-Orders-HighReadLatency" \
  --metric-name SuccessfulRequestLatency \
  --namespace AWS/DynamoDB \
  --dimensions Name=TableName,Value=Orders Name=Operation,Value=GetItem \
  --extended-statistic p99 \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:oncall-alerts \
  --region us-east-1
```

---

## 9. Reserved Capacity

For provisioned tables with stable, predictable long-term workloads, Reserved Capacity provides significant discounts: up to **76% off** standard provisioned pricing.

| Reserved Capacity Term | Upfront Payment Option | Discount |
|---|---|---|
| 1 Year | No upfront | ~57% off |
| 1 Year | All upfront | ~64% off |
| 3 Years | No upfront | ~67% off |
| 3 Years | All upfront | ~76% off |

Reserved Capacity applies automatically to any provisioned DynamoDB usage in your account in the same region. It cannot be applied to on-demand tables.

```
AWS Console → DynamoDB → Reserved Capacity → Purchase Reserved Capacity
```

**When to purchase:** Use Reserved Capacity when:
- You have at least 1 year of traffic history
- Your provisioned capacity has been stable for several months
- The table will continue to exist for the reservation period

---

## 10. Cost Optimization Strategies

### Strategy 1: Use Eventually Consistent Reads Where Possible

The default `GetItem` uses eventually consistent reads. If your application can tolerate a few hundred milliseconds of staleness (most can), do not request strongly consistent reads explicitly.

```python
# Eventually consistent (default) — 0.5 RCU per read
response = table.get_item(Key={'UserID': 'alice'})

# Strongly consistent — 1 RCU per read (2× the cost)
response = table.get_item(
    Key={'UserID': 'alice'},
    ConsistentRead=True
)
```

### Strategy 2: Use ProjectionExpression to Limit Returned Data

`GetItem` and `Query` charge RCU based on the full item size, even if you only need 2 of its 20 attributes. However, using `ProjectionExpression` does NOT reduce RCU cost — it reduces network transfer and application processing, but the read still charges the full item size in RCU.

The exception: if you use `ProjectionExpression` on a `Query` and the projection is served from an index with `KEYS_ONLY` or `INCLUDE` projection, only the index item size is charged.

### Strategy 3: Use BatchGetItem and BatchWriteItem

For multiple independent reads or writes, batch operations reduce the number of API calls and connection overhead. Each item in a batch is still charged individually for RCU/WCU, but fewer API calls means lower latency.

```python
# BatchGetItem — up to 100 items from one or more tables
response = dynamodb.batch_get_item(
    RequestItems={
        'AppTable': {
            'Keys': [
                {'PK': {'S': 'USER#alice'}, 'SK': {'S': 'PROFILE'}},
                {'PK': {'S': 'USER#bob'}, 'SK': {'S': 'PROFILE'}},
                {'PK': {'S': 'USER#carol'}, 'SK': {'S': 'PROFILE'}},
            ]
        }
    }
)
```

### Strategy 4: Use TTL for Automatic Cleanup

Items that expire via TTL are deleted in the background with **no WCU charge**. This is the most cost-effective way to clean up old data.

```bash
# Enable TTL on a table (point to a Unix timestamp attribute)
aws dynamodb update-time-to-live \
  --table-name Sessions \
  --time-to-live-specification Enabled=true,AttributeName=ExpiresAt \
  --region us-east-1
```

```python
import time

# Store session with 30-minute expiry
expiry_time = int(time.time()) + 1800  # 30 minutes from now

table.put_item(Item={
    'SessionID': 'sess-001',
    'SK': 'METADATA',
    'UserID': 'alice',
    'ExpiresAt': expiry_time  # Unix timestamp
})
```

### Strategy 5: Move Infrequently Accessed Data to Standard-IA

For tables with predominantly cold data (infrequently accessed archives), Standard-IA storage class reduces storage cost by 60%.

```bash
aws dynamodb update-table \
  --table-name ArchivedOrders \
  --table-class STANDARD_INFREQUENT_ACCESS \
  --region us-east-1
```

**Trade-off:** Standard-IA charges more per RCU and WCU. It is cost-effective only when the storage savings outweigh the higher throughput cost. AWS recommends Standard-IA when monthly storage cost exceeds throughput cost by at least 50%.

### Strategy 6: Compress Large Attributes

If items contain large string or binary attributes (e.g., JSON payloads, serialized objects), compress them client-side before storing. Smaller item sizes mean fewer RCU/WCU consumed per operation.

```python
import gzip
import json

def compress_attribute(data: dict) -> bytes:
    return gzip.compress(json.dumps(data).encode())

def decompress_attribute(data: bytes) -> dict:
    return json.loads(gzip.decompress(data))

# Store compressed payload
table.put_item(Item={
    'PK': 'ORDER#ord-001',
    'SK': 'PAYLOAD',
    'CompressedData': compress_attribute(large_order_payload)
})
```

---

## 11. Pricing Reference

All prices for `us-east-1` (N. Virginia). Other regions are 5–30% higher.

### Storage

| Storage Class | Price |
|---|---|
| Standard | $0.25 per GB/month |
| Standard-IA | $0.10 per GB/month |

### Provisioned Capacity

| Capacity Type | Price |
|---|---|
| 1 RCU | $0.00013 per hour (~$0.094/month) |
| 1 WCU | $0.00065 per hour (~$0.468/month) |

### On-Demand Capacity

| Operation | Price |
|---|---|
| Read Request Unit | $0.25 per million |
| Write Request Unit | $1.25 per million |
| Transactional Read | $0.50 per million |
| Transactional Write | $2.50 per million |

### Optional Features

| Feature | Price |
|---|---|
| DynamoDB Streams | $0.02 per 100,000 stream read requests |
| Global Tables (replicated WRU) | $1.875 per million (on-demand) |
| Global Tables (replicated WCU) | $0.000975/hour per WCU (provisioned) |
| On-Demand Backup | $0.10 per GB/month |
| Point-in-Time Recovery (PITR) | $0.20 per GB/month |
| Export to S3 | $0.10 per GB exported |
| DAX (cache) | Starting at ~$0.04/hr (`dax.t3.small`) |

### Cost Example Comparison

**Scenario:** 50M reads/day, 5M writes/day, 100 GB storage

| Mode | Monthly Cost | Notes |
|---|---|---|
| **On-Demand** | ~$205 | 1.5B reads × $0.25/M + 150M writes × $1.25/M + $25 storage |
| **Provisioned (auto-scaled)** | ~$80 | 600 RCU + 60 WCU sustained + $25 storage |
| **Provisioned + Reserved (3yr)** | ~$30 | 76% discount on provisioned capacity |

---

**Next:** Part 4 covers Secondary Indexes in depth — GSI vs LSI, projection types, sparse indexes, GSI throttling, and cost analysis.
