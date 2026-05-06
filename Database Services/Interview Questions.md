# Database Services — Interview Questions

## Table of Contents

1. [Self-Check Questions](#1-self-check-questions)
2. [RDS — Q&A](#2-rds--qa)
3. [DynamoDB — Q&A](#3-dynamodb--qa)
4. [ElastiCache — Q&A](#4-elasticache--qa)
5. [Database Migration Service (DMS) — Q&A](#5-database-migration-service-dms--qa)
6. [DocumentDB — Q&A](#6-documentdb--qa)
7. [Neptune — Q&A](#7-neptune--qa)
8. [Timestream — Q&A](#8-timestream--qa)
9. [QLDB — Q&A](#9-qldb--qa)
10. [OpenSearch — Q&A](#10-opensearch--qa)

---

## 1. Self-Check Questions

**Q: What's the difference between Relational, NoSQL and In-Memory databases?**

A:
- **Relational (RDBMS)**: Data stored in tables with fixed schemas. Relationships enforced via foreign keys. Queried with SQL. Guarantees full ACID transactions. Best for structured data with complex relationships — finance, ERP, CRM. Examples: RDS MySQL/PostgreSQL, Aurora.
- **NoSQL**: Umbrella term for non-relational databases. Schema-flexible. Trade strict consistency for scale and speed. Sub-categories: key-value (DynamoDB, Redis), document (DocumentDB, MongoDB), graph (Neptune), wide-column (Cassandra), time-series (Timestream). Best for large-scale applications with unpredictable access patterns or unstructured data.
- **In-Memory**: Data lives in RAM, not disk. Sub-millisecond read/write latency. Typically used as a cache layer in front of a primary database, not as a primary store. Data can be lost on node restart unless persistence is configured (Redis RDB/AOF). Examples: ElastiCache for Redis, ElastiCache for Memcached.

The key trade-off: Relational = consistency + schema + JOINs. NoSQL = scale + flexibility + speed. In-Memory = raw speed at the cost of durability and capacity.

---

**Q: If you need to store simple data (string/integer) in cache, will you use Redis or Memcached?**

A: Either can work for simple strings and integers — both support the basic `get`/`set` operations. However, **Redis is the better default choice** even for simple data, for these reasons:
- Redis supports persistence (RDB snapshots / AOF) so the cache survives a restart without a cold start.
- Redis supports replication and Multi-AZ automatic failover — Memcached does not.
- Redis ACL and AUTH provide access control — Memcached has no authentication.
- Redis can be extended later with richer data types without replacing the cache tier.

Choose Memcached only if you specifically need its multi-threaded architecture for CPU-bound caching workloads, or if you are already running Memcached and the migration cost outweighs the benefit.

---

**Q: When you want to make sure you get the latest data from DynamoDB, which type of read will you use?**

A: **Strongly consistent read**. By default, DynamoDB reads are eventually consistent — they may return data that is up to ~1 second stale because they can be served from any replica. A strongly consistent read always contacts the primary replica and returns the most recently committed value.

Cost: Strongly consistent reads consume **2× the RCUs** of eventually consistent reads (1 RCU per 4 KB vs 0.5 RCU per 4 KB).

In the SDK, set `ConsistentRead=True` on `GetItem`, `Query`, or `BatchGetItem`. Note: strongly consistent reads are not supported on GSIs — GSIs always return eventually consistent results.

---

**Q: Can you make your DynamoDB highly available? If yes, how?**

A: Yes. DynamoDB is highly available by default within a single region — data is automatically replicated across **3 Availability Zones** in every region. A single AZ failure does not cause any downtime or data loss.

For **multi-region** high availability, enable **DynamoDB Global Tables**. This provides active-active replication — every replica in every configured region accepts reads and writes. If an entire region fails, your application can route to another region with no data loss (conflict resolution is last-writer-wins). Global Tables requires PAY_PER_REQUEST billing or provisioned capacity with Auto Scaling enabled in all regions.

Summary:
- Single-region HA: built-in, no configuration needed.
- Multi-region HA: enable Global Tables, add replicas in target regions.

---

**Q: How can you backup Database services?**

A: Backup approaches vary by service:

| Service | Continuous / PITR | Manual Snapshot | Notes |
|---|---|---|---|
| **RDS** | Automated backups (0–35 days PITR) | Manual snapshots (retained until deleted) | Snapshots can be copied cross-region |
| **Aurora** | PITR (1–35 days) | Manual snapshots | Aurora Backtrack rewinds without snapshot restore |
| **DynamoDB** | PITR (35 days, always-on when enabled) | On-demand backups | Export to S3 for Athena queries; TTL + Streams for archival |
| **DocumentDB** | PITR (1–35 days) | Manual snapshots | Restored to a new cluster, not in-place |
| **ElastiCache** | Not applicable (cache, not source of truth) | RDB snapshots to S3 | Redis only; Memcached has no persistence |
| **Neptune** | PITR (1–35 days) | Manual snapshots | Same Aurora-style storage layer |
| **QLDB** | Continuous journal (immutable, permanent) | Export journal to S3 | Journal itself is the backup; cannot be deleted |
| **Timestream** | Magnetic store retention (configurable) | Not applicable | Cold data retention managed via retention policies |
| **OpenSearch** | Manual snapshots to S3 | Manual snapshots | Automated snapshots taken hourly by AWS (retained 14 days) |

AWS Backup provides a unified policy engine to manage backup schedules, retention, and cross-region copies across RDS, DynamoDB, DocumentDB, Neptune, and EFS from a single place.

---

**Q: What is RDS? What engines does it support?**

A: Amazon RDS (Relational Database Service) is a fully managed relational database service. AWS handles provisioning, patching, automated backups, Multi-AZ replication, storage scaling, and monitoring. You retain control over the database engine configuration but have no OS-level access.

Supported engines:
- **MySQL** (5.7, 8.0)
- **PostgreSQL** (13, 14, 15, 16)
- **MariaDB** (10.5, 10.6, 10.11)
- **Oracle** (Standard Edition 2, Enterprise Edition — BYOL or License Included)
- **Microsoft SQL Server** (Express, Web, Standard, Enterprise — BYOL or License Included)
- **Amazon Aurora** (MySQL-compatible and PostgreSQL-compatible) — custom AWS engine, listed separately but managed through RDS

---

**Q: What is RDS pricing?**

A: RDS pricing has four components:

1. **Instance hours**: Charged per hour the DB instance is running. Price depends on instance class (e.g., `db.t3.medium`, `db.r6g.large`), engine, and license model. Example: `db.t3.medium` MySQL ≈ $0.068/hr in us-east-1.
2. **Storage**: Per GB-month for EBS volumes. gp3 ≈ $0.115/GB-month. io1 ≈ $0.125/GB-month + $0.10/IOPS-month.
3. **Backup storage**: Free up to 100% of the DB storage size. Beyond that: $0.095/GB-month.
4. **Data transfer**: Inbound free. Outbound to internet charged at standard AWS rates. Transfer between RDS and EC2 in the same AZ is free; cross-AZ is $0.01/GB.

Cost reduction options:
- **Reserved Instances**: 1-year or 3-year commitment for up to ~42% (1-yr) or ~60% (3-yr) discount.
- **Multi-AZ**: Roughly doubles instance cost (you pay for standby instance hours).
- **Read Replicas**: Each replica is billed as a separate instance.

---

**Q: Can you make your RDS instance highly available? If yes, how?**

A: Yes, via **Multi-AZ deployment**:

- **Multi-AZ DB Instance** (traditional): AWS provisions a synchronous standby replica in a different AZ. All writes are confirmed on both primary and standby before acknowledging to the application. On primary failure, AWS automatically promotes the standby and updates the DNS endpoint. No data loss (RPO = 0). Failover time ~1–2 minutes (RTO).
- **Multi-AZ DB Cluster** (newer, PostgreSQL and MySQL): Two readable standby instances across two AZs. Uses semi-synchronous replication. Faster failover (~35 seconds). Standbys can serve read traffic.
- **Aurora**: Inherently multi-AZ — 6-copy storage replication across 3 AZs by default. Up to 15 read replicas. Failover to a replica in ~30 seconds.

The standby in traditional Multi-AZ is not accessible for reads — it exists solely for failover. To scale reads, create separate read replicas.

---

**Q: What is read replica in RDS and how it works?**

A: A read replica is a copy of the primary RDS instance that receives asynchronous replication of all writes. Applications direct read-only queries to the replica's endpoint, offloading read traffic from the primary.

How it works:
1. RDS uses the database engine's native asynchronous replication (MySQL binlog replication, PostgreSQL streaming replication).
2. Writes go to the primary. The primary ships transaction logs to the replica asynchronously.
3. The replica applies the changes. There is a small replication lag (typically milliseconds to seconds depending on write volume).
4. Replicas are read-only. You cannot write to them directly.
5. A replica can be promoted to a standalone primary — this breaks replication permanently.

Key facts:
- Up to 5 read replicas for MySQL/PostgreSQL/MariaDB, 15 for Aurora.
- Cross-region replicas are supported.
- Replicas use a separate endpoint — your application must be coded to route reads there.
- Replica lag is a CloudWatch metric (`ReplicaLag`). Monitor it to detect replication falling behind.

---

**Q: What RDS operational (maintenance, monitoring) practices do you know?**

A:

**Maintenance:**
- Configure the **maintenance window** to a low-traffic period (e.g., Sunday 2–3 AM). AWS applies minor patches and engine upgrades during this window.
- Use **auto minor version upgrade** for non-production. Review and test major version upgrades manually.
- Enable **Enhanced Monitoring** (1-second OS metrics) for production instances to diagnose CPU, memory, and I/O issues beyond CloudWatch's 1-minute resolution.
- Use **Performance Insights** to identify the top SQL queries consuming the most DB time. Free for most instance types, 7-day retention.

**Monitoring CloudWatch metrics to watch:**
- `CPUUtilization` — sustained >80% signals the need to scale up or optimize queries.
- `FreeStorageSpace` — alarm when below 10% of allocated storage.
- `ReadIOPS` / `WriteIOPS` — compare against provisioned IOPS to detect saturation.
- `DatabaseConnections` — approaching `max_connections` causes connection refused errors.
- `ReplicaLag` — for read replicas; >30s suggests replication falling behind.
- `FreeableMemory` — low memory triggers swap and degrades performance.

**Operational practices:**
- Set up CloudWatch alarms with SNS notifications for all critical metrics above.
- Use AWS Trusted Advisor or RDS recommendations for instance right-sizing.
- Enable deletion protection on production instances.
- Test failover periodically using "Reboot with failover" on non-critical hours.

---

**Q: What RDS capacity planning best practices do you know?**

A:

- **Right-size the instance class**: Start with a `db.t3` for dev/test. Use `db.r6g` (memory-optimized) for production workloads that benefit from large buffer pools. Avoid `db.t3` in production due to CPU credits bursting.
- **Storage**: Use **gp3** by default — decoupled IOPS from size, cheaper than gp2 at most sizes. Enable **storage auto-scaling** with a maximum threshold to prevent unexpected cost spikes.
- **IOPS estimation**: `max_connections × avg_queries_per_sec × avg_IO_per_query`. Provision at least 20–30% headroom above measured peak.
- **Connection pooling**: Each DB connection consumes memory. Use **RDS Proxy** or application-level connection pooling (PgBouncer, ProxySQL) to keep `DatabaseConnections` well below `max_connections`.
- **Read scaling**: Add read replicas before the primary is CPU-saturated. Route 80/20 read/write traffic early rather than reacting to a crisis.
- **Reserved Instances**: For stable production workloads running 24/7, purchase 1-year Reserved Instances to save ~42% over on-demand pricing.
- **Aurora Serverless v2**: For variable workloads, Aurora Serverless v2 auto-scales ACUs (Aurora Capacity Units) between a minimum and maximum — pay only for what you use.

---

**Q: What RDS testing and profiling best practices do you know?**

A:

- **Performance Insights**: The first tool to reach for. Identifies the top SQL statements by `db load` (average active sessions). Drill into a spike to see the exact query, wait events, and user.
- **slow query log**: Enable `slow_query_log` (MySQL) or `log_min_duration_statement` (PostgreSQL) to capture queries exceeding a time threshold. Export to CloudWatch Logs for analysis.
- **EXPLAIN / EXPLAIN ANALYZE**: Run `EXPLAIN` on slow queries to view the execution plan. Look for full table scans (`Seq Scan` in PG, `ALL` in MySQL). Add indexes to eliminate them.
- **Load testing**: Use tools like `pgbench` (PostgreSQL), `sysbench` (MySQL), or custom scripts to simulate production traffic on a non-production instance before launching. Test with the same instance class as production.
- **Restore testing**: Periodically restore from a production snapshot to a test instance to verify backup integrity and measure restore time.
- **Failover testing**: Trigger "Reboot with failover" in a maintenance window to measure actual RTO and confirm the application reconnects correctly.
- **Read replica lag under load**: During load tests, monitor `ReplicaLag` on replicas to confirm they can keep pace with write volume.

---

**Q: What are the advantages of RDS over manually managed databases?**

A:

| Aspect | RDS (Managed) | Self-managed on EC2 |
|---|---|---|
| Patching | AWS patches OS and engine automatically | You patch OS, DB engine, and dependencies |
| Backups | Automated PITR backups, one-click restore | You script backups, manage S3 lifecycle, test restores |
| High availability | Multi-AZ with automatic failover, DNS flip | You configure replication, monitor, and fail over manually |
| Storage scaling | Storage auto-scaling, no downtime | Resize EBS volume, may require restart |
| Monitoring | CloudWatch, Performance Insights, Enhanced Monitoring built-in | You install and configure monitoring agents |
| Read replicas | One-click creation, managed lag monitoring | You configure replication, manage endpoints |
| Compliance | HIPAA, PCI-DSS, SOC eligible out of the box | You implement and audit controls yourself |
| Operational cost | Higher per-instance cost | Lower per-instance cost, much higher ops cost |

RDS trades raw cost per instance for dramatically reduced operational overhead. The break-even point — where self-managing becomes cheaper — is typically only at very large scale (hundreds of instances) where dedicated database engineers are already employed.

---

**Q: What is the difference between multi-AZ deployment and read replicas in RDS?**

A:

| Aspect | Multi-AZ | Read Replica |
|---|---|---|
| Purpose | High availability / disaster recovery | Read scaling / offloading |
| Replication type | Synchronous | Asynchronous |
| Standby accessible for reads? | No (traditional Multi-AZ) | Yes — has its own read endpoint |
| Automatic failover? | Yes — DNS flips in ~1–2 min | No — must be promoted manually |
| Data loss on failure | Zero (RPO = 0, synchronous) | Possible (RPO = replication lag) |
| Same region required? | Yes (different AZs, same region) | No — cross-region replicas supported |
| Increases read throughput? | No | Yes |
| Cost | ~2× instance cost | Separate instance cost per replica |

In short: Multi-AZ is for **resilience** (withstanding failures). Read replicas are for **performance** (scaling reads). They serve different purposes and are often used together.

---

**Q: What security features does RDS provide?**

A:

- **Network isolation**: Deploy in a VPC. Use private subnets and security groups to restrict access to specific application security groups. Disable public accessibility for production.
- **IAM database authentication**: Generate short-lived auth tokens via IAM role instead of static passwords. Supported for MySQL and PostgreSQL.
- **Encryption at rest**: KMS encryption for all data, backups, snapshots, and read replicas. Must be enabled at creation time.
- **Encryption in transit**: SSL/TLS for all client connections. Enforce TLS-only by setting `rds.force_ssl=1` (PostgreSQL) or `require_secure_transport=ON` (MySQL).
- **Database user management**: Native DB users with role-based permissions inside the engine.
- **Secrets Manager integration**: Rotate database passwords automatically on a schedule without application downtime.
- **Enhanced Monitoring + CloudTrail**: OS-level metrics and API call audit trail for compliance.
- **Deletion protection**: Prevents accidental instance deletion from Console or CLI.
- **VPC endpoint**: Keep traffic between application and RDS within the AWS network (no internet transit needed even for public subnets).

---

**Q: What is AWS Aurora?**

A: Aurora is Amazon's proprietary relational database engine, offered in two flavors: **Aurora MySQL-compatible** and **Aurora PostgreSQL-compatible**. It speaks the same wire protocol as MySQL/PostgreSQL — existing applications connect to it without changing code.

Aurora reimagines the storage layer:
- **6-copy replication** across 3 AZs (2 copies per AZ). Writes are acknowledged after 4 of 6 copies confirm. Tolerates the loss of 2 copies without write impact and 3 copies without read impact.
- **Shared storage** between all instances — replicas do not maintain separate copies of data. Adding replicas adds compute (connections, read throughput) without multiplying storage cost.
- **Storage auto-grows** in 10 GB increments up to 128 TiB. No storage provisioning required.
- **Faster failover**: ~30 seconds (vs ~1–2 minutes for RDS Multi-AZ) because replicas already see the same storage.
- **Up to 15 read replicas** (vs 5 for standard RDS).
- **Aurora Serverless v2**: Auto-scales compute between a configured min/max in seconds.
- **Aurora Global Database**: Cross-region replication with ~1-second lag and sub-1-minute failover.

Aurora costs ~20% more per instance than equivalent RDS, but the storage efficiency (no per-replica storage cost), faster failover, and higher replica count often justify it for production workloads.

---

**Q: You need to make a snapshot in RDS at a specific time, is that possible? If yes, how?**

A: Yes, in two ways:

1. **Manual snapshot on demand**: Trigger a snapshot at any time via the Console, CLI, or SDK. It captures the DB state at that exact moment. Retained until you delete it.

```bash
aws rds create-db-snapshot \
  --db-instance-identifier my-rds-instance \
  --db-snapshot-identifier my-snapshot-20241201-1400
```

2. **Scheduled via EventBridge + Lambda**: Automate a snapshot at a specific time by creating an EventBridge (CloudWatch Events) scheduled rule that invokes a Lambda function calling `create-db-snapshot`.

```bash
# EventBridge rule: run at 2 PM UTC daily
aws events put-rule \
  --name "DailyRDSSnapshot" \
  --schedule-expression "cron(0 14 * * ? *)" \
  --state ENABLED

# Lambda target calls create_db_snapshot via boto3
```

Note: Automated backups (PITR) run continuously in the background — they are not point-in-time snapshots but rather a continuous transaction log backup that allows restore to any second within the retention window. Manual snapshots are discrete full copies at the moment you trigger them.

---

## 2. RDS — Q&A

**Q: Can you SSH into an RDS instance?**
A: No. RDS is fully managed. AWS owns the OS.

**Q: Can you use a standby instance (Multi-AZ) for reads?**
A: No (traditional Multi-AZ). Yes (Multi-AZ DB Cluster standbys).

**Q: How do you scale read traffic on RDS?**
A: Create Read Replicas. Use a different endpoint for reads.

**Q: What is the RPO for Multi-AZ?**
A: Zero. Synchronous replication means no data loss.

**Q: What is the RTO for Multi-AZ?**
A: ~1–2 minutes (detection + DNS flip + standby promotion).

**Q: What happens to your app endpoint after Multi-AZ failover?**
A: Nothing — same DNS name, but the IP behind it changes. App must reconnect.

**Q: Can you encrypt an existing unencrypted RDS instance?**
A: Not directly. Take snapshot → copy snapshot with encryption enabled → restore from encrypted snapshot.

**Q: What is the max storage for RDS?**
A: 64 TB (standard engines), 128 TiB (Aurora).

**Q: How many read replicas can RDS have?**
A: 5 (MySQL / PostgreSQL / MariaDB), 15 (Aurora).

**Q: What is Aurora Backtrack?**
A: Rewind the Aurora database to a prior point in time without restoring from a snapshot. Available for Aurora MySQL only. No data export required.

**Q: What is the difference between Aurora and RDS MySQL?**
A: Same SQL/wire protocol, completely different storage engine. Aurora uses a 6-copy shared distributed storage layer with compute-storage separation, faster failover (~30s), and up to 15 replicas. RDS MySQL uses single-node EBS storage with synchronous standby replication.

**Q: What triggers Multi-AZ failover?**
A: Host failure, AZ failure, storage failure, network failure, manual "Reboot with failover." NOT triggered by: bad SQL queries, high CPU, wrong password, application errors.

**Q: What is the difference between automated backups and manual snapshots in RDS?**
A: Automated backups are retained 0–35 days, deleted when the instance is deleted (unless you take a final snapshot). Manual snapshots persist until you explicitly delete them. Both can be used to restore.

**Q: Can RDS read replicas be in a different region?**
A: Yes. Cross-region read replicas are supported for MySQL, PostgreSQL, MariaDB, and Aurora. Useful for disaster recovery and reducing read latency for global users.

**Q: What is RDS Proxy and when would you use it?**
A: RDS Proxy is a fully managed connection pooler that sits between your application and RDS. Use it when Lambda functions (which can open thousands of connections on burst) would otherwise exhaust the database's connection limit. Also reduces failover time to seconds by maintaining a warm connection pool.

**Q: Can you promote an RDS read replica to a standalone instance?**
A: Yes. Promotion is one-way and permanent — the replica becomes an independent primary. The replication link with the source is broken.

**Q: What is the difference between gp2 and gp3 storage in RDS?**
A: gp3 decouples IOPS from storage size — you can provision up to 16,000 IOPS regardless of disk size. gp2 ties IOPS to storage (3 IOPS/GB baseline, up to 16,000 IOPS at ~5.3 TB). gp3 is typically cheaper and more flexible.

---

## 3. DynamoDB — Q&A

**Q: What is the difference between a partition key and a sort key?**
A: The partition key determines which physical partition the item lives on. The sort key is the second part of the composite primary key and sorts items within the same partition, enabling range queries (`begins_with`, `between`, `<`, `>`). A simple primary key has only a partition key.

**Q: What is a strongly consistent read vs an eventually consistent read in DynamoDB?**
A: Strongly consistent read always returns the latest written value. It costs 2× the RCUs of an eventually consistent read and may return a slightly higher latency response. Eventually consistent reads may return stale data up to ~1 second old but cost 0.5 RCUs per 4 KB.

**Q: What happens when a DynamoDB partition is hot?**
A: A hot partition is one that receives a disproportionate share of traffic. It throttles requests once it hits 3,000 RCUs or 1,000 WCUs per second. Solutions: use a high-cardinality partition key, add a random suffix for write sharding, use DAX for read-heavy hot keys, or switch to on-demand mode.

**Q: What is the difference between GSI and LSI?**
A: LSI (Local Secondary Index) uses the same partition key as the base table with a different sort key. Can only be created at table creation time. Limit: 10 GB per partition key value. Supports strong consistency. GSI (Global Secondary Index) can use any partition key and sort key. Can be created or deleted at any time. Has its own read/write capacity. Only supports eventual consistency.

**Q: What is DynamoDB on-demand mode and when should you use it?**
A: On-demand mode scales automatically — you pay per request with no provisioning. Use it for unpredictable or spiky traffic, new tables where usage is unknown, or dev/test environments. Provisioned mode is cheaper for predictable, steady workloads.

**Q: What is DynamoDB DAX and when does it NOT help?**
A: DAX is an in-memory write-through cache for DynamoDB. It helps with read-heavy workloads and hot key access patterns (sub-millisecond latency). It does NOT help with write-heavy workloads, strongly consistent read requirements (DAX serves eventual reads only), or when you need fine-grained cache TTL per key.

**Q: How does DynamoDB Global Tables work and what is the conflict resolution strategy?**
A: Global Tables provides active-active multi-region replication using DynamoDB Streams internally. Each region accepts reads and writes. Conflict resolution is last-writer-wins based on wall clock timestamp. Design recommendation: avoid concurrent writes to the same item from multiple regions.

**Q: What is TTL in DynamoDB and is it instant?**
A: TTL automatically deletes items after a Unix epoch timestamp stored in a designated attribute. Deletions are free (no WCUs consumed) but are not instantaneous — they may take up to 48 hours. Always filter expired items in application code when strict expiry is required.

**Q: How do DynamoDB transactions work and what do they cost?**
A: `TransactWriteItems` and `TransactGetItems` group up to 100 items across multiple tables into an atomic ACID operation. They cost 2× normal read/write units due to the two-phase commit protocol. Use transactions only when atomicity is genuinely required.

**Q: What is the maximum item size in DynamoDB?**
A: 400 KB per item (including attribute names and values). For larger items, store the data in S3 and keep a reference (S3 key/URL) in DynamoDB.

**Q: How do you model one-to-many relationships in DynamoDB?**
A: Use a composite primary key where the partition key is the parent entity ID and the sort key is a prefix + child entity ID (e.g., PK=`USER#123`, SK=`ORDER#2024-12-01`). Query with `KeyConditionExpression` on the partition key to retrieve all children.

**Q: What is DynamoDB Streams and how long are records retained?**
A: Streams capture all item-level changes as an ordered log. Records are retained for 24 hours. Lambda event source mapping is the standard way to consume them. Use `NEW_AND_OLD_IMAGES` stream view type for audit and change detection.

---

## 4. ElastiCache — Q&A

**Q: What is the difference between Redis and Memcached in ElastiCache?**
A: Redis supports rich data structures (strings, hashes, lists, sets, sorted sets, streams), persistence (RDB/AOF), replication, Multi-AZ failover, Pub/Sub, Lua scripting, and Redis ACL. Memcached supports only string values, is multi-threaded, and has no persistence or replication. Choose Redis unless you need pure horizontal scaling with no persistence requirement and the simplest possible setup.

**Q: What is ElastiCache cluster mode and what does it enable?**
A: Cluster mode enabled splits data across multiple shards (up to 500), each responsible for a range of hash slots (0–16,383). This enables horizontal write scaling. Cluster mode disabled has one shard — all data lives on a single primary with up to 5 read replicas. Use cluster mode enabled when your data exceeds the memory of a single node or you need horizontal write throughput.

**Q: How does ElastiCache Multi-AZ failover work?**
A: When the primary node fails, ElastiCache promotes the least-lagging read replica (in a different AZ). The cluster DNS endpoint is updated. Total failover time is typically 30–60 seconds. Applications must use the DNS endpoint (not a hardcoded IP) and implement retry logic with exponential backoff.

**Q: What is cache stampede (thundering herd) and how do you prevent it?**
A: When a popular cached key expires, many concurrent requests all miss the cache simultaneously and all hit the database at once. Prevention strategies: mutex locking (only one request rebuilds the cache), probabilistic early expiration (rebuild before expiry), stale-while-revalidate (serve stale data while rebuilding), TTL jitter (randomize expiry to spread reloads).

**Q: What is the difference between Cache-Aside and Write-Through caching?**
A: Cache-Aside: application manages the cache — on miss, reads from DB and populates cache. Simple, lazy population. Risk: cache miss on first access after expiry. Write-Through: every write goes to both cache and DB synchronously. Cache is always warm but adds write latency and wastes cache space for rarely-read data.

**Q: What eviction policy should you use for a session store?**
A: `noeviction` — raise an error if memory is full rather than silently evicting sessions. Silently evicting sessions would log users out without warning. For a general-purpose cache where staleness is acceptable, `allkeys-lru` (evict least-recently-used key) is the standard choice.

**Q: What is ElastiCache Global Datastore?**
A: Cross-region replication for ElastiCache for Redis. One primary cluster (accepts writes) with up to two secondary clusters in other regions (read-only). Replication lag is typically under 1 second. Used for geo-distributed read latency and cross-region disaster recovery.

**Q: What is Redis AUTH vs ACL and when do you use each?**
A: AUTH is a single shared password for all connections (Redis 5.x and earlier). ACL (Redis 6.x+) provides named users with individual passwords and per-command/per-key access restrictions. Use ACL for any production deployment where multiple applications or teams share a cluster.

**Q: Can you add TLS to an existing ElastiCache cluster?**
A: No. TLS (transit encryption) must be enabled at cluster creation time. An existing non-TLS cluster cannot have TLS added. Plan for TLS from day one in production.

**Q: What is replication lag in ElastiCache and when does it matter?**
A: Replication lag is the delay between a write on the primary and its appearance on replicas. For most applications, sub-second lag is acceptable. It matters when: your application requires read-your-writes consistency (always read from primary after a write), or during failover when the promoted replica may be slightly behind the failed primary.

---

## 5. Database Migration Service (DMS) — Q&A

**Q: What is the difference between a homogeneous and heterogeneous migration in DMS?**
A: Homogeneous: source and target use the same engine (MySQL → RDS MySQL). Schema is compatible — no conversion needed. Heterogeneous: source and target use different engines (Oracle → Aurora PostgreSQL). Schema and data types differ — AWS Schema Conversion Tool (SCT) must convert the DDL before DMS migrates the data.

**Q: What is CDC in DMS and how does it work?**
A: Change Data Capture captures ongoing changes (inserts, updates, deletes) from the source database after the initial full load. DMS reads the database's native change log: binary log (MySQL), Write-Ahead Log (PostgreSQL), redo log (Oracle), transaction log (SQL Server). CDC enables minimal-downtime migrations — load the full snapshot, then replay changes until the lag reaches zero, then cut over.

**Q: What is AWS Schema Conversion Tool (SCT) and is it free?**
A: SCT converts schema DDL from one engine dialect to another (e.g., Oracle PL/SQL procedures to PostgreSQL functions). It also produces an assessment report showing which objects can be automatically converted and which require manual attention. SCT is a free download — it is not a cloud service.

**Q: What does a DMS replication instance actually do?**
A: It is a managed EC2 instance that AWS runs in your VPC. It connects to the source endpoint, reads data (full load or CDC), transforms it if needed, and writes to the target endpoint. You choose the instance size (e.g., `dms.t3.medium` for small migrations, `dms.r5.xlarge` for large/high-throughput migrations).

**Q: What does DMS NOT migrate?**
A: DMS migrates data only. It does not migrate: secondary indexes, stored procedures, triggers, views, sequences, user accounts, grants/permissions, or table constraints. These must be recreated manually or via SCT after migration.

**Q: What is the DMS data validation feature?**
A: After migration, DMS can compare row counts and checksums between source and target to verify completeness. It reports mismatches per table. Enable it via the task settings `ValidationEnabled: true`.

**Q: How do you minimize downtime during an RDS migration?**
A: Use Full Load + CDC mode. Full Load copies all existing data. CDC then captures changes that occurred during the full load. Once CDC lag reaches near-zero, take a brief maintenance window, stop writes to the source, let CDC catch up, then point the application at the new target.

---

## 6. DocumentDB — Q&A

*Content coming soon.*

---

## 7. Neptune — Q&A

*Content coming soon.*

---

## 8. Timestream — Q&A

*Content coming soon.*

---

## 9. QLDB — Q&A

*Content coming soon.*

---

## 10. OpenSearch — Q&A

*Content coming soon.*
