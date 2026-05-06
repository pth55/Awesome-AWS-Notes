# Part 4: Redis Data Structures & Use Cases

---

## Table of Contents

1. [Data Structure Overview](#1-data-structure-overview)
2. [Strings](#2-strings)
3. [Hashes](#3-hashes)
4. [Lists](#4-lists)
5. [Sets](#5-sets)
6. [Sorted Sets](#6-sorted-sets)
7. [Bitmaps](#7-bitmaps)
8. [HyperLogLog](#8-hyperloglog)
9. [Pub/Sub](#9-pubsub)
10. [Streams](#10-streams)
11. [Geospatial Index](#11-geospatial-index)
12. [Data Structure Selection Guide](#12-data-structure-selection-guide)

---

## 1. Data Structure Overview

Redis is not a plain key-value store — it supports multiple native data structures, each with specialized commands and time complexity guarantees. Choosing the right structure reduces memory usage and improves performance compared to storing everything as serialized JSON strings.

| Structure | Analogous to | Max Size | Best For |
|---|---|---|---|
| String | Scalar / byte array | 512 MB | Counters, cached values, flags |
| Hash | JSON object / hash map | 2^32 fields | User sessions, object attributes |
| List | Doubly linked list | 2^32 elements | Queues, activity feeds, logs |
| Set | Unordered unique collection | 2^32 members | Unique visitors, tags, permissions |
| Sorted Set | Score-ranked set | 2^32 members | Leaderboards, scheduled tasks, rate limits |
| Bitmap | Bit array | 512 MB (2^32 bits) | Daily active user tracking, feature flags |
| HyperLogLog | Probabilistic cardinality | Fixed 12 KB | Unique count estimation |
| Stream | Append-only log | Unlimited (bounded with MAXLEN) | Event logs, activity feeds, audit trails |
| Geospatial | Coordinate index | 2^32 members | Location-based queries |

---

## 2. Strings

The simplest and most versatile type. A String in Redis can hold text, numbers, or binary data up to 512 MB.

### Key Commands

| Command | Time Complexity | Description |
|---|---|---|
| `SET key value [EX seconds]` | O(1) | Set a value with optional TTL |
| `GET key` | O(1) | Get a value |
| `MSET k1 v1 k2 v2` | O(N) | Set multiple keys atomically |
| `MGET k1 k2 k3` | O(N) | Get multiple values |
| `INCR key` | O(1) | Atomic increment by 1 |
| `INCRBY key amount` | O(1) | Atomic increment by N |
| `SETNX key value` | O(1) | Set only if not exists (used for locks) |
| `GETSET key value` | O(1) | Set and return previous value |
| `GETDEL key` | O(1) | Get and delete atomically |

### Use Case 1: Caching Serialized Objects

```python
import json
import redis

cache = redis.Redis(host='...', ssl=True, password='...', decode_responses=True)

def get_cached_product(product_id: str) -> dict:
    return json.loads(cache.get(f'product:{product_id}') or 'null')

def cache_product(product: dict, ttl: int = 300):
    cache.setex(f'product:{product["id"]}', ttl, json.dumps(product))
```

### Use Case 2: Distributed Locking (SETNX)

```python
import uuid
import time

def acquire_lock(resource: str, ttl: int = 10) -> str | None:
    lock_id = str(uuid.uuid4())
    acquired = cache.set(f'lock:{resource}', lock_id, ex=ttl, nx=True)
    return lock_id if acquired else None

def release_lock(resource: str, lock_id: str) -> bool:
    # Lua script ensures atomic check-and-delete (prevents releasing another process's lock)
    lua_script = """
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
    """
    result = cache.eval(lua_script, 1, f'lock:{resource}', lock_id)
    return bool(result)

# Usage
lock_id = acquire_lock('inventory:update', ttl=30)
if lock_id:
    try:
        # Critical section
        update_inventory()
    finally:
        release_lock('inventory:update', lock_id)
```

### Use Case 3: Feature Flags

```python
def is_feature_enabled(feature: str) -> bool:
    value = cache.get(f'feature:{feature}')
    if value is None:
        # Load from configuration store
        enabled = config_store.get(f'feature.{feature}', False)
        cache.setex(f'feature:{feature}', 300, str(int(enabled)))
        return enabled
    return bool(int(value))
```

### Use Case 4: Rate Limiting

```python
def is_rate_limited(user_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f'ratelimit:{user_id}'
    current = cache.incr(key)
    if current == 1:
        cache.expire(key, window)  # Start the window on first request
    return current > limit

# Sliding window rate limit (more accurate)
def is_rate_limited_sliding(user_id: str, limit: int = 100, window: int = 60) -> bool:
    now = int(time.time() * 1000)  # milliseconds
    key = f'ratelimit:sliding:{user_id}'
    
    pipeline = cache.pipeline()
    pipeline.zremrangebyscore(key, 0, now - window * 1000)  # Remove old entries
    pipeline.zadd(key, {str(now): now})                    # Add current timestamp
    pipeline.zcard(key)                                     # Count entries in window
    pipeline.expire(key, window + 1)
    _, _, count, _ = pipeline.execute()
    
    return count > limit
```

---

## 3. Hashes

A Hash stores a map of field-value pairs within a single key. Instead of storing an entire object as a serialized string, you store individual fields. This allows reading or updating a single field without deserializing the entire object.

### Key Commands

| Command | Time Complexity | Description |
|---|---|---|
| `HSET key field value` | O(1) | Set one or more fields |
| `HGET key field` | O(1) | Get one field |
| `HMGET key f1 f2 f3` | O(N) | Get multiple fields |
| `HGETALL key` | O(N) | Get all fields and values |
| `HDEL key field` | O(1) | Delete a field |
| `HINCRBY key field amount` | O(1) | Atomic increment a numeric field |
| `HEXISTS key field` | O(1) | Check if field exists |
| `HKEYS key / HVALS key` | O(N) | Get all keys or values |

### Use Case 1: User Session Object

```python
def create_session(session_id: str, user_id: str, user_data: dict, ttl: int = 1800):
    key = f'session:{session_id}'
    pipeline = cache.pipeline()
    pipeline.hset(key, mapping={
        'user_id': user_id,
        'email': user_data['email'],
        'role': user_data['role'],
        'created_at': str(int(time.time())),
        'last_active': str(int(time.time()))
    })
    pipeline.expire(key, ttl)
    pipeline.execute()

def get_session(session_id: str) -> dict:
    return cache.hgetall(f'session:{session_id}')

def update_last_active(session_id: str, ttl: int = 1800):
    key = f'session:{session_id}'
    pipeline = cache.pipeline()
    pipeline.hset(key, 'last_active', str(int(time.time())))
    pipeline.expire(key, ttl)  # Extend TTL on activity (rolling session)
    pipeline.execute()

def get_session_field(session_id: str, field: str) -> str:
    return cache.hget(f'session:{session_id}', field)
```

**Why Hash instead of String?** With String, you serialize the entire session to JSON, then deserialize on every read. With Hash, you can do `HGET session:xyz last_active` without loading the entire session object. Under high request volume this matters.

### Use Case 2: User Preferences

```python
def update_preference(user_id: str, preference: str, value: str):
    cache.hset(f'prefs:{user_id}', preference, value)
    cache.expire(f'prefs:{user_id}', 3600)

def get_all_preferences(user_id: str) -> dict:
    return cache.hgetall(f'prefs:{user_id}')

def get_preference(user_id: str, preference: str, default=None) -> str:
    value = cache.hget(f'prefs:{user_id}', preference)
    return value if value is not None else default
```

### Use Case 3: Aggregated Counters per Entity

```python
def increment_product_stats(product_id: str, views: int = 1, purchases: int = 0):
    key = f'product_stats:{product_id}'
    pipeline = cache.pipeline()
    if views:
        pipeline.hincrby(key, 'views', views)
    if purchases:
        pipeline.hincrby(key, 'purchases', purchases)
    pipeline.expire(key, 86400)  # Keep for 24 hours
    pipeline.execute()

def get_product_stats(product_id: str) -> dict:
    stats = cache.hgetall(f'product_stats:{product_id}')
    return {k: int(v) for k, v in stats.items()} if stats else {}
```

---

## 4. Lists

A List is a doubly linked list of string values. It supports push/pop from both ends (head and tail) in O(1) time. Lists are ordered by insertion order.

### Key Commands

| Command | Time Complexity | Description |
|---|---|---|
| `LPUSH key value` | O(1) | Push to head |
| `RPUSH key value` | O(1) | Push to tail |
| `LPOP key` | O(1) | Pop from head |
| `RPOP key` | O(1) | Pop from tail |
| `BLPOP key timeout` | O(1) | Blocking pop from head (waits for data) |
| `BRPOP key timeout` | O(1) | Blocking pop from tail |
| `LRANGE key start stop` | O(S+N) | Get a range (0-indexed, -1=last) |
| `LLEN key` | O(1) | Length of list |
| `LREM key count value` | O(N) | Remove occurrences of value |
| `LTRIM key start stop` | O(N) | Keep only a range, discard rest |

### Use Case 1: Task Queue

```python
# Producer — push jobs to queue
def enqueue_job(job: dict):
    cache.rpush('queue:email', json.dumps(job))

# Consumer — blocking pop (waits up to 5 seconds)
def process_jobs():
    while True:
        result = cache.blpop('queue:email', timeout=5)
        if result:
            _, job_data = result
            job = json.loads(job_data)
            send_email(job)
```

**Note:** For durable job queues where message loss is unacceptable, use SQS. Redis Lists are appropriate for best-effort queues where occasional message loss (on Redis restart without AOF) is acceptable.

### Use Case 2: Bounded Activity Feed

```python
MAX_FEED_ITEMS = 100

def add_to_feed(user_id: str, event: dict):
    key = f'feed:{user_id}'
    pipeline = cache.pipeline()
    pipeline.lpush(key, json.dumps(event))         # Push to front
    pipeline.ltrim(key, 0, MAX_FEED_ITEMS - 1)    # Keep only last N items
    pipeline.expire(key, 86400)
    pipeline.execute()

def get_feed(user_id: str, limit: int = 20) -> list:
    raw = cache.lrange(f'feed:{user_id}', 0, limit - 1)
    return [json.loads(item) for item in raw]
```

### Use Case 3: Recent Searches

```python
def add_recent_search(user_id: str, query: str):
    key = f'recent_searches:{user_id}'
    pipeline = cache.pipeline()
    pipeline.lrem(key, 0, query)      # Remove if already in list (deduplicate)
    pipeline.lpush(key, query)         # Push to front
    pipeline.ltrim(key, 0, 9)          # Keep only 10 most recent
    pipeline.expire(key, 7 * 86400)    # 7-day retention
    pipeline.execute()

def get_recent_searches(user_id: str) -> list:
    return cache.lrange(f'recent_searches:{user_id}', 0, -1)
```

---

## 5. Sets

A Set is an unordered collection of unique strings. Redis Sets support O(1) add, remove, and membership test, plus set operations (union, intersection, difference).

### Key Commands

| Command | Time Complexity | Description |
|---|---|---|
| `SADD key member` | O(1) | Add member |
| `SREM key member` | O(1) | Remove member |
| `SISMEMBER key member` | O(1) | Check membership |
| `SMEMBERS key` | O(N) | All members (use sparingly for large sets) |
| `SCARD key` | O(1) | Cardinality (size) |
| `SUNION k1 k2` | O(N) | Union of two sets |
| `SINTER k1 k2` | O(N×M) | Intersection |
| `SDIFF k1 k2` | O(N) | Difference |
| `SSCAN key cursor` | O(1) per call | Incremental scan |

### Use Case 1: User Tags and Permissions

```python
def add_permission(user_id: str, permission: str):
    key = f'permissions:{user_id}'
    cache.sadd(key, permission)
    cache.expire(key, 600)

def has_permission(user_id: str, permission: str) -> bool:
    return bool(cache.sismember(f'permissions:{user_id}', permission))

def get_all_permissions(user_id: str) -> set:
    return cache.smembers(f'permissions:{user_id}')
```

### Use Case 2: Unique Visitors per Day

```python
def track_unique_visitor(page_id: str, user_id: str) -> int:
    today = datetime.utcnow().strftime('%Y-%m-%d')
    key = f'unique_visitors:{page_id}:{today}'
    cache.sadd(key, user_id)
    cache.expire(key, 86400 * 2)  # Keep for 2 days
    return cache.scard(key)  # Return current unique count
```

**Note:** For large-scale unique counting where approximate results are acceptable, use HyperLogLog (Section 8) — it uses only 12 KB regardless of set size.

### Use Case 3: Common Friends / Shared Interests

```python
def get_mutual_interests(user_a: str, user_b: str) -> set:
    return cache.sinter(f'interests:{user_a}', f'interests:{user_b}')

def get_content_for_user(user_id: str, friend_ids: list) -> set:
    # Union all friends' liked articles
    keys = [f'liked_articles:{fid}' for fid in friend_ids]
    all_liked = cache.sunion(*keys)
    user_liked = cache.smembers(f'liked_articles:{user_id}')
    return all_liked - user_liked  # Articles friends liked but user hasn't seen
```

---

## 6. Sorted Sets

A Sorted Set (ZSet) associates a floating-point **score** with each member. Members are unique, scores can repeat. The set is always ordered by score (ascending). This makes it ideal for leaderboards, priority queues, and time-series indexed lookups.

### Key Commands

| Command | Time Complexity | Description |
|---|---|---|
| `ZADD key score member` | O(log N) | Add or update member with score |
| `ZINCRBY key increment member` | O(log N) | Increment score |
| `ZRANK key member` | O(log N) | Rank (0-based, ascending) |
| `ZREVRANK key member` | O(log N) | Rank (0-based, descending) |
| `ZSCORE key member` | O(1) | Get score |
| `ZRANGE key start stop [WITHSCORES]` | O(log N+M) | Range by rank (ascending) |
| `ZREVRANGE key start stop` | O(log N+M) | Range by rank (descending) |
| `ZRANGEBYSCORE key min max` | O(log N+M) | Range by score |
| `ZRANGEBYLEX key min max` | O(log N+M) | Range by lexicographic order (equal scores) |
| `ZREM key member` | O(log N) | Remove member |
| `ZREMRANGEBYSCORE key min max` | O(log N+M) | Remove by score range |
| `ZCARD key` | O(1) | Cardinality |

### Use Case 1: Game Leaderboard

```python
def submit_score(game_id: str, player_id: str, score: float):
    key = f'leaderboard:{game_id}'
    current = cache.zscore(key, player_id)
    if current is None or score > current:
        cache.zadd(key, {player_id: score})

def get_top_players(game_id: str, n: int = 10) -> list:
    results = cache.zrevrange(f'leaderboard:{game_id}', 0, n - 1, withscores=True)
    return [{'player': player, 'score': score, 'rank': i + 1}
            for i, (player, score) in enumerate(results)]

def get_player_rank(game_id: str, player_id: str) -> dict:
    rank = cache.zrevrank(f'leaderboard:{game_id}', player_id)
    score = cache.zscore(f'leaderboard:{game_id}', player_id)
    nearby = cache.zrevrange(
        f'leaderboard:{game_id}', 
        max(0, rank - 2), rank + 2,
        withscores=True
    )
    return {'rank': rank + 1 if rank is not None else None, 'score': score, 'nearby': nearby}
```

### Use Case 2: Sliding Window Rate Limiting (Precise)

```python
def is_rate_limited_precise(user_id: str, limit: int = 100, window_ms: int = 60000) -> bool:
    now = int(time.time() * 1000)
    key = f'ratelimit:precise:{user_id}'
    
    pipeline = cache.pipeline()
    # Remove requests outside the window
    pipeline.zremrangebyscore(key, 0, now - window_ms)
    # Add this request (score = timestamp, member = unique ID)
    pipeline.zadd(key, {f'{now}-{uuid.uuid4()}': now})
    # Count requests in window
    pipeline.zcard(key)
    pipeline.expire(key, window_ms // 1000 + 1)
    _, _, count, _ = pipeline.execute()
    
    return count > limit
```

### Use Case 3: Priority Queue / Delayed Jobs

```python
def schedule_job(job_id: str, job_data: dict, execute_at: float):
    # Score = Unix timestamp → jobs sorted by execution time
    cache.zadd('scheduled_jobs', {job_id: execute_at})
    cache.hset(f'job_data:{job_id}', mapping=job_data)

def get_due_jobs() -> list:
    now = time.time()
    due_jobs = cache.zrangebyscore('scheduled_jobs', 0, now)
    if due_jobs:
        pipeline = cache.pipeline()
        pipeline.zremrangebyscore('scheduled_jobs', 0, now)
        for job_id in due_jobs:
            pipeline.hgetall(f'job_data:{job_id}')
            pipeline.delete(f'job_data:{job_id}')
        pipeline.execute()
    return due_jobs
```

### Use Case 4: Autocomplete / Type-Ahead

Store all possible completions with score 0 and use lexicographic range queries.

```python
def add_autocomplete_entry(prefix_key: str, value: str):
    # Score 0 for all — use lexicographic ordering
    cache.zadd(f'autocomplete:{prefix_key}', {value: 0})

def get_completions(prefix_key: str, prefix: str, limit: int = 10) -> list:
    return cache.zrangebylex(
        f'autocomplete:{prefix_key}',
        f'[{prefix}',
        f'[{prefix}\xff',
        start=0, num=limit
    )
```

---

## 7. Bitmaps

A Bitmap is a String where you manipulate individual bits. Redis provides bit-level get/set commands. An array of 2^32 bits (512 MB max) can track 4 billion boolean states.

### Key Commands

| Command | Description |
|---|---|
| `SETBIT key offset value` | Set bit at offset to 0 or 1 |
| `GETBIT key offset` | Get bit value at offset |
| `BITCOUNT key [start end]` | Count set bits |
| `BITOP AND/OR/XOR dest k1 k2` | Bitwise operations between bitmaps |
| `BITPOS key bit` | Find first bit with given value |

### Use Case: Daily Active Users

Each user is assigned a unique integer ID (0-based). Their bit is set on the day they are active.

```python
def record_user_activity(user_id: int, date: str = None):
    if date is None:
        date = datetime.utcnow().strftime('%Y-%m-%d')
    cache.setbit(f'dau:{date}', user_id, 1)
    cache.expire(f'dau:{date}', 86400 * 30)  # 30-day retention

def get_dau(date: str) -> int:
    return cache.bitcount(f'dau:{date}')

def get_users_active_both_days(date1: str, date2: str) -> int:
    # Bitwise AND to find users active on both days
    dest_key = f'dau_intersect:{date1}_{date2}'
    cache.bitop('AND', dest_key, f'dau:{date1}', f'dau:{date2}')
    cache.expire(dest_key, 3600)
    return cache.bitcount(dest_key)
```

**Memory:** 1 million users = 1 million bits = 125 KB per day. 365 days = 45 MB. Very space-efficient.

---

## 8. HyperLogLog

HyperLogLog provides a probabilistic approximation of the cardinality (unique count) of a set. It uses only 12 KB of memory regardless of the number of elements, with ~0.81% error rate.

### Key Commands

| Command | Description |
|---|---|
| `PFADD key element` | Add element to HyperLogLog |
| `PFCOUNT key` | Get estimated cardinality |
| `PFMERGE dest k1 k2` | Merge multiple HyperLogLogs |

### Use Case: Unique Page Views at Scale

```python
def track_unique_page_view(page_id: str, visitor_id: str):
    date = datetime.utcnow().strftime('%Y-%m-%d')
    key = f'hll:unique_views:{page_id}:{date}'
    cache.pfadd(key, visitor_id)
    cache.expire(key, 86400 * 7)  # 7-day retention

def get_estimated_unique_views(page_id: str, date: str) -> int:
    return cache.pfcount(f'hll:unique_views:{page_id}:{date}')

def get_weekly_unique_views(page_id: str) -> int:
    # Merge 7 days of HLLs
    keys = [f'hll:unique_views:{page_id}:{(datetime.utcnow() - timedelta(days=i)).strftime("%Y-%m-%d")}' 
            for i in range(7)]
    dest = f'hll:weekly:{page_id}'
    cache.pfmerge(dest, *keys)
    cache.expire(dest, 3600)
    return cache.pfcount(dest)
```

**When to use HyperLogLog vs Set:** If you have millions of unique visitors and need an exact count, use a Set (but it consumes O(N) memory). If approximate (±0.81%) is acceptable, use HyperLogLog (fixed 12 KB).

---

## 9. Pub/Sub

Redis Pub/Sub enables real-time message broadcasting. Publishers send messages to a channel. Subscribers receive all messages published to channels they subscribe to. Messages are not persisted — if no subscriber is connected, the message is lost.

### Key Commands

| Command | Description |
|---|---|
| `PUBLISH channel message` | Publish a message to a channel |
| `SUBSCRIBE channel` | Subscribe to a channel |
| `PSUBSCRIBE pattern` | Subscribe using a glob pattern |
| `UNSUBSCRIBE channel` | Unsubscribe |

### Use Case: Real-Time Notifications

```python
# Publisher (in your API service)
def notify_order_update(order_id: str, status: str, user_id: str):
    event = json.dumps({
        'event': 'order_status_change',
        'order_id': order_id,
        'status': status,
        'timestamp': datetime.utcnow().isoformat()
    })
    cache.publish(f'user:{user_id}:notifications', event)
    cache.publish('order:status_changes', event)  # Broadcast to ops

# Subscriber (in your WebSocket or notification service)
def listen_for_user_notifications(user_id: str):
    pub_sub = cache.pubsub(ignore_subscribe_messages=True)
    pub_sub.subscribe(f'user:{user_id}:notifications')
    
    for message in pub_sub.listen():
        if message['type'] == 'message':
            event = json.loads(message['data'])
            send_websocket_message(user_id, event)
```

**Important:** Pub/Sub subscribers must be connected when a message is published. If the subscriber disconnects, messages are lost. For durable messaging, use SQS or Redis Streams.

---

## 10. Streams

Redis Streams (added in Redis 5.0) are append-only logs with consumer group support. Unlike Pub/Sub, messages are persisted and can be replayed. Consumer groups enable parallel processing with at-least-once delivery.

### Key Commands

| Command | Description |
|---|---|
| `XADD stream * field value` | Append entry (auto-ID) |
| `XREAD COUNT n STREAMS stream id` | Read entries since ID |
| `XREADGROUP GROUP g CONSUMER c COUNT n STREAMS s >` | Read undelivered messages for group |
| `XACK stream group id` | Acknowledge processed message |
| `XLEN stream` | Number of entries |
| `XRANGE stream - +` | Read all entries |
| `XTRIM stream MAXLEN n` | Cap stream length |

### Use Case: Event Log with Consumer Groups

```python
# Producer — append events to stream
def log_user_action(user_id: str, action: str, metadata: dict):
    cache.xadd('user_actions', {
        'user_id': user_id,
        'action': action,
        'metadata': json.dumps(metadata),
        'timestamp': datetime.utcnow().isoformat()
    })
    # Optionally cap the stream to prevent unbounded growth
    cache.xtrim('user_actions', maxlen=1000000, approximate=True)

# Consumer group setup (run once)
def setup_consumer_group():
    try:
        cache.xgroup_create('user_actions', 'analytics_workers', id='0', mkstream=True)
    except redis.ResponseError as e:
        if 'BUSYGROUP' not in str(e):
            raise

# Consumer — processes messages with at-least-once delivery
def process_events(worker_id: str):
    while True:
        messages = cache.xreadgroup(
            groupname='analytics_workers',
            consumername=f'worker:{worker_id}',
            streams={'user_actions': '>'},  # '>' = undelivered only
            count=10,
            block=1000  # Wait up to 1 second
        )
        
        for stream, entries in (messages or []):
            for entry_id, fields in entries:
                try:
                    process_analytics_event(fields)
                    cache.xack('user_actions', 'analytics_workers', entry_id)
                except Exception as e:
                    log_error(f'Failed to process {entry_id}: {e}')
                    # Message stays in pending list for retry
```

**Streams vs Pub/Sub:**
- **Streams:** Persistent, replayable, consumer groups, at-least-once delivery
- **Pub/Sub:** Non-persistent, fire-and-forget, broadcast to all subscribers

---

## 11. Geospatial Index

Redis Geospatial stores coordinates and supports distance queries, nearby searches, and bounding box queries.

### Key Commands

| Command | Description |
|---|---|
| `GEOADD key longitude latitude member` | Add location |
| `GEODIST key m1 m2 [unit]` | Distance between two members |
| `GEOPOS key member` | Get coordinates |
| `GEOSEARCH key FROMMEMBER m BYRADIUS r m` | Find members within radius |
| `GEORADIUSBYMEMBER key member r m` | Radius search (legacy) |

### Use Case: Find Nearby Stores

```python
# Populate store locations
def add_store(store_id: str, longitude: float, latitude: float):
    cache.geoadd('stores', longitude, latitude, store_id)

# Find stores within 5 km of a user
def find_nearby_stores(user_lon: float, user_lat: float, radius_km: float = 5) -> list:
    results = cache.geosearch(
        'stores',
        longitude=user_lon, latitude=user_lat,
        radius=radius_km, unit='km',
        withcoord=True, withdist=True,
        sort='ASC', count=10
    )
    return [{'store_id': r[0], 'distance_km': r[1], 'coords': r[2]}
            for r in results]

# Get distance between two stores
def get_store_distance(store_a: str, store_b: str) -> float:
    return cache.geodist('stores', store_a, store_b, unit='km')
```

---

## 12. Data Structure Selection Guide

| Requirement | Best Structure | Why |
|---|---|---|
| Cache a serialized object | String | Simple, fast |
| Store object fields independently | Hash | Update individual fields without full deserialize |
| FIFO job queue | List | O(1) LPUSH/RPOP |
| Bounded recent activity | List + LTRIM | Automatic size cap |
| Unique tag collection | Set | O(1) membership test |
| Unique visitor count (exact) | Set | Precise count |
| Unique visitor count (approximate, millions) | HyperLogLog | Fixed 12 KB regardless of size |
| Sorted ranking / leaderboard | Sorted Set | Score-based ordering |
| Rate limiting (fixed window) | String + INCR + EXPIRE | Simple and fast |
| Rate limiting (sliding window) | Sorted Set | Timestamp as score, prune old entries |
| Delayed/scheduled tasks | Sorted Set | Score = execute_at timestamp |
| Daily active user tracking | Bitmap | 1 bit per user per day |
| Real-time broadcast (fire and forget) | Pub/Sub | No persistence needed |
| Durable event log with replay | Stream | Persistent, consumer groups |
| Location-based queries | Geospatial | Native radius search |
| Distributed lock | String + SETNX | Atomic conditional set |
| Atomic multi-step operation | Lua script | Single round trip, atomic |

---

**Next:** Part 5 covers High Availability, Security, Monitoring, and Migration — including Multi-AZ failover behavior, Global Datastore, encryption details, CloudWatch alarms, and migration strategies from Memcached or self-managed Redis.
