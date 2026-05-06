# Part 1: ElastiCache Fundamentals

---

## Table of Contents

1. [What is Amazon ElastiCache](#1-what-is-amazon-elasticache)
2. [Redis vs Memcached — Architecture and Feature Comparison](#2-redis-vs-memcached--architecture-and-feature-comparison)
3. [ElastiCache Components](#3-elasticache-components)
4. [Redis Cluster Modes](#4-redis-cluster-modes)
5. [Common Use Cases](#5-common-use-cases)
6. [When to Use ElastiCache — and When Not To](#6-when-to-use-elasticache--and-when-not-to)
7. [Pricing Overview](#7-pricing-overview)
8. [Key Terminology](#8-key-terminology)

---

## 1. What is Amazon ElastiCache

**Amazon ElastiCache** is a fully managed in-memory data store and cache service. It deploys and operates Redis or Memcached clusters, handling hardware provisioning, software patching, monitoring, failure detection, and recovery automatically.

In-memory storage means all data is held in RAM, not on disk. This makes reads and writes orders of magnitude faster than any disk-based database — typically sub-millisecond latency. The trade-off is that RAM is volatile; data is lost on node failure unless persistence is configured (Redis only).

ElastiCache sits between your application layer and your primary database. Its primary role is to absorb high-frequency reads so the database does not receive unnecessary load.

**What ElastiCache manages:**
- Node provisioning and replacement
- Patching and version upgrades
- Replication and failover
- Automatic snapshot (Redis)
- CloudWatch metric integration
- VPC isolation and encryption

**What you manage:**
- Cache key design and data structure selection
- TTL (expiry) strategy
- Application-level cache invalidation logic
- Cluster sizing and scaling decisions

---

## 2. Redis vs Memcached — Architecture and Feature Comparison

Both engines are supported by ElastiCache. Choosing between them depends on the features your application requires.

### Side-by-Side Comparison

| Feature | Redis | Memcached |
|---|---|---|
| **Data structures** | Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps, HyperLogLog, Streams, Geospatial | String (key-value only) |
| **Persistence** | RDB snapshots + AOF (append-only file) | None — data lost on restart |
| **Replication** | Supported (primary + up to 5 replicas) | Not supported |
| **Multi-AZ failover** | Supported | Not supported |
| **Cluster mode (sharding)** | Supported (up to 500 shards) | Supported (simple multi-node) |
| **Pub/Sub messaging** | Supported | Not supported |
| **Lua scripting** | Supported | Not supported |
| **Transactions (MULTI/EXEC)** | Supported | Not supported |
| **Sorted Sets (leaderboards)** | Supported | Not supported |
| **Backup and restore** | Supported | Not supported |
| **Encryption at rest** | Supported | Supported |
| **Encryption in transit (TLS)** | Supported | Supported |
| **AUTH password** | Supported | Not supported |
| **RBAC (Redis ACL)** | Supported (Redis 6.x+) | Not supported |
| **Geospatial commands** | Supported | Not supported |
| **Thread model** | Single-threaded (for commands) | Multi-threaded |
| **Memory efficiency** | Slightly higher overhead per key | Slightly more memory-efficient for pure k-v |

### When to Choose Redis

Choose Redis when any of the following apply:
- You need **persistence** — data must survive node restarts (sessions, rate limits)
- You need **replication and Multi-AZ failover** — production workloads requiring high availability
- You need **advanced data structures** — sorted sets for leaderboards, lists for queues, hashes for user session objects
- You need **Pub/Sub** — real-time messaging between services
- You need **Lua scripting** for atomic multi-key operations
- You need **fine-grained access control** (Redis ACL via RBAC in ElastiCache for Redis 6.x+)

### When to Choose Memcached

Choose Memcached only when:
- Your use case is **pure key-value caching** with no need for data structures, persistence, or replication
- You need **multi-threaded performance** for CPU-bound workloads (Memcached scales better across CPU cores)
- You are migrating an existing Memcached application and want managed hosting

**Practical guidance:** For new projects, default to Redis. Memcached's simplicity advantage is minimal, and the lack of replication and persistence makes it unsuitable for most production workloads.

---

## 3. ElastiCache Components

### Node

A **node** is a single compute instance running Redis or Memcached. It has a specific amount of RAM (determined by the node type) that defines maximum data capacity.

Node types follow the naming convention `cache.<family>.<size>`, for example:
- `cache.t4g.micro` — 0.5 GB RAM, burstable (dev/test only)
- `cache.t4g.medium` — 1.37 GB RAM, burstable
- `cache.r6g.large` — 13.07 GB RAM, memory-optimized (production standard)
- `cache.r6g.2xlarge` — 52.82 GB RAM, memory-optimized
- `cache.r6gd.xlarge` — 26.04 GB RAM + NVMe SSD (enhanced I/O)

**Node selection rule:** Choose the smallest node that fits your expected working set with ~20% headroom for metadata, Redis overhead, and growth.

### Shard

A **shard** (Redis cluster mode) is a grouping of one primary node and up to 5 replica nodes. A shard holds a subset of the total keyspace (defined by a hash slot range). With cluster mode disabled, there is exactly one shard.

### Cluster

A **cluster** is the full deployment unit in ElastiCache:
- **Redis (cluster mode disabled):** One shard — one primary + up to 5 replicas
- **Redis (cluster mode enabled):** Up to 500 shards, each with primary + replicas
- **Memcached:** Multiple independent nodes (no replication)

### Replication Group

A **replication group** is the ElastiCache object that manages a Redis cluster, its replicas, and replication configuration. It provides a single endpoint that your application connects to.

### Subnet Group

A **subnet group** defines the VPC subnets in which ElastiCache will launch nodes. Use subnets from multiple Availability Zones to enable Multi-AZ failover.

### Parameter Group

A **parameter group** is a named set of runtime configuration settings applied to all nodes in a cluster (e.g., `maxmemory-policy`, `timeout`, `notify-keyspace-events`). Modifying a parameter group applies changes to all associated clusters.

---

## 4. Redis Cluster Modes

![ElastiCache Architecture](images/elasticache-architecture.svg)

### Cluster Mode Disabled

All data is stored across one shard. The primary node handles writes and reads (with strong consistency). Replica nodes handle eventually consistent reads and serve as standby for failover.

**Capacity:** Limited by the RAM of a single node. Maximum ~407 GB on `cache.r6g.12xlarge`.

**Endpoints:**
- **Primary endpoint:** All writes. Always points to the current primary node.
- **Reader endpoint:** Load-balanced reads across all replicas.
- **Node endpoint:** Direct connection to a specific node.

**Use when:**
- Dataset fits on a single node
- Simple caching with no advanced sharding required
- Dataset < 100 GB

### Cluster Mode Enabled

Data is partitioned across multiple shards using consistent hashing. Redis uses 16,384 **hash slots** distributed evenly across shards. Each key is mapped to a hash slot via `CRC16(key) mod 16384`.

**Capacity:** Up to 500 shards × up to ~407 GB per shard = ~200 TB maximum.

**Endpoint:** A single **configuration endpoint** that cluster-aware Redis clients use to discover the topology.

**Hash Tags:** To co-locate related keys on the same shard (required for multi-key commands), use hash tags in the key name. Only the portion within `{}` is hashed:

```
Keys: {user:alice}:session, {user:alice}:cart, {user:alice}:profile
All hash on "user:alice" → same shard
```

**Online Resharding:** Add or remove shards without downtime. Data moves automatically.

**Limitations with Cluster Mode Enabled:**
- `MULTI/EXEC` transactions only work when all keys are in the same hash slot
- Multi-key commands (`MGET`, `MSET`, `DEL` with multiple keys) require all keys to be in the same slot
- `SELECT` command (multiple databases) is not supported

**Use when:**
- Dataset exceeds single-node memory
- Very high write throughput requiring distribution across multiple primaries
- Data can be partitioned by user, tenant, or other natural key

---

## 5. Common Use Cases

### Session Store

User sessions (login state, shopping cart, preferences) are the classic ElastiCache use case. Sessions are short-lived key-value data accessed on every request.

```python
import redis
import json
import uuid

client = redis.Redis(
    host='my-cluster.abc123.ng.0001.use1.cache.amazonaws.com',
    port=6379,
    ssl=True
)

def create_session(user_id: str, data: dict) -> str:
    session_id = str(uuid.uuid4())
    client.setex(
        name=f'session:{session_id}',
        time=1800,  # 30 minutes
        value=json.dumps({'user_id': user_id, **data})
    )
    return session_id

def get_session(session_id: str) -> dict:
    data = client.get(f'session:{session_id}')
    return json.loads(data) if data else None

def refresh_session(session_id: str, ttl: int = 1800):
    client.expire(f'session:{session_id}', ttl)
```

### Database Query Caching

Cache the results of expensive database queries with a short TTL. On a cache miss, fetch from the database and populate the cache.

```python
def get_product(product_id: str) -> dict:
    cache_key = f'product:{product_id}'
    cached = client.get(cache_key)
    if cached:
        return json.loads(cached)
    product = db.execute('SELECT * FROM products WHERE id = %s', product_id)
    client.setex(cache_key, 300, json.dumps(product))
    return product
```

### Rate Limiting

Use Redis atomic increment and expiry to implement per-user rate limits.

```python
def check_rate_limit(user_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f'ratelimit:{user_id}'
    current = client.incr(key)
    if current == 1:
        client.expire(key, window)
    return current <= limit
```

### Leaderboards

Redis Sorted Sets maintain a score-ranked collection. Adding a score, retrieving the top-N, and getting a user's rank are all O(log N) operations.

```python
def update_score(game_id: str, user_id: str, score: float):
    client.zadd(f'leaderboard:{game_id}', {user_id: score})

def get_top_players(game_id: str, n: int = 10) -> list:
    return client.zrevrange(f'leaderboard:{game_id}', 0, n - 1, withscores=True)

def get_player_rank(game_id: str, user_id: str) -> int:
    rank = client.zrevrank(f'leaderboard:{game_id}', user_id)
    return rank + 1 if rank is not None else None
```

### Pub/Sub Messaging

Redis Pub/Sub enables real-time message broadcasting between application components without a dedicated message queue.

```python
# Publisher
def publish_event(channel: str, event: dict):
    client.publish(channel, json.dumps(event))

# Subscriber
def subscribe_to_events(channel: str):
    pubsub = client.pubsub()
    pubsub.subscribe(channel)
    for message in pubsub.listen():
        if message['type'] == 'message':
            event = json.loads(message['data'])
            handle_event(event)
```

### Real-Time Analytics Counters

Use Redis atomic operations to maintain counters without database write contention.

```python
from datetime import datetime

def track_page_view(page_id: str):
    today = datetime.utcnow().strftime('%Y-%m-%d')
    client.incr(f'views:{page_id}:{today}')
    client.expire(f'views:{page_id}:{today}', 86400 * 30)  # Keep 30 days

def get_daily_views(page_id: str, date: str) -> int:
    return int(client.get(f'views:{page_id}:{date}') or 0)
```

---

## 6. When to Use ElastiCache — and When Not To

### Use ElastiCache When

- **Repeated reads of the same data:** User profiles, product catalogs, configuration values accessed on every request
- **Sub-millisecond latency required:** Leaderboards, auto-complete, real-time dashboards
- **Database read load is unsustainable:** CPU, connections, or IOPS on RDS/Aurora are at capacity
- **Session state must be shared across multiple application instances:** Multiple EC2 or ECS tasks need access to the same session data
- **Rate limiting or feature flags:** High-frequency, low-overhead per-request checks

### Do Not Use ElastiCache As

- **Primary data store:** ElastiCache is not a database. Data lost on node failure (without persistence configured). Always keep a durable primary database.
- **Replacement for proper query optimization:** If your database queries are slow, fix the queries and indexes first. Cache is not a substitute for a missing index.
- **Message queue for durable work:** Redis Streams and Pub/Sub can handle this, but for durable job processing use SQS. Redis messages can be lost.
- **Full-text search engine:** Use OpenSearch (Elasticsearch) for text search.

---

## 7. Pricing Overview

Pricing is per node-hour (regardless of whether the node is being used). All prices for `us-east-1`.

### Node Pricing (Redis)

| Node Type | RAM | vCPU | On-Demand Price |
|---|---|---|---|
| `cache.t4g.micro` | 0.5 GB | 2 | $0.016/hr (~$11.52/mo) |
| `cache.t4g.small` | 1.37 GB | 2 | $0.032/hr (~$23.04/mo) |
| `cache.t4g.medium` | 3.09 GB | 2 | $0.064/hr (~$46.08/mo) |
| `cache.r6g.large` | 13.07 GB | 2 | $0.151/hr (~$108.72/mo) |
| `cache.r6g.xlarge` | 26.24 GB | 4 | $0.302/hr (~$217.44/mo) |
| `cache.r6g.2xlarge` | 52.82 GB | 8 | $0.604/hr (~$434.88/mo) |

### Multi-AZ and Replicas

Each replica node is priced the same as a primary node. A 3-node cluster (1 primary + 2 replicas using `cache.r6g.large`) costs:
```
3 × $0.151/hr = $0.453/hr = $326.16/month
```

### Backup Storage

- Backup storage up to 100% of total cluster storage: Free
- Additional backup storage: $0.085 per GB/month

### Reserved Nodes (Discount)

| Term | Payment | Discount vs On-Demand |
|---|---|---|
| 1 year | No upfront | ~37% |
| 1 year | All upfront | ~42% |
| 3 years | No upfront | ~49% |
| 3 years | All upfront | ~55% |

### Cost Estimate Example

**Production setup:** 1 primary + 2 replicas using `cache.r6g.large`, Multi-AZ, with backups

```
Nodes: 3 × $0.151/hr × 730 hr/month = $330.69/month
Backup: Assume 10 GB total data → first 10 GB free
Total: ~$331/month
```

---

## 8. Key Terminology

| Term | Definition |
|---|---|
| **Node** | Single compute instance running Redis or Memcached. RAM determines max data capacity. |
| **Shard** | In Redis cluster mode: one primary + up to 5 replicas holding a hash slot range. |
| **Replication Group** | Managed object for a Redis cluster with its primary, replicas, and endpoints. |
| **Cluster Mode Disabled** | Single-shard Redis setup. All data on one node. Simple and supports all Redis commands. |
| **Cluster Mode Enabled** | Multi-shard Redis setup. Data partitioned by hash slot. Scales horizontally. |
| **Hash Slot** | 1 of 16,384 buckets used to assign keys to shards. `CRC16(key) mod 16384`. |
| **Hash Tag** | Notation `{key_part}` that forces co-location of related keys on the same shard. |
| **Primary Endpoint** | Endpoint for writes. Always routes to current primary node. |
| **Reader Endpoint** | Endpoint for reads. Load-balanced across replica nodes. |
| **Configuration Endpoint** | Cluster mode endpoint. Client discovers shard topology from it. |
| **Subnet Group** | VPC subnets where ElastiCache nodes are placed. |
| **Parameter Group** | Runtime configuration applied to a cluster (eviction policy, timeout, etc.). |
| **TTL** | Time to Live. Keys automatically expire after the configured duration. |
| **Eviction Policy** | What ElastiCache does when memory is full (e.g., LRU eviction, no eviction). |
| **AOF** | Append-Only File. Redis persistence mode — writes every command to disk. |
| **RDB Snapshot** | Point-in-time Redis persistence. Less durable than AOF but faster restores. |
| **Multi-AZ** | Automatic failover — replica in a different AZ is promoted if primary fails. |
| **maxmemory-policy** | Parameter that defines eviction behavior: `allkeys-lru`, `volatile-lru`, `noeviction`, etc. |

---

**Next:** Part 2 covers Cluster Setup and Configuration — creating ElastiCache clusters, configuring parameter groups, security groups, VPC isolation, AUTH, and encryption.
