# Part 5 — ElastiCache High Availability, Global Datastore, Security & Migration

## Table of Contents

1. [Multi-AZ Automatic Failover](#1-multi-az-automatic-failover)
2. [Global Datastore](#2-global-datastore)
3. [Security — Authentication and Authorization](#3-security--authentication-and-authorization)
4. [Security — Network and Encryption](#4-security--network-and-encryption)
5. [CloudWatch Monitoring Reference](#5-cloudwatch-monitoring-reference)
6. [Migration from Self-Managed Redis](#6-migration-from-self-managed-redis)
7. [Migration from Memcached to Redis](#7-migration-from-memcached-to-redis)
8. [Maintenance and Upgrades](#8-maintenance-and-upgrades)

---

## 1. Multi-AZ Automatic Failover

### How Failover Works

When Multi-AZ is enabled and a primary node fails:

1. ElastiCache detects the failure (typically within **10–30 seconds**).
2. The service promotes the replica in the least-lagging state to primary.
3. The DNS CNAME for the **primary endpoint** is updated to point to the new primary.
4. The previous primary, once recovered, rejoins as a replica.

Total failover time (detection + DNS propagation + client reconnect) is typically **30–60 seconds**. Applications using the cluster endpoint (not a hardcoded IP) recover automatically once the DNS TTL expires.

### Cluster Mode Disabled — Failover Behavior

With one shard and one or more read replicas, failover is straightforward:

```
Before Failover:
  Primary (AZ-a) ─── Replica (AZ-b) ─── Replica (AZ-c)
  ↑ Primary Endpoint points here

After Primary (AZ-a) fails:
  [Failed] (AZ-a)  ─── Replica promoted (AZ-b) ─── Replica (AZ-c)
                        ↑ Primary Endpoint now points here
```

At least **one read replica is required** for automatic failover. A standalone primary node (no replicas) does not benefit from Multi-AZ automatic promotion.

### Cluster Mode Enabled — Failover Behavior

Each shard independently promotes its own replica if the shard's primary fails. Other shards are unaffected. This is why cluster mode is more resilient for large datasets split across many shards.

```
Shard 1: Primary (AZ-a) → fails  →  Replica-1 (AZ-b) promoted
Shard 2: Primary (AZ-b)           →  (unaffected)
Shard 3: Primary (AZ-c)           →  (unaffected)
```

### Enabling Multi-AZ

```bash
aws elasticache modify-replication-group \
  --replication-group-id my-redis-cluster \
  --multi-az-enabled \
  --automatic-failover-enabled \
  --apply-immediately
```

Both `--multi-az-enabled` and `--automatic-failover-enabled` must be set together. Multi-AZ without automatic failover only deploys replicas across AZs but does not auto-promote on failure.

### Testing Failover

```bash
# Trigger a manual failover to test your application's reconnect behavior
aws elasticache test-failover \
  --replication-group-id my-redis-cluster \
  --node-group-id 0001  # Shard ID; for cluster mode disabled: 0001
```

This is a non-destructive test — the current primary is gracefully demoted and the least-lagging replica is promoted. Use this during a maintenance window before going to production.

### Application-Side Resilience

DNS-based failover only works if your application uses the cluster endpoint DNS name — not a hardcoded IP.

```python
import redis
from redis.retry import Retry
from redis.backoff import ExponentialBackoff

# Configure retry logic for automatic reconnect after failover
retry = Retry(ExponentialBackoff(), retries=5)

r = redis.Redis(
    host='my-redis-cluster.xxxxxx.ng.0001.use1.cache.amazonaws.com',  # Use DNS endpoint
    port=6379,
    ssl=True,
    retry=retry,
    retry_on_error=[redis.ConnectionError, redis.TimeoutError],
    socket_connect_timeout=5,
    socket_timeout=5,
    health_check_interval=30
)
```

Always handle `redis.ConnectionError` and `redis.TimeoutError` in your application and retry with backoff. Cache misses during a failover window are acceptable — your application should fall back to the database for that window.

---

## 2. Global Datastore

Global Datastore provides **cross-region replication** for ElastiCache for Redis. One region is the **primary cluster** (accepts reads and writes); up to **two secondary clusters** in other regions receive replicated writes.

### Use Cases

- **Low-latency reads** for a globally distributed application (serve reads from the nearest region).
- **Disaster recovery** across regions (failover to a secondary if the primary region fails).
- **Data locality compliance** (keep copies of data in specific geographic regions).

### Architecture

```
Primary Region (us-east-1)
  Primary Cluster ──writes──► Replication ──► Secondary Cluster (eu-west-1)
                                          └──► Secondary Cluster (ap-southeast-1)
```

Replication lag is typically **under 1 second** for standard workloads. Secondary clusters are read-only. You cannot write to a secondary cluster directly.

### Creating a Global Datastore

```bash
# Create the primary cluster first (must be Redis 6.x+ with cluster mode disabled or enabled)
aws elasticache create-replication-group \
  --replication-group-id primary-cluster \
  --replication-group-description "Global Datastore primary" \
  --engine redis \
  --engine-version 7.0 \
  --node-type cache.r6g.large \
  --num-cache-clusters 2 \
  --multi-az-enabled \
  --automatic-failover-enabled \
  --region us-east-1

# Create the Global Datastore using the primary cluster
aws elasticache create-global-replication-group \
  --global-replication-group-id-suffix my-global-store \
  --primary-replication-group-id primary-cluster \
  --region us-east-1

# Add a secondary cluster in eu-west-1
aws elasticache create-replication-group \
  --replication-group-id secondary-eu \
  --replication-group-description "Global Datastore secondary EU" \
  --global-replication-group-id ldgnf-my-global-store \
  --node-type cache.r6g.large \
  --region eu-west-1
```

### Cross-Region Failover

If the primary region becomes unavailable:

```bash
# Promote a secondary cluster to primary
aws elasticache failover-global-replication-group \
  --global-replication-group-id ldgnf-my-global-store \
  --primary-region eu-west-1 \
  --primary-replication-group-id secondary-eu
```

After promotion, the former secondary in `eu-west-1` becomes the new primary and starts accepting writes. Update your application's primary endpoint configuration to point to the new primary region.

### Global Datastore Pricing

- Standard ElastiCache node pricing applies in each region.
- **Cross-region data transfer**: ~$0.02 per GB transferred out from primary to secondary regions.
- No additional Global Datastore management fee.

---

## 3. Security — Authentication and Authorization

### Redis AUTH

AUTH is a simple password mechanism for Redis 5.x and earlier. The server requires a password on every connection. All clients share the same password — there is no per-user access control.

```bash
# Create cluster with AUTH token
aws elasticache create-replication-group \
  --replication-group-id secure-cluster \
  --auth-token "MyStrongPassword123!" \
  --transit-encryption-enabled \  # AUTH requires TLS
  ...
```

```python
import redis
r = redis.Redis(host='...', port=6379, ssl=True, password='MyStrongPassword123!')
```

AUTH tokens must be 16–128 characters and may contain only printable ASCII characters (excluding `@`, `/`, `"`).

### Rotating AUTH Tokens

ElastiCache supports in-place AUTH token rotation without downtime using a **ROTATE** strategy:

```bash
# Step 1: Add a new token while keeping the old one active
aws elasticache modify-replication-group \
  --replication-group-id secure-cluster \
  --auth-token "NewStrongPassword456!" \
  --auth-token-update-strategy ROTATE \
  --apply-immediately

# Step 2: Update all application clients to use the new token

# Step 3: Remove the old token
aws elasticache modify-replication-group \
  --replication-group-id secure-cluster \
  --auth-token "NewStrongPassword456!" \
  --auth-token-update-strategy SET \
  --apply-immediately
```

During the ROTATE window, both the old and new tokens are accepted. This allows zero-downtime rotation.

### Redis ACL (Redis 6.x+)

ACL (Access Control List) replaces the single AUTH password with **named users**, each with its own password and set of command/key permissions. This is the recommended approach for Redis 6.x+.

**Default user**: Every cluster has a `default` user. On new clusters, disable it or set a strong password to prevent unauthenticated access.

**Creating ACL users via ElastiCache User/User Group:**

```bash
# Create a read-only user for the cache layer
aws elasticache create-user \
  --user-id cache-reader \
  --user-name cache-reader \
  --engine redis \
  --passwords "CacheReaderPass123!" \
  --access-string "on ~* &* -@all +@read"

# Create a write user for the application backend
aws elasticache create-user \
  --user-id app-writer \
  --user-name app-writer \
  --engine redis \
  --passwords "AppWriterPass456!" \
  --access-string "on ~session:* &* -@all +@read +@write +@string +@hash"

# Create a user group and add both users
aws elasticache create-user-group \
  --user-group-id my-app-users \
  --engine redis \
  --user-ids cache-reader app-writer default

# Associate the user group with the cluster
aws elasticache modify-replication-group \
  --replication-group-id secure-cluster \
  --user-group-ids-to-add my-app-users \
  --apply-immediately
```

**ACL access string syntax:**

| Component | Meaning |
|---|---|
| `on` / `off` | Enable or disable the user |
| `~*` | Allow access to all keys |
| `~session:*` | Allow access only to keys matching `session:*` |
| `&*` | Allow access to all Pub/Sub channels |
| `-@all` | Remove all command permissions as a base |
| `+@read` | Add all read commands |
| `+@write` | Add all write commands |
| `+@string +@hash` | Add only String and Hash commands |
| `nopass` | Allow connection without a password (not recommended for production) |

```python
# Connecting with ACL user credentials
r = redis.Redis(
    host='...', port=6379, ssl=True,
    username='app-writer',
    password='AppWriterPass456!'
)
```

---

## 4. Security — Network and Encryption

### Security Groups

ElastiCache nodes must be placed in a VPC. Create a dedicated security group for the cluster and allow inbound Redis traffic **only from application security groups** — never from `0.0.0.0/0`.

```bash
# Create dedicated SG for ElastiCache
aws ec2 create-security-group \
  --group-name elasticache-sg \
  --description "ElastiCache Redis access" \
  --vpc-id vpc-xxxxxxxx

CACHE_SG_ID=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=elasticache-sg \
  --query "SecurityGroups[0].GroupId" --output text)

# Allow Redis port only from the application SG
aws ec2 authorize-security-group-ingress \
  --group-id $CACHE_SG_ID \
  --protocol tcp \
  --port 6379 \
  --source-group sg-app-security-group-id
```

Redis Cluster mode uses ports **6379** (data) and **6380** (bus). Both must be open between cluster nodes, but only 6379 needs to be accessible from applications.

### Encryption in Transit (TLS)

```bash
# Create cluster with TLS enabled
aws elasticache create-replication-group \
  --replication-group-id tls-cluster \
  --transit-encryption-enabled \
  ...
```

Once TLS is enabled, **it cannot be disabled** without recreating the cluster. Plan for TLS from day one.

```python
# Python client with TLS
import redis
import ssl

r = redis.Redis(
    host='tls-cluster.xxxxxx.ng.0001.use1.cache.amazonaws.com',
    port=6379,
    ssl=True,
    ssl_cert_reqs=ssl.CERT_REQUIRED,
    ssl_ca_certs='/etc/ssl/certs/ca-certificates.crt'
)
```

For cluster mode enabled clients:

```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    host='tls-cluster.xxxxxx.clustercfg.use1.cache.amazonaws.com',
    port=6379,
    ssl=True,
    ssl_cert_reqs=ssl.CERT_REQUIRED
)
```

### Encryption at Rest

```bash
aws elasticache create-replication-group \
  --replication-group-id encrypted-cluster \
  --at-rest-encryption-enabled \
  --kms-key-id alias/my-elasticache-key \
  ...
```

At-rest encryption encrypts data on disk (used for persistent RDB snapshots and AOF logs). It cannot be enabled on an existing cluster — enable it at creation time.

### KMS Key Policy for ElastiCache

```json
{
  "Sid": "AllowElastiCache",
  "Effect": "Allow",
  "Principal": {
    "Service": "elasticache.amazonaws.com"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey",
    "kms:CreateGrant"
  ],
  "Resource": "*"
}
```

---

## 5. CloudWatch Monitoring Reference

### Critical Alarms to Configure

| Metric | Namespace | Threshold | Action |
|---|---|---|---|
| `EngineCPUUtilization` | `AWS/ElastiCache` | > 80% | Scale up node type or add replicas |
| `DatabaseMemoryUsagePercentage` | `AWS/ElastiCache` | > 80% | Scale up; review eviction policy |
| `Evictions` | `AWS/ElastiCache` | > 0 (for critical data) | Memory is full; eviction happening |
| `CurrConnections` | `AWS/ElastiCache` | Approaching `maxclients` | Connection pool exhaustion |
| `ReplicationLag` | `AWS/ElastiCache` | > 10s | Replica falling behind; network or load issue |
| `CacheHits` + `CacheMisses` | `AWS/ElastiCache` | Hit rate < 80% | Cache effectiveness degraded |
| `NetworkBytesIn` / `NetworkBytesOut` | `AWS/ElastiCache` | Near node limit | Node bandwidth saturation |

### Creating a Memory Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "ElastiCache-HighMemory" \
  --metric-name DatabaseMemoryUsagePercentage \
  --namespace AWS/ElastiCache \
  --dimensions Name=ReplicationGroupId,Value=my-redis-cluster \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --statistic Average \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --treat-missing-data breaching
```

### Calculating Cache Hit Rate

```bash
# Hit rate = CacheHits / (CacheHits + CacheMisses)
# Use a CloudWatch Metric Math expression:

aws cloudwatch get-metric-data \
  --metric-data-queries '[
    {"Id": "hits", "MetricStat": {"Metric": {"Namespace": "AWS/ElastiCache",
      "MetricName": "CacheHits", "Dimensions": [{"Name":"ReplicationGroupId","Value":"my-cluster"}]},
      "Period": 300, "Stat": "Sum"}},
    {"Id": "misses", "MetricStat": {"Metric": {"Namespace": "AWS/ElastiCache",
      "MetricName": "CacheMisses", "Dimensions": [{"Name":"ReplicationGroupId","Value":"my-cluster"}]},
      "Period": 300, "Stat": "Sum"}},
    {"Id": "hitrate", "Expression": "hits / (hits + misses) * 100",
     "Label": "CacheHitRate%"}
  ]' \
  --start-time 2024-12-01T00:00:00Z \
  --end-time 2024-12-01T01:00:00Z
```

### Redis-Specific Metrics via `INFO`

CloudWatch gives cluster-level metrics. For per-command statistics, keyspace hit/miss details, and memory fragmentation, use the Redis `INFO` command directly:

```bash
redis-cli -h my-cluster-endpoint -p 6379 --tls -a $AUTH_TOKEN INFO all
```

Key sections to review:
- `INFO memory` — `used_memory`, `mem_fragmentation_ratio` (> 1.5 means fragmentation; > 0.8 means swap in use)
- `INFO stats` — `keyspace_hits`, `keyspace_misses`, `evicted_keys`, `expired_keys`
- `INFO replication` — `role`, `connected_slaves`, `master_repl_offset`, lag per slave
- `INFO clients` — `connected_clients`, `blocked_clients`

---

## 6. Migration from Self-Managed Redis

### Option 1: RDB Snapshot Import

If your self-managed Redis supports `BGSAVE` and you can tolerate a brief migration window:

```bash
# Step 1: On source Redis, create an RDB snapshot
redis-cli BGSAVE
# Wait for: redis-cli LASTSAVE to return a recent timestamp

# Step 2: Copy the dump.rdb file to S3
aws s3 cp /var/lib/redis/dump.rdb s3://my-migration-bucket/redis-snapshot.rdb

# Step 3: Create ElastiCache cluster from the snapshot
aws elasticache create-snapshot \
  --replication-group-id source-cluster \
  --snapshot-name imported-snapshot

# Or create a new cluster and restore:
aws elasticache create-replication-group \
  --replication-group-id new-prod-cluster \
  --snapshotting-cluster-id "" \
  --snapshot-name s3://my-migration-bucket/redis-snapshot.rdb \
  ...
```

Note: ElastiCache only accepts RDB snapshots uploaded to S3 in the same region. The snapshot must be in the RDB format for the same Redis engine version.

### Option 2: Online Migration with redis-riot

For live migration without downtime, `redis-riot` (Redis I/O Tooling) can replicate data from a source Redis to ElastiCache while the source is still serving traffic:

```bash
# Install redis-riot
wget https://github.com/redis-developer/redis-riot/releases/latest/download/redis-riot-standalone.jar

# Run live replication (reads from source, writes to target)
java -jar redis-riot-standalone.jar \
  replicate \
  --source-uri redis://source-redis:6379 \
  --target-uri rediss://new-cluster.xxxxxx.use1.cache.amazonaws.com:6379 \
  --target-password "$AUTH_TOKEN" \
  --mode live  # Use PSYNC-based live replication
```

`--mode live` uses Redis replication protocol (PSYNC) to continuously stream changes. You point your application at the new ElastiCache endpoint once replication lag reaches zero.

### Option 3: Parallel Write with Gradual Cutover

For critical production migrations with zero data-loss tolerance:

1. **Dual-write phase**: Application writes to both old Redis and new ElastiCache.
2. **Warm-up phase**: Wait for the new cluster's cache to warm up (hit rate stabilizes).
3. **Read cutover**: Switch reads to ElastiCache.
4. **Write cutover**: Remove dual-write; write only to ElastiCache.
5. **Decommission**: Shut down old Redis after monitoring for a few days.

```python
def cache_set(key, value, ttl=300):
    try:
        old_redis.setex(key, ttl, value)  # Write to old
    except Exception:
        pass  # Don't fail on old Redis errors during migration
    new_elasticache.setex(key, ttl, value)  # Write to new (primary)

def cache_get(key):
    # Phase: read from new; fall back to old if miss
    value = new_elasticache.get(key)
    if value is None:
        value = old_redis.get(key)
        if value:
            # Backfill new cluster
            new_elasticache.setex(key, 300, value)
    return value
```

---

## 7. Migration from Memcached to Redis

Memcached and Redis use different protocols, data models, and client libraries. There is no direct migration path — keys cannot be transferred because Memcached has no persistence or export capability.

### Migration Strategy

The only practical approach is **gradual key migration via TTL expiry**:

1. Deploy new ElastiCache Redis cluster alongside existing Memcached cluster.
2. Update application to use a wrapper that writes to Redis and falls back to Memcached on miss.
3. As Memcached keys expire naturally (based on their TTLs), they are populated into Redis on the next cache miss.
4. Once all active keys have migrated (typically 1-2× the longest TTL period), decommission Memcached.

```python
class MigrationCache:
    def __init__(self, redis_client, memcached_client):
        self.redis = redis_client
        self.memcached = memcached_client

    def get(self, key):
        value = self.redis.get(key)
        if value is not None:
            return value
        # Fall back to Memcached
        value = self.memcached.get(key)
        if value is not None:
            # Migrate to Redis
            self.redis.setex(key, 3600, value)
        return value

    def set(self, key, value, ttl=3600):
        self.redis.setex(key, ttl, value)
        # Optionally write to Memcached during transition for rollback safety
        try:
            self.memcached.set(key, value, expire=ttl)
        except Exception:
            pass

    def delete(self, key):
        self.redis.delete(key)
        self.memcached.delete(key)
```

### Key Differences to Account For During Migration

| Aspect | Memcached | Redis |
|---|---|---|
| Client library | `pylibmc`, `python-memcached` | `redis-py`, `redis-py-cluster` |
| Key encoding | ASCII string | Binary-safe string |
| Max value size | 1 MB | 512 MB |
| Data types | String only | All Redis types |
| Expiry granularity | Seconds | Seconds or milliseconds |
| Cluster endpoint | `--configuration-endpoint` | `--cluster-configuration-endpoint` |

---

## 8. Maintenance and Upgrades

### Maintenance Window

ElastiCache applies patches and minor version upgrades during a weekly maintenance window. Configure it to a low-traffic period:

```bash
aws elasticache modify-replication-group \
  --replication-group-id my-cluster \
  --preferred-maintenance-window "sun:02:00-sun:03:00" \
  --apply-immediately
```

### Engine Version Upgrades

Minor version upgrades happen automatically within the maintenance window. Major version upgrades (e.g., Redis 6.x → Redis 7.x) require a manual action:

```bash
aws elasticache modify-replication-group \
  --replication-group-id my-cluster \
  --engine-version "7.0" \
  --apply-immediately  # Or let it apply in next maintenance window
```

With Multi-AZ enabled, ElastiCache performs rolling upgrades: replicas are upgraded one at a time, then the primary is failed over to an upgraded replica. Downtime is limited to the failover window (~30 seconds).

### Snapshot-Based Backup Schedule

```bash
# Enable automatic snapshots with 7-day retention
aws elasticache modify-replication-group \
  --replication-group-id my-cluster \
  --snapshot-retention-limit 7 \
  --snapshot-window "03:00-04:00" \
  --apply-immediately
```

Snapshots are stored in S3 (managed by ElastiCache) and can be used to create new clusters or restore to a point in time. Snapshot storage is charged at **$0.085 per GB-month** (us-east-1).

---

## Key Takeaways

- Multi-AZ automatic failover requires at least one read replica. Failover typically completes in 30–60 seconds; applications must use DNS endpoints and retry logic.
- Global Datastore provides active-passive cross-region replication for DR and geo-distributed reads. Cross-region writes are not supported — only the primary cluster accepts writes.
- Use Redis ACL (Redis 6.x+) instead of AUTH for per-user command and key-level permissions. Disable or restrict the `default` user.
- Enable TLS from day one — it cannot be added to an existing cluster. At-rest encryption is also create-time only.
- For migration from self-managed Redis, use RDB snapshot import for small datasets, or `redis-riot` live replication for zero-downtime migrations of larger datasets.
- Migrating from Memcached to Redis has no direct path — use gradual TTL-expiry migration with a dual-layer cache wrapper.
