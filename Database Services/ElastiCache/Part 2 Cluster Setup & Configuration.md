# Part 2: Cluster Setup & Configuration

---

## Table of Contents

1. [Planning a Cluster](#1-planning-a-cluster)
2. [Creating a Redis Cluster — Console Walkthrough](#2-creating-a-redis-cluster--console-walkthrough)
3. [Creating a Redis Cluster via CLI](#3-creating-a-redis-cluster-via-cli)
4. [Subnet Groups](#4-subnet-groups)
5. [Parameter Groups and Key Settings](#5-parameter-groups-and-key-settings)
6. [Security Groups and VPC Access](#6-security-groups-and-vpc-access)
7. [Authentication and Authorization](#7-authentication-and-authorization)
8. [Encryption (In Transit and At Rest)](#8-encryption-in-transit-and-at-rest)
9. [Connecting to an ElastiCache Cluster](#9-connecting-to-an-elasticache-cluster)
10. [Scaling a Cluster](#10-scaling-a-cluster)
11. [Monitoring Endpoints and Cluster Health](#11-monitoring-endpoints-and-cluster-health)

---

## 1. Planning a Cluster

Before creating a cluster, answer these questions:

| Decision | Options | Guidance |
|---|---|---|
| **Engine** | Redis, Memcached | Default to Redis unless you have a specific Memcached requirement |
| **Cluster mode** | Disabled, Enabled | Disabled for <100 GB datasets; Enabled for horizontal scale |
| **Multi-AZ** | Yes, No | Always Yes for production |
| **Number of replicas** | 0–5 | 1–2 for production; 0 for dev/test only |
| **Node type** | cache.t4g, r6g, m6g, r6gd | t-series for dev; r-series for production |
| **Encryption** | In-transit, At-rest | Both enabled for production |
| **AUTH** | Redis AUTH password or ACL | Always enabled for production |
| **VPC** | Same VPC as your application | Required — ElastiCache is not publicly accessible by default |

### Estimating Node Size

Your node must fit the **working set** — the data that is frequently accessed. You do not need to fit all possible data in cache.

```
Working set estimate:
  Peak concurrent sessions: 100,000
  Average session size: 500 bytes
  Total: 100,000 × 500 = 50 MB → cache.t4g.micro is sufficient

  For product catalog:
  10,000 products × 2 KB average = 20 MB → easily fits on t4g.micro

  For gaming leaderboard:
  1,000,000 players × 50 bytes (sorted set entry) = 50 MB
```

Add 25–30% overhead for Redis metadata and key names.

---

## 2. Creating a Redis Cluster — Console Walkthrough

```
AWS Console → ElastiCache → Redis clusters → Create Redis cluster

Step 1: Cluster settings
  Cluster creation method: Configure and create a new cluster
  Cluster mode: Disabled (or Enabled for sharding)
  Cluster info:
    Name: prod-cache
    Description: [optional]
    Engine version: 7.x (latest stable recommended)
    Port: 6379 (default)
    Parameter group: default.redis7 (or custom group)
    Node type: cache.r6g.large

Step 2: Connectivity
  Subnet groups: [select or create]
  Availability zone placement: Distribute replicas (recommended)
  Number of replicas: 2 (1 primary + 2 replicas = 3 nodes total)
  Multi-AZ: Enabled (required if replicas > 0)

Step 3: Advanced settings
  Security groups: [your security group]
  Encryption at rest: Enabled
  Encryption in transit: Enabled
  AUTH: Enabled → set a strong token

Step 4: Backup
  Automatic backups: Enabled
  Retention period: 7 days
  Backup window: 02:00 UTC (off-peak for your timezone)

Step 5: Maintenance
  Maintenance window: Tuesday 03:00–04:00 UTC

Create → cluster takes 5–10 minutes to become available
```

---

## 3. Creating a Redis Cluster via CLI

### Create Subnet Group

```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name prod-cache-subnet-group \
  --cache-subnet-group-description "Production cache subnet group" \
  --subnet-ids subnet-abc123 subnet-def456 subnet-ghi789 \
  --region us-east-1
```

### Create Parameter Group (Custom)

```bash
# Create a custom parameter group based on Redis 7 family
aws elasticache create-cache-parameter-group \
  --cache-parameter-group-name prod-redis7-params \
  --cache-parameter-group-family redis7 \
  --description "Production Redis 7 parameters" \
  --region us-east-1

# Set custom parameter values
aws elasticache modify-cache-parameter-group \
  --cache-parameter-group-name prod-redis7-params \
  --parameter-name-values \
      ParameterName=maxmemory-policy,ParameterValue=allkeys-lru \
      ParameterName=timeout,ParameterValue=300 \
      ParameterName=notify-keyspace-events,ParameterValue="" \
  --region us-east-1
```

### Create the Replication Group (Cluster Mode Disabled)

```bash
aws elasticache create-replication-group \
  --replication-group-id prod-cache \
  --replication-group-description "Production cache cluster" \
  --engine redis \
  --engine-version 7.0.7 \
  --cache-node-type cache.r6g.large \
  --num-cache-clusters 3 \
  --cache-subnet-group-name prod-cache-subnet-group \
  --security-group-ids sg-abc12345 \
  --cache-parameter-group-name prod-redis7-params \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "YourStrongPassword123!" \
  --snapshot-retention-limit 7 \
  --preferred-maintenance-window "tue:03:00-tue:04:00" \
  --region us-east-1
```

### Create a Cluster Mode Enabled Setup

```bash
aws elasticache create-replication-group \
  --replication-group-id prod-cache-cluster \
  --replication-group-description "Production sharded cache" \
  --engine redis \
  --engine-version 7.0.7 \
  --cache-node-type cache.r6g.large \
  --num-node-groups 3 \
  --replicas-per-node-group 1 \
  --cache-subnet-group-name prod-cache-subnet-group \
  --security-group-ids sg-abc12345 \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "YourStrongPassword123!" \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --region us-east-1
```

`--num-node-groups 3` creates 3 shards. `--replicas-per-node-group 1` creates 1 replica per shard (3 primaries + 3 replicas = 6 nodes total).

### Check Cluster Status

```bash
aws elasticache describe-replication-groups \
  --replication-group-id prod-cache \
  --query "ReplicationGroups[0].{Status:Status,Endpoints:NodeGroups[0].PrimaryEndpoint,ReadEndpoint:NodeGroups[0].ReaderEndpoint}" \
  --region us-east-1
```

---

## 4. Subnet Groups

A subnet group specifies which VPC subnets ElastiCache can use when placing nodes. For Multi-AZ deployments, include subnets from at least 2 Availability Zones.

### Requirements

- All subnets must be in the same VPC
- Include subnets from at least 2 AZs for Multi-AZ failover
- Use **private** subnets — ElastiCache should not be accessible from the public internet

### Creating a Subnet Group

```bash
# List available subnets in your VPC
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=vpc-abc123 \
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}" \
  --output table \
  --region us-east-1

# Create subnet group with subnets from 3 AZs
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name prod-cache-subnets \
  --cache-subnet-group-description "Private subnets for cache" \
  --subnet-ids \
      subnet-az1 \
      subnet-az2 \
      subnet-az3 \
  --region us-east-1
```

---

## 5. Parameter Groups and Key Settings

Parameter groups control runtime behavior. The critical parameters to understand:

### maxmemory-policy

Controls what happens when the cache reaches its memory limit. This is the most important parameter.

| Policy | Behavior | Use When |
|---|---|---|
| `noeviction` | Refuse new writes with error | When you cannot afford to lose any cached data |
| `allkeys-lru` | Evict least recently used keys (all keys eligible) | **Most common for caches** — ensures the working set stays in memory |
| `allkeys-lfu` | Evict least frequently used (all keys eligible) | Better than LRU when some keys are accessed in long bursts |
| `volatile-lru` | Evict LRU among keys with TTL set | When mix of persistent and cached data in same Redis instance |
| `volatile-ttl` | Evict key with shortest remaining TTL | When freshness is the priority |
| `allkeys-random` | Evict random key | Rarely appropriate |

**Recommendation for ElastiCache caches:** Use `allkeys-lru`. It ensures the cache is always usable and the most recently accessed data stays in memory.

```bash
# Check current maxmemory-policy
redis-cli -h your-endpoint -p 6379 -a "yourpassword" --tls CONFIG GET maxmemory-policy

# Change via parameter group (not directly via CONFIG SET — parameter group is the right way)
aws elasticache modify-cache-parameter-group \
  --cache-parameter-group-name prod-redis7-params \
  --parameter-name-values ParameterName=maxmemory-policy,ParameterValue=allkeys-lru \
  --region us-east-1
```

### timeout

Number of seconds before idle client connections are closed. Default is 0 (no timeout). Setting this prevents connection accumulation from crashed clients.

```
Recommended: timeout=300 (5 minutes)
```

### notify-keyspace-events

Enables keyspace notifications — Redis publishes events when keys are modified. Default is `""` (disabled).

Enable only when needed (e.g., for cache invalidation triggers) because it adds CPU overhead.

```
For key expiry notifications: notify-keyspace-events = Ex
For all events: notify-keyspace-events = KEA
```

### tcp-keepalive

Time in seconds between TCP keepalive probes. Helps detect dead connections. Default is 300.

### reserved-memory-percent

Percentage of node memory reserved for ElastiCache internal use and OS overhead. Default is 25 for cluster mode enabled, 0 otherwise.

For cluster mode enabled, set this to `25` to leave headroom for slot migration during resharding.

---

## 6. Security Groups and VPC Access

ElastiCache nodes are deployed inside your VPC. They have no public IP addresses by default and are not accessible from the internet.

### Security Group Setup

Create a dedicated security group for your ElastiCache cluster. Only allow inbound traffic from your application security groups.

```bash
# Create security group for ElastiCache
aws ec2 create-security-group \
  --group-name prod-cache-sg \
  --description "ElastiCache Redis security group" \
  --vpc-id vpc-abc123 \
  --region us-east-1

# Allow inbound Redis traffic only from application security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-cache12345 \
  --protocol tcp \
  --port 6379 \
  --source-group sg-app67890 \
  --region us-east-1
```

**Principle:** ElastiCache security group accepts TCP 6379 only from the application security group. No other sources. Do not allow 0.0.0.0/0.

### VPC Endpoint (Optional)

For applications running outside the VPC (e.g., Lambda in a different VPC, on-premises via Direct Connect), configure VPC peering or use a VPC endpoint. ElastiCache does not support AWS PrivateLink directly — use VPC peering or transit gateway.

---

## 7. Authentication and Authorization

### Redis AUTH Token

The `--auth-token` parameter sets a password that clients must provide with every connection.

```bash
# Connect with AUTH token (in-transit encryption must be enabled)
redis-cli -h your-endpoint.cache.amazonaws.com -p 6379 \
  -a "YourAuthToken" --tls

# Rotate the auth token (without downtime)
aws elasticache modify-replication-group \
  --replication-group-id prod-cache \
  --auth-token "NewAuthToken456!" \
  --auth-token-update-strategy ROTATE \
  --apply-immediately \
  --region us-east-1
```

`ROTATE` strategy: both the old and new tokens are valid for a transition period. Once you update all clients, switch to `SET` to make only the new token valid.

### Role-Based Access Control (ACL) — Redis 6.x+

ElastiCache for Redis 6.x+ supports Redis ACL, which provides per-user, per-command, per-key access control.

```bash
# Create a user group with a user policy
aws elasticache create-user \
  --user-id app-user \
  --user-name app-user \
  --engine redis \
  --passwords "SecurePassword123!" \
  --access-string "on ~session:* +GET +SET +DEL +EXPIRE" \
  --region us-east-1

# Create a user group
aws elasticache create-user-group \
  --user-group-id prod-cache-users \
  --engine redis \
  --user-ids default app-user \
  --region us-east-1

# Associate user group with replication group
aws elasticache modify-replication-group \
  --replication-group-id prod-cache \
  --user-group-ids-to-add prod-cache-users \
  --apply-immediately \
  --region us-east-1
```

Access string format: `on` (enabled) + `~keypattern:*` (allowed key patterns) + `+COMMAND` (allowed commands).

---

## 8. Encryption (In Transit and At Rest)

### Encryption In Transit (TLS)

Encrypts all data between your application and ElastiCache. Enabled at cluster creation time (cannot be changed later).

Client configuration:
```python
import redis

# Connect with TLS and AUTH
client = redis.Redis(
    host='prod-cache.abc123.ng.0001.use1.cache.amazonaws.com',
    port=6379,
    password='YourAuthToken',
    ssl=True,
    ssl_cert_reqs='required'
)
```

### Encryption At Rest

Encrypts data stored on disk (RDB snapshots, AOF logs). Enabled at cluster creation time. Uses AWS-managed KMS keys by default. You can specify a customer-managed KMS key.

```bash
# Create cluster with customer-managed KMS key
aws elasticache create-replication-group \
  --replication-group-id secure-cache \
  --at-rest-encryption-enabled \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123-... \
  [other parameters]
```

**Note:** At-rest encryption adds a small performance overhead (<5%) due to encryption/decryption at the storage layer.

---

## 9. Connecting to an ElastiCache Cluster

ElastiCache nodes are only accessible from within the same VPC. You cannot connect to them from your local machine unless you set up a bastion host or VPN.

### From an EC2 Instance or Lambda in the Same VPC

```python
import redis

# Cluster mode disabled — use primary and reader endpoints
write_client = redis.Redis(
    host='prod-cache.abc123.ng.0001.use1.cache.amazonaws.com',
    port=6379,
    password='YourAuthToken',
    ssl=True,
    decode_responses=True
)

read_client = redis.Redis(
    host='prod-cache.abc123.ng.0002.use1.cache.amazonaws.com',
    port=6379,
    password='YourAuthToken',
    ssl=True,
    decode_responses=True
)
```

### Cluster Mode Enabled — Use Cluster Client

```python
from rediscluster import RedisCluster

startup_nodes = [
    {'host': 'prod-cache.abc123.clustercfg.use1.cache.amazonaws.com', 'port': 6379}
]

client = RedisCluster(
    startup_nodes=startup_nodes,
    password='YourAuthToken',
    ssl=True,
    decode_responses=True,
    skip_full_coverage_check=True
)
```

### Testing Connectivity via Bastion

```bash
# SSH tunnel through bastion host
ssh -L 6380:prod-cache.abc123.ng.0001.use1.cache.amazonaws.com:6379 \
    -N ec2-user@bastion-ip

# Then connect locally via tunnel
redis-cli -h 127.0.0.1 -p 6380 -a "YourAuthToken" --tls
```

---

## 10. Scaling a Cluster

### Vertical Scaling (Change Node Type)

```bash
aws elasticache modify-replication-group \
  --replication-group-id prod-cache \
  --cache-node-type cache.r6g.xlarge \
  --apply-immediately \
  --region us-east-1
```

AWS performs the resize by:
1. Creating a new node of the target type
2. Replicating data to the new node
3. Promoting the new node and decommissioning the old one

With Multi-AZ enabled, this happens without downtime (replicas are resized one at a time, failover handles primary replacement).

### Add Replicas (Cluster Mode Disabled)

```bash
aws elasticache increase-replica-count \
  --replication-group-id prod-cache \
  --new-replica-count 3 \
  --apply-immediately \
  --region us-east-1
```

### Horizontal Scaling (Cluster Mode Enabled — Add/Remove Shards)

```bash
# Add 2 shards to existing 3-shard cluster
aws elasticache modify-replication-group-shard-configuration \
  --replication-group-id prod-cache-cluster \
  --node-group-count 5 \
  --apply-immediately \
  --region us-east-1

# Remove 2 shards (must specify which slots to redistribute)
aws elasticache modify-replication-group-shard-configuration \
  --replication-group-id prod-cache-cluster \
  --node-group-count 3 \
  --node-groups-to-remove NodeGroupId1 NodeGroupId2 \
  --apply-immediately \
  --region us-east-1
```

Resharding is online — the cluster remains available during slot redistribution.

---

## 11. Monitoring Endpoints and Cluster Health

### Get Cluster Endpoints

```bash
# Cluster mode disabled: primary + reader endpoints
aws elasticache describe-replication-groups \
  --replication-group-id prod-cache \
  --query "ReplicationGroups[0].NodeGroups[0].{Primary:PrimaryEndpoint,Reader:ReaderEndpoint}" \
  --region us-east-1

# Cluster mode enabled: configuration endpoint
aws elasticache describe-replication-groups \
  --replication-group-id prod-cache-cluster \
  --query "ReplicationGroups[0].ConfigurationEndpoint" \
  --region us-east-1
```

### Check Node Health

```bash
aws elasticache describe-cache-clusters \
  --show-cache-node-info \
  --query "CacheClusters[?ReplicationGroupId=='prod-cache'].{ClusterID:CacheClusterId,Status:CacheClusterStatus,Node:CacheNodes[0].{Status:CacheNodeStatus,Endpoint:Endpoint.Address}}" \
  --region us-east-1
```

### Key CloudWatch Metrics to Monitor

| Metric | What to Watch For |
|---|---|
| `CacheHits` | Should be high (>85% hit rate) |
| `CacheMisses` | High misses indicate cold cache or poor key design |
| `CurrConnections` | Spike indicates connection leak |
| `Evictions` | Non-zero means memory is full — scale up or improve TTL policy |
| `DatabaseMemoryUsagePercentage` | Alert at 75%, scale before hitting 90% |
| `EngineCPUUtilization` | Alert at 80%; Redis is single-threaded |
| `ReplicationLag` | High lag on replicas indicates replication is falling behind |
| `CurrItems` | Total key count |
| `NetworkBytesIn/Out` | Bandwidth monitoring |

---

**Next:** Part 3 covers Caching Patterns — Cache-Aside, Write-Through, Write-Behind, and Read-Through — with code examples, TTL strategies, cache stampede prevention, and invalidation approaches.
