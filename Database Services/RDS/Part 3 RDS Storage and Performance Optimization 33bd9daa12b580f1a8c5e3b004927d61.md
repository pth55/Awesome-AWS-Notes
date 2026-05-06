# Part 3: RDS Storage and Performance Optimization

---

## Table of Contents

1. [Understanding RDS Storage Architecture](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
2. [Storage Types Deep Dive](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
3. [IOPS and Throughput Explained](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
4. [Storage Autoscaling Configuration](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
5. [Monitoring Storage Performance](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
6. [Scaling Storage (Hands-On)](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
7. [Changing Storage Type](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
8. [Instance Class Optimization](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
9. [Connection Pooling and Management](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
10. [Query Performance Optimization](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
11. [RDS Proxy for Connection Management](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
12. [Performance Insights Deep Dive](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)
13. [Best Practices Summary](Part%203%20RDS%20Storage%20and%20Performance%20Optimization%2033bd9daa12b580f1a8c5e3b004927d61.md)

---

## 1. Understanding RDS Storage Architecture

RDS storage is built on **Amazon Elastic Block Store (EBS)**, the same block storage used by EC2 instances. However, RDS manages EBS automatically — you never see or configure EBS volumes directly.

### How RDS Storage Works

```
Your RDS Instance
    ├── Compute (DB engine running on virtual server)
    │   └── Uses instance memory for caching
    └── Storage (EBS volume attached to instance)
        ├── Database files (.ibd, .frm for MySQL)
        ├── Transaction logs
        ├── Temp files
        └── Binary logs (if enabled)
```

### Key Facts

- Storage is **persistent** — survives instance restarts
- Storage is **replicated automatically** within the same Availability Zone (AZ) for durability
- Multi-AZ deployments replicate storage **across AZs**
- Storage can be scaled **up** (increased) without downtime
- Storage **cannot** be scaled down after creation
- Maximum storage: **64 TB** for most engines (128 TB for Aurora)

---

## 2. Storage Types Deep Dive

### Storage Types Comparison

![RDS Storage Types](images/rds-storage-types-comparison.svg)

### General Purpose SSD (gp3) — Recommended for Most Workloads

**Key characteristics:**
- Latest generation (released 2020)
- **Baseline performance:** 3,000 IOPS and 125 MB/s throughput **regardless of size**
- Can scale IOPS and throughput independently from storage size
- Cost: ~$0.115 per GB-month (us-east-1)

**Performance scaling:**
```
Storage: 20 GB to 64 TB
Baseline IOPS: 3,000 (included in price)
Max IOPS: 16,000 (additional cost: $0.005 per IOPS per month)
Baseline throughput: 125 MB/s (included)
Max throughput: 1,000 MB/s (additional cost: $0.04 per MB/s per month)
```

**Example configurations:**

| Storage | IOPS | Throughput | Use Case |
|:--------|:-----|:-----------|:---------|
| 100 GB | 3,000 | 125 MB/s | Small app, moderate traffic |
| 500 GB | 6,000 | 250 MB/s | Medium app with read-heavy workload |
| 1 TB | 12,000 | 500 MB/s | Large app with mixed workload |

**When to use gp3:**
- 95% of production workloads
- Cost-sensitive environments
- Workloads with predictable I/O patterns
- You want flexibility to tune IOPS and throughput separately

---

### General Purpose SSD (gp2) — Legacy, Still Supported

**Key characteristics:**
- Previous generation
- Performance **scales with storage size**: 3 IOPS per GB
- Burst capability up to 3,000 IOPS for small volumes
- Cost: ~$0.115 per GB-month

**Performance scaling:**
```
Storage: 20 GB → 60 IOPS (bursts to 3,000)
Storage: 100 GB → 300 IOPS (bursts to 3,000)
Storage: 1,000 GB → 3,000 IOPS (no burst, baseline = burst limit)
Storage: 5,334 GB → 16,000 IOPS (max)
```

**Burst credits:**
- Small volumes (< 1 TB) can burst to 3,000 IOPS
- Credits accumulate when IOPS usage is below baseline
- Credits deplete during high I/O periods
- Once credits are exhausted, performance drops to baseline

**When to use gp2:**
- Legacy applications already using gp2
- You prefer the simplicity of performance tied to size
- **Recommendation:** Migrate to gp3 for better price-performance

---

### Provisioned IOPS SSD (io2) — High Performance

**Key characteristics:**
- Designed for I/O-intensive workloads
- You specify **exact IOPS** (independent of storage size)
- **99.999% durability** (vs 99.9% for gp3/gp2)
- Sub-millisecond latency
- Cost: ~$0.138 per GB-month + $0.119 per IOPS per month

**Performance scaling:**
```
Storage: 100 GB to 64 TB
IOPS: 1,000 to 64,000 (up to 1,000 IOPS per GB)
Throughput: Up to 1,000 MB/s
```

**Example pricing (us-east-1):**
```
500 GB storage + 20,000 IOPS:
- Storage: 500 GB × $0.138 = $69/month
- IOPS: 20,000 × $0.119 = $2,380/month
Total: ~$2,449/month (vs ~$58/month for gp3 with 3,000 IOPS)
```

**When to use io2:**
- Latency-sensitive applications (< 1 ms response time required)
- Sustained high IOPS workloads (> 16,000 IOPS)
- Financial transactions, payment processing
- Large databases with heavy concurrent writes
- You need 99.999% durability SLA

---

### Magnetic Storage (Standard) — Deprecated

**Do not use** for new deployments. Supported only for backward compatibility.

---

## 3. IOPS and Throughput Explained

### What is IOPS?

**IOPS (Input/Output Operations Per Second)** measures how many read/write operations your storage can handle per second.

**One IOP = one I/O operation:**
- For databases, typically 8 KB to 16 KB per operation
- Reading a single row from a table = 1 IOP (usually)
- Writing a row = 1 IOP (plus transaction log writes)

**Example:**
- Application makes 1,000 database queries per second
- Each query reads 3 rows (3 IOPs)
- Total: 3,000 IOPS required

---

### What is Throughput?

**Throughput (MB/s)** measures how much data your storage can transfer per second.

**Important for:**
- Full table scans
- Bulk inserts
- Backup and restore operations
- Large data exports

**Example:**
- Backing up a 100 GB database in 10 minutes
- Required throughput: 100 GB / 600 seconds ≈ 170 MB/s

---

### IOPS vs Throughput

```
IOPS:       How many operations
Throughput: How much data per second
```

**Analogy:**
- IOPS = how many trucks can pass through a checkpoint per minute
- Throughput = how much cargo those trucks can carry

You might have:
- High IOPS, low throughput: Many small transactions (OLTP workloads)
- Low IOPS, high throughput: Few large data transfers (analytical queries, backups)

---

### How to Choose

| Workload Type | IOPS Needs | Throughput Needs | Recommended Storage |
|:--------------|:-----------|:-----------------|:--------------------|
| Small web app | Low (< 3,000) | Low (< 125 MB/s) | gp3 (baseline) |
| E-commerce site | Medium (3,000-10,000) | Medium (125-500 MB/s) | gp3 (scaled) |
| Analytics database | Medium | High (> 500 MB/s) | gp3 (high throughput) |
| High-frequency trading | Very high (> 20,000) | High | io2 |
| Data warehouse (batch) | Low | Very high | gp3 (high throughput) |

---

## 4. Storage Autoscaling Configuration

Storage autoscaling automatically increases storage when you are running out of space.

### How It Works

```
1. RDS monitors free storage space
2. When free space < 10% (or < 6 GB, whichever is lower)
3. AND low space condition lasts 5 minutes
4. AND at least 6 hours since last scaling
5. → RDS increases storage by 10 GB or 10%, whichever is greater
```

### Enable Autoscaling (Hands-On)

```
AWS Console → RDS → Databases → Select your instance → Modify
```

Scroll to **Storage autoscaling:**

```
☑ Enable storage autoscaling
Maximum storage threshold: 1000 GB (set your limit)
```

Click **Continue → Apply immediately** (or during next maintenance window)

---

### Autoscaling Limits

**Minimum increase:** 10 GB  
**Maximum storage:** The limit you set (up to 64 TB)  
**Cooldown period:** 6 hours between scaling events

---

### Why Set a Maximum?

Prevents runaway costs. If your application has a bug (e.g., infinite logging), autoscaling could grow storage to 64 TB, costing thousands of dollars.

**Best practice:** Set maximum to 2-3x your expected growth over 1 year.

---

## 5. Monitoring Storage Performance

### Key CloudWatch Metrics

#### Storage Metrics

| Metric | Description | Alert Threshold |
|:-------|:------------|:----------------|
| `FreeStorageSpace` | Available storage (bytes) | < 10 GB |
| `FreeableMemory` | Available RAM (bytes) | < 20% of instance memory |
| `ReadIOPS` | Read IOPS | Approaching provisioned IOPS |
| `WriteIOPS` | Write IOPS | Approaching provisioned IOPS |
| `ReadThroughput` | Read MB/s | Approaching limit (125 MB/s for gp3) |
| `WriteThroughput` | Write MB/s | Approaching limit |
| `ReadLatency` | Average read latency (ms) | > 10 ms |
| `WriteLatency` | Average write latency (ms) | > 10 ms |

---

#### CPU and Connection Metrics

| Metric | Description | Alert Threshold |
|:-------|:------------|:----------------|
| `CPUUtilization` | CPU usage (%) | > 80% sustained |
| `DatabaseConnections` | Active connections | > 80% of max_connections |
| `DiskQueueDepth` | I/O operations waiting | > 10 (storage bottleneck) |
| `SwapUsage` | Swap space used (bytes) | > 0 (means out of memory) |

---

### Viewing Metrics

```
AWS Console → RDS → Databases → Select instance → Monitoring tab
```

Set timeframe to **1 hour** or **3 hours** to see recent patterns.

---

### Create CloudWatch Alarms

**Example: Alert when free storage < 10 GB**

```
AWS Console → CloudWatch → Alarms → Create alarm
```

```
Metric:      RDS > Per-Database Metrics > FreeStorageSpace
Instance:    myapp-db
Condition:   Lower than 10,000,000,000 bytes (10 GB)
Period:      5 minutes
Datapoints:  2 out of 3 (avoids false alarms)
Notification: SNS topic (email or Slack)
```

---

## 6. Scaling Storage (Hands-On)

### Increase Storage Size

You can increase storage size at any time with **no downtime** (for most engines).

```
AWS Console → RDS → Databases → Select instance → Modify
```

```
Allocated storage: 20 GB → 50 GB (change this)
```

Click **Continue → Apply immediately** (or schedule for maintenance window)

---

### What Happens During Scaling

```
1. RDS initiates storage resize
2. Instance remains online (status: Available)
3. Performance may be temporarily reduced (during disk expansion)
4. Scaling completes in 5-10 minutes (small changes) to hours (large changes)
```

**Important:** You **cannot** decrease storage size. Plan carefully.

---

### Scaling Restrictions

- Can only scale storage **once every 6 hours**
- Cannot scale down (100 GB → 50 GB is not possible)
- For SQL Server, scaling may cause a brief instance restart

---

## 7. Changing Storage Type

You can change storage type (e.g., gp2 → gp3, gp3 → io2).

### gp2 to gp3 Migration (Recommended)

```
AWS Console → RDS → Databases → Modify
```

```
Storage type: General Purpose SSD (gp2) → General Purpose SSD (gp3)
Provisioned IOPS: (only appears if you select io2)
Storage throughput: 125 MB/s (gp3 default)
```

**Cost impact:** Neutral (same price) but better performance control.

---

### gp3 to io2 Migration

Use this when gp3 performance (max 16,000 IOPS) is insufficient.

```
Storage type: gp3 → Provisioned IOPS SSD (io2)
Provisioned IOPS: 20,000 (specify your needs)
```

**Cost impact:** Significantly higher (IOPS are charged separately).

---

### Downtime

- **gp2 ↔ gp3:** No downtime
- **gp3/gp2 → io2:** Brief outage (instance restarts)
- **io2 → gp3:** Brief outage

---

## 8. Instance Class Optimization

Storage isn't the only performance factor. **Instance class** (CPU and memory) matters just as much.

### When to Upgrade Instance Class

**Symptoms:**
- Consistent high CPU (> 80%)
- Queries timing out
- Application slowness
- High `DiskQueueDepth` even though IOPS is not maxed (CPU bottleneck)

---

### Upgrade Path

```
db.t3.micro (2 vCPU, 1 GB)       — Dev/test only
    ↓
db.t3.small (2 vCPU, 2 GB)       — Very light production
    ↓
db.t3.medium (2 vCPU, 4 GB)      — Small production
    ↓
db.m6g.large (2 vCPU, 8 GB)      — Standard production
    ↓
db.m6g.xlarge (4 vCPU, 16 GB)    — Medium production
    ↓
db.r6g.xlarge (4 vCPU, 32 GB)    — Memory-intensive
    ↓
db.r6g.4xlarge (16 vCPU, 128 GB) — Large production
```

---

### Change Instance Class

```
AWS Console → RDS → Modify
```

```
DB instance class: db.t3.micro → db.m6g.large
```

**Downtime:** Yes (5-10 minutes during instance restart)

**Best practice:** Schedule during maintenance window or low-traffic period.

---

## 9. Connection Pooling and Management

Too many database connections can kill performance.

### The Problem

Each connection consumes:
- Memory (1-4 MB per connection)
- CPU (context switching overhead)
- Network sockets

**Default max_connections:**
```
db.t3.micro:    ~60 connections
db.t3.small:    ~130 connections
db.m6g.large:   ~400 connections
db.r6g.xlarge:  ~900 connections
```

Formula (MySQL/MariaDB):
```
max_connections = {DBInstanceClassMemory / 12582880}
```

---

### Application-Side Connection Pooling

Instead of opening/closing connections for every query, use a **connection pool**.

**Without pooling:**
```
App makes 100 requests/sec → 100 new connections/sec → overload
```

**With pooling:**
```
App maintains 10 persistent connections → reuses them → efficient
```

---

### Example: Python with SQLAlchemy

```python
from sqlalchemy import create_engine

engine = create_engine(
    'mysql+pymysql://admin:password@myapp-db.amazonaws.com/mydatabase',
    pool_size=10,          # Keep 10 connections open
    max_overflow=20,       # Allow 20 more during spikes
    pool_recycle=3600,     # Recycle connections after 1 hour
    pool_pre_ping=True     # Check if connection is alive before using
)
```

---

## 10. Query Performance Optimization

Storage speed doesn't matter if your queries are inefficient.

### Identify Slow Queries

Enable **Slow Query Log:**

```
AWS Console → RDS → Parameter groups → Create new parameter group
```

```
Parameter group family: mysql8.0
Name: my-mysql-params
```

**Set parameters:**
```
slow_query_log = 1 (enable)
long_query_time = 2 (log queries taking > 2 seconds)
log_output = FILE
```

Apply parameter group to your instance (requires restart).

---

### Analyze Queries with EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
```

Look for:
- `type: ALL` → Full table scan (bad, add index)
- `rows: 1000000` → Scanning too many rows
- `key: NULL` → No index used

---

### Add Indexes

```sql
CREATE INDEX idx_email ON users(email);
```

Re-run EXPLAIN:
```
type: ref (using index)
rows: 1
```

Much better!

---

## 11. RDS Proxy for Connection Management

**RDS Proxy** is a managed connection pooler.

### Benefits

- Reduces connection overhead (especially for Lambda functions)
- Reuses database connections
- Handles connection spikes gracefully
- Improves failover time (reduces DNS cache issues)

### When to Use

- Serverless apps (Lambda) with unpredictable connection spikes
- Apps that open/close connections frequently
- Multi-tenant apps with many clients

### Cost

~$0.015 per vCPU-hour (~$11/month for 1 vCPU proxy)

---

## 12. Performance Insights Deep Dive

**Performance Insights** shows which queries consume the most resources.

### Enable Performance Insights

```
AWS Console → RDS → Modify
```

```
☑ Enable Performance Insights
Retention: 7 days (free) or 731 days (paid)
```

---

### View Performance Data

```
AWS Console → RDS → Performance Insights
```

You see:
- **Top SQL:** Queries consuming most CPU/memory
- **Waits:** What queries are waiting for (locks, I/O, CPU)
- **Database load:** Total active sessions over time

---

### Interpreting Results

**High "IO:XactSync" waits:**
- Waiting for transaction log writes
- Solution: Upgrade to io2 storage

**High "CPU" waits:**
- Queries consuming too much CPU
- Solution: Optimize queries, add indexes, upgrade instance class

**High "Lock:transaction" waits:**
- Queries blocking each other
- Solution: Shorten transactions, optimize queries

---

## 13. Best Practices Summary

### Storage
- ✅ Use **gp3** for most workloads
- ✅ Enable **storage autoscaling** with a reasonable maximum
- ✅ Set CloudWatch alarms for low storage
- ✅ Use **io2** only for workloads needing > 16,000 IOPS
- ❌ Never use Magnetic storage
- ❌ Don't over-provision storage just to get more IOPS (gp3 scales independently)

### Performance
- ✅ Use **connection pooling** in applications
- ✅ Enable **Performance Insights**
- ✅ Add **indexes** to frequently queried columns
- ✅ Monitor **CPUUtilization, ReadIOPS, WriteIOPS**
- ✅ Use **RDS Proxy** for serverless applications
- ❌ Don't run full table scans in production
- ❌ Don't open a new connection for every query

### Instance Sizing
- ✅ Start small (db.t3.small) and scale up based on metrics
- ✅ Use **Graviton instances (m6g, r6g)** for better price-performance
- ✅ Upgrade instance class if CPU > 80% sustained
- ❌ Don't use burstable (t3) instances for steady production workloads

---