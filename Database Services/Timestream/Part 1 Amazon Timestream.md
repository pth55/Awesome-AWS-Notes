# Part 1 — Amazon Timestream

## Table of Contents

1. [What is Timestream](#1-what-is-timestream)
2. [Core Concepts](#2-core-concepts)
3. [Storage Tiers](#3-storage-tiers)
4. [Writing Data](#4-writing-data)
5. [Querying Data with SQL](#5-querying-data-with-sql)
6. [Scheduled Queries and Derived Tables](#6-scheduled-queries-and-derived-tables)
7. [Security and Access Control](#7-security-and-access-control)
8. [Integrations](#8-integrations)
9. [Pricing and When to Use](#9-pricing-and-when-to-use)

---

## 1. What is Timestream

Amazon Timestream is a **fully managed, serverless time-series database** optimized for storing and querying measurements that are recorded over time — IoT sensor data, application metrics, financial tick data, clickstream events, and monitoring telemetry.

Time-series data has two distinguishing characteristics:
1. **Append-heavy**: Records are almost always written, rarely updated or deleted.
2. **Time-ordered**: Queries are almost always bounded by a time range and ordered by time.

Relational databases and NoSQL databases can store time-series data, but they are not optimized for it. They store data in row-based or document formats where time is just another column. Timestream stores data in a **columnar format with time as a first-class dimension**, enabling efficient time-range scans and aggregations without full table scans.

### Key Properties

- **Serverless**: No instances to provision. Scales automatically.
- **Automatic tiering**: Recent data stays in fast in-memory storage; older data moves to magnetic (S3-backed) storage automatically.
- **SQL-compatible**: Queries use a SQL dialect with time-series functions.
- **Write throughput**: Millions of data points per second.
- **Retention policies**: Separate retention for memory tier and magnetic tier.

---

## 2. Core Concepts

### Data Hierarchy

```
Timestream
  └── Database
        └── Table
              └── Record (row)
                    ├── Time (timestamp — required)
                    ├── Dimensions (metadata identifying the series)
                    └── Measures (the values being recorded)
```

### Dimensions vs Measures

| Concept | Description | Example |
|---|---|---|
| **Dimensions** | Metadata identifying the time series. Indexed and used in WHERE/GROUP BY. | `region`, `instanceId`, `sensorId`, `environment` |
| **Measures** | The values being recorded at each timestamp. Not indexed. | `cpu_utilization`, `temperature`, `error_count` |
| **Time** | The timestamp of the record. Always required. | `2024-12-01T10:00:00Z` |

A time series is defined by its **unique combination of dimensions**. For example, `{region: "us-east-1", instanceId: "i-001"}` is one time series; `{region: "us-east-1", instanceId: "i-002"}` is another.

### Multi-Measure Records

Timestream supports two record formats:
- **Single-measure records**: One record per measurement. Each row has one measure name and one measure value.
- **Multi-measure records**: Multiple measures in a single row (one timestamp, multiple measure columns). More efficient for writes and queries.

Multi-measure records are the recommended format:

```
time                     | region     | instanceId | cpu_util | memory_util | disk_read_bytes
2024-12-01T10:00:00Z     | us-east-1  | i-001      | 72.5     | 45.2        | 1048576
2024-12-01T10:00:10Z     | us-east-1  | i-001      | 71.8     | 46.1        | 2097152
2024-12-01T10:00:00Z     | us-east-1  | i-002      | 35.2     | 68.3        | 524288
```

---

## 3. Storage Tiers

Timestream automatically manages two storage tiers:

| Tier | Storage Type | Performance | Cost | Typical Retention |
|---|---|---|---|---|
| **Memory store** | In-memory (PCIe SSD-backed) | Sub-millisecond queries | $0.036/GB-hour | Recent: 6–24 hours |
| **Magnetic store** | S3-backed columnar | Milliseconds to seconds | $0.03/GB-month | Historical: months to years |

Data is automatically moved from memory to magnetic store based on the **memory store retention period** you configure on the table. There is no manual intervention required.

```bash
# Set memory store retention to 12 hours, magnetic store to 365 days
aws timestream-write update-table \
  --database-name my-metrics-db \
  --table-name ec2-metrics \
  --retention-properties MemoryStoreRetentionPeriodInHours=12,MagneticStoreRetentionPeriodInDays=365
```

### Magnetic Store Writes

By default, writes to Timestream must be **within the memory store retention window**. If you try to write a record with a timestamp older than the memory store retention, it is rejected.

For late-arriving data (IoT devices with intermittent connectivity), enable **magnetic store writes**:

```bash
aws timestream-write update-table \
  --database-name my-metrics-db \
  --table-name sensor-data \
  --magnetic-store-write-properties EnableMagneticStoreWrites=true,\
MagneticStoreRejectedDataLocation='{"S3Configuration": {"BucketName": "my-rejected-data", "EncryptionOption": "SSE_S3"}}'
```

With magnetic store writes enabled, late-arriving data is written directly to magnetic storage. Records that cannot be stored (malformed, duplicate on the exact same timestamp + dimensions) are sent to the configured S3 bucket for inspection.

---

## 4. Writing Data

### Creating a Database and Table

```bash
# Create database
aws timestream-write create-database \
  --database-name my-metrics-db

# Create table
aws timestream-write create-table \
  --database-name my-metrics-db \
  --table-name ec2-metrics \
  --retention-properties MemoryStoreRetentionPeriodInHours=24,MagneticStoreRetentionPeriodInDays=365 \
  --schema '{
    "CompositePartitionKey": [{
      "Type": "DIMENSION",
      "Name": "region",
      "EnforcementInRecord": "REQUIRED"
    }]
  }'
```

### Writing Records (Python)

```python
import boto3
import time

timestream_write = boto3.client('timestream-write', region_name='us-east-1')

def write_ec2_metrics(instance_id, region, cpu, memory, disk_bytes):
    current_time_ms = str(int(time.time() * 1000))

    records = [{
        'Dimensions': [
            {'Name': 'region', 'Value': region},
            {'Name': 'instanceId', 'Value': instance_id},
            {'Name': 'environment', 'Value': 'production'},
        ],
        'MeasureName': 'instance_metrics',  # Multi-measure record name
        'MeasureValueType': 'MULTI',
        'MeasureValues': [
            {'Name': 'cpu_utilization', 'Value': str(cpu), 'Type': 'DOUBLE'},
            {'Name': 'memory_utilization', 'Value': str(memory), 'Type': 'DOUBLE'},
            {'Name': 'disk_read_bytes', 'Value': str(disk_bytes), 'Type': 'BIGINT'},
        ],
        'Time': current_time_ms,
        'TimeUnit': 'MILLISECONDS'
    }]

    response = timestream_write.write_records(
        DatabaseName='my-metrics-db',
        TableName='ec2-metrics',
        Records=records,
        CommonAttributes={}  # Attributes common to all records in the batch
    )
    print(f"Written records: {response['RecordsIngested']['Total']}")

write_ec2_metrics('i-001', 'us-east-1', 72.5, 45.2, 1048576)
```

### Batch Writes

Timestream accepts up to **100 records per batch write request**. For high-throughput workloads, batch records together:

```python
def batch_write(records_list):
    BATCH_SIZE = 100
    for i in range(0, len(records_list), BATCH_SIZE):
        batch = records_list[i:i+BATCH_SIZE]
        timestream_write.write_records(
            DatabaseName='my-metrics-db',
            TableName='ec2-metrics',
            Records=batch
        )
```

### Writing from IoT Core

Timestream integrates natively with AWS IoT Core. An IoT rule action can write IoT device telemetry directly to a Timestream table without Lambda:

```json
{
  "sql": "SELECT * FROM 'sensors/+/telemetry'",
  "actions": [{
    "timestream": {
      "roleArn": "arn:aws:iam::123456789012:role/IoTTimestreamRole",
      "databaseName": "my-metrics-db",
      "tableName": "sensor-data",
      "dimensions": [
        {"name": "deviceId", "value": "${topic(2)}"},
        {"name": "region", "value": "us-east-1"}
      ]
    }
  }]
}
```

---

## 5. Querying Data with SQL

Timestream queries use a SQL dialect with extensions for time-series operations.

### Basic Queries

```sql
-- Find average CPU utilization per instance in the last hour
SELECT instanceId,
       AVG(cpu_utilization) AS avg_cpu,
       MAX(cpu_utilization) AS max_cpu
FROM "my-metrics-db"."ec2-metrics"
WHERE time > ago(1h)
GROUP BY instanceId
ORDER BY avg_cpu DESC;

-- Get the latest metric for each instance
SELECT instanceId,
       cpu_utilization,
       memory_utilization,
       time
FROM "my-metrics-db"."ec2-metrics"
WHERE time = (
    SELECT MAX(time) FROM "my-metrics-db"."ec2-metrics"
    WHERE instanceId = t.instanceId
)
GROUP BY instanceId, cpu_utilization, memory_utilization, time;

-- Time range query
SELECT time, instanceId, cpu_utilization
FROM "my-metrics-db"."ec2-metrics"
WHERE instanceId = 'i-001'
  AND time BETWEEN '2024-12-01 00:00:00' AND '2024-12-01 23:59:59'
ORDER BY time ASC;
```

### Time-Series Functions

```sql
-- INTERPOLATE_LINEAR: Fill gaps in data with linear interpolation
SELECT instanceId,
       INTERPOLATE_LINEAR(
           CREATE_TIME_SERIES(time, cpu_utilization),
           SEQUENCE(min(time), max(time), 10s)  -- Every 10 seconds
       )
FROM "my-metrics-db"."ec2-metrics"
WHERE time > ago(1h)
GROUP BY instanceId;

-- BIN: Bin timestamps into fixed intervals (time bucketing)
SELECT BIN(time, 5m) AS time_bucket,
       instanceId,
       AVG(cpu_utilization) AS avg_cpu
FROM "my-metrics-db"."ec2-metrics"
WHERE time > ago(1h)
GROUP BY BIN(time, 5m), instanceId
ORDER BY time_bucket, instanceId;

-- RATE: Calculate rate of change
SELECT time,
       instanceId,
       RATE(CREATE_TIME_SERIES(time, network_bytes_out)) AS bytes_per_sec
FROM "my-metrics-db"."ec2-metrics"
WHERE time > ago(1h)
GROUP BY time, instanceId;

-- RUNNING_AVG: Moving average
SELECT time,
       instanceId,
       RUNNING_AVG(CREATE_TIME_SERIES(time, cpu_utilization), 5) AS moving_avg_5
FROM "my-metrics-db"."ec2-metrics"
WHERE time > ago(2h)
GROUP BY time, instanceId;
```

### Querying with Python

```python
import boto3

timestream_query = boto3.client('timestream-query', region_name='us-east-1')

query = """
SELECT BIN(time, 5m) AS time_bucket,
       instanceId,
       AVG(cpu_utilization) AS avg_cpu
FROM "my-metrics-db"."ec2-metrics"
WHERE time > ago(1h)
GROUP BY BIN(time, 5m), instanceId
ORDER BY time_bucket
"""

paginator = timestream_query.get_paginator('query')
response_iterator = paginator.paginate(QueryString=query)

for page in response_iterator:
    col_info = page['ColumnInfo']
    col_names = [col['Name'] for col in col_info]
    for row in page['Rows']:
        values = [datum.get('ScalarValue', 'NULL') for datum in row['Data']]
        print(dict(zip(col_names, values)))
```

---

## 6. Scheduled Queries and Derived Tables

Scheduled queries run on a recurring schedule and write results to a derived table. This is useful for pre-aggregating data for dashboards and reducing query latency for repeated aggregations.

```bash
aws timestream-query create-scheduled-query \
  --name "hourly-cpu-summary" \
  --query-string "
    SELECT region,
           instanceId,
           BIN(time, 1h) AS time_bucket,
           AVG(cpu_utilization) AS avg_cpu,
           MAX(cpu_utilization) AS max_cpu
    FROM \""my-metrics-db\"".\""ec2-metrics\""
    WHERE time BETWEEN @scheduled_runtime - 1h AND @scheduled_runtime
    GROUP BY region, instanceId, BIN(time, 1h)
  " \
  --schedule-configuration ScheduleExpression="cron(0 * * * ? *)" \  # Run every hour
  --notification-configuration SnsConfiguration={TopicArn=arn:aws:sns:...} \
  --target-configuration '{"TimestreamConfiguration": {
    "DatabaseName": "my-metrics-db",
    "TableName": "ec2-metrics-hourly",
    "TimeColumn": "time_bucket",
    "DimensionMappings": [
      {"Name": "region", "DimensionValueType": "VARCHAR"},
      {"Name": "instanceId", "DimensionValueType": "VARCHAR"}
    ],
    "MixedMeasureMappings": [{
      "MultiMeasureAttributeMappings": [
        {"SourceColumn": "avg_cpu", "MeasureValueType": "DOUBLE"},
        {"SourceColumn": "max_cpu", "MeasureValueType": "DOUBLE"}
      ]
    }]
  }}' \
  --scheduled-query-execution-role-arn arn:aws:iam::123456789012:role/TimestreamScheduledQueryRole
```

---

## 7. Security and Access Control

### IAM Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "timestream:WriteRecords",
        "timestream:DescribeEndpoints"
      ],
      "Resource": "arn:aws:timestream:us-east-1:123456789012:database/my-metrics-db/table/ec2-metrics"
    },
    {
      "Effect": "Allow",
      "Action": [
        "timestream:Select",
        "timestream:DescribeEndpoints",
        "timestream:CancelQuery",
        "timestream:ListDatabases",
        "timestream:ListTables",
        "timestream:DescribeDatabase",
        "timestream:DescribeTable"
      ],
      "Resource": "*"
    }
  ]
}
```

Note: `timestream:DescribeEndpoints` is a prerequisite for all Timestream operations — the SDK calls this first to discover the regional endpoint.

### Encryption

- **At rest**: All data encrypted with AWS-owned or customer-managed KMS keys.
- **In transit**: TLS enforced for all API calls.
- **KMS integration**: Specify a CMK at database creation:

```bash
aws timestream-write create-database \
  --database-name my-metrics-db \
  --kms-key-id alias/my-timestream-key
```

---

## 8. Integrations

| Service | Integration Type | Use Case |
|---|---|---|
| **AWS IoT Core** | Native rule action | Write IoT device telemetry directly to Timestream |
| **Amazon Kinesis Data Streams** | Lambda consumer → Timestream SDK | High-throughput event stream ingestion |
| **Telegraf** | Plugin | Write server/application metrics (CPU, memory, disk) |
| **Grafana** | Official Timestream data source | Real-time dashboards |
| **Amazon QuickSight** | Native connector | BI dashboards on time-series data |
| **SageMaker** | Export → S3 → SageMaker | ML on time-series data (anomaly detection, forecasting) |
| **Lambda** | SDK | Custom ingestion, processing, alerting |

### Telegraf Configuration

```toml
[[outputs.timestream]]
  region = "us-east-1"
  database_name = "my-metrics-db"
  mapping_mode = "multi-table"
  create_table_if_not_exists = true
  create_table_magnetic_store_retention_period_in_days = 365
  create_table_memory_store_retention_period_in_hours = 24
```

---

## 9. Pricing and When to Use

### Pricing (us-east-1)

| Component | Price |
|---|---|
| Memory store | $0.036/GB-hour |
| Magnetic store | $0.03/GB-month |
| Writes | $0.50/million records |
| Queries | $0.01/GB of data scanned |
| Scheduled queries | $0.01/GB of data scanned during execution |

### Cost Example

Ingesting 1 million IoT sensor readings per hour (multi-measure, 4 measures each):
- Write cost: 1M records × $0.50/million = $0.50/hour = $360/month
- Memory store: ~1 GB active data × $0.036/hr × 720 hrs = $25.92/month
- Magnetic store: accumulated historical data × $0.03/GB

For comparison, storing the same data in DynamoDB:
- Same 1M records/hour in DynamoDB: 1M WCUs/hour × 730 hrs × $1.25/million = $912/month (significantly more expensive for append-only time-series data).

### When to Use Timestream

Use Timestream when:
- Data is **append-only time-series**: metrics, telemetry, sensor readings, log events with timestamps.
- You need **SQL-based time-series queries** (BIN, INTERPOLATE, RATE, RUNNING_AVG).
- You want **automatic hot/cold tiering** without manual lifecycle management.
- Integration with **IoT Core, Telegraf, or Grafana** is needed.
- Workload is **write-heavy** and queries are primarily time-range bounded.

Avoid Timestream when:
- Your data needs **updates** to existing records — Timestream is append-only.
- Your queries span **arbitrary non-time dimensions** without time filters — Timestream requires time-bounded queries.
- Your data volume is small and the cost/complexity is not justified — CloudWatch Metrics may be sufficient.
- You need **full SQL JOINs** across multiple tables — Timestream has limited JOIN support.

---

## Key Takeaways

- Timestream is serverless and append-only. Do not use it for data that needs updates or deletes.
- Separate dimensions (metadata) from measures (values). Dimensions are indexed; measures are not.
- Use multi-measure records instead of single-measure for better write efficiency and query performance.
- Data automatically tiers from fast in-memory storage to cheap magnetic storage based on the retention policy — no manual data lifecycle management.
- BIN, INTERPOLATE, RATE, and RUNNING_AVG are the core time-series SQL functions. Use BIN for time bucketing aggregations.
- For late-arriving IoT data, enable magnetic store writes with an S3 rejection bucket.
