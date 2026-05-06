# Part 2: Data Modeling & Access Pattern Design

---

## Table of Contents

1. [The Access-Pattern-First Methodology](#1-the-access-pattern-first-methodology)
2. [Partition Key Design Principles](#2-partition-key-design-principles)
3. [Sort Key Patterns](#3-sort-key-patterns)
4. [Single-Table Design](#4-single-table-design)
5. [Modeling One-to-Many Relationships](#5-modeling-one-to-many-relationships)
6. [Modeling Many-to-Many Relationships](#6-modeling-many-to-many-relationships)
7. [Hierarchical Data Patterns](#7-hierarchical-data-patterns)
8. [Avoiding Hot Partitions](#8-avoiding-hot-partitions)
9. [Overloading Keys: Generic PK and SK Names](#9-overloading-keys-generic-pk-and-sk-names)
10. [Access Pattern Planning Template](#10-access-pattern-planning-template)

---

## 1. The Access-Pattern-First Methodology

DynamoDB requires that you know your application's data access patterns **before** designing the table. This is the inverse of relational database design, where you model entities, add relationships, and write queries later.

**The DynamoDB design process:**

1. List every data access operation your application performs (reads and writes)
2. For each operation, identify: what keys are available, what needs to be returned, and how results should be ordered
3. Design the primary key (PK + SK) to serve the most critical and frequent operations directly
4. Design GSIs for the remaining access patterns that cannot be served by the primary key

**Example for an e-commerce application:**

| # | Access Pattern | Keys Available | Sort/Filter |
|---|---|---|---|
| 1 | Get user by user ID | `user_id` | None |
| 2 | Get all orders for a user | `user_id` | By date (newest first) |
| 3 | Get order by order ID | `order_id` | None |
| 4 | Get all items in an order | `order_id` | None |
| 5 | Get product by product ID | `product_id` | None |
| 6 | Get all pending orders | Status = PENDING | By creation date |

This list drives every table design decision. Patterns 1–5 can be served by a composite key design. Pattern 6 requires a Global Secondary Index.

---

## 2. Partition Key Design Principles

The Partition Key (PK) determines which storage partition an item lives on. Good PK design is critical for:
- Even data distribution across partitions
- Avoiding hot partitions (covered in Section 8)
- Efficient query performance

### Rule 1: High Cardinality

The PK must have many distinct values to distribute traffic evenly. Low-cardinality values concentrate traffic on few partitions.

| Bad PK (Low Cardinality) | Better PK (High Cardinality) |
|---|---|
| `Status` ("PENDING", "SHIPPED") | `UserID` (UUID per user) |
| `Country` (200 countries) | `DeviceID` (UUID per device) |
| `Year` (4 values for a decade) | `SessionID` (UUID per session) |
| `Department` (10 departments) | `OrderID` (UUID per order) |

### Rule 2: Uniform Access Distribution

Even with high cardinality, if 80% of all reads target 1% of your items (celebrity problem), you have a hot partition regardless. Use write sharding for these cases (see Section 8).

### Rule 3: Predictable Values

The PK should be a value your application always has available at query time without needing to scan or derive it. If you cannot supply the PK, you cannot query efficiently.

---

## 3. Sort Key Patterns

The Sort Key (SK) adds a second dimension to the primary key. It enables:
- Range queries within a partition
- Hierarchical storage (parent → children)
- Ordering of items within a partition

DynamoDB Sort Key condition expressions:

| Expression | Example | Result |
|---|---|---|
| `=` | `SK = "ORDER#123"` | Exact match |
| `<`, `<=`, `>`, `>=` | `SK > "2024-01-01"` | Comparisons |
| `begins_with(SK, val)` | `begins_with(SK, "ORDER#")` | Prefix filter |
| `between val1 and val2` | `SK BETWEEN "2024-01" AND "2024-03"` | Range |

### Sort Key as a Prefix Hierarchy

Using prefixes in the SK value creates a natural hierarchy:

```
PK: "USER#alice"     SK: "PROFILE"           → User profile item
PK: "USER#alice"     SK: "ORDER#2024-01-10"  → Order 1
PK: "USER#alice"     SK: "ORDER#2024-02-20"  → Order 2
PK: "USER#alice"     SK: "ADDRESS#home"      → Home address
PK: "USER#alice"     SK: "ADDRESS#work"      → Work address
```

Query to get all orders for alice:
```bash
aws dynamodb query \
  --table-name AppTable \
  --key-condition-expression "PK = :pk AND begins_with(SK, :sk_prefix)" \
  --expression-attribute-values '{
    ":pk": {"S": "USER#alice"},
    ":sk_prefix": {"S": "ORDER#"}
  }' \
  --scan-index-forward false \
  --region us-east-1
```

`--scan-index-forward false` returns items in descending Sort Key order (newest first, since dates are lexicographically ordered).

### Sort Key for Time-Based Data

For time series data, use ISO 8601 timestamps as Sort Keys. ISO 8601 timestamps are lexicographically sortable:

```
"2024-01-01T00:00:00Z"  < "2024-01-15T10:30:00Z"  < "2024-12-31T23:59:59Z"
```

This enables efficient range queries:
```bash
# Get events in the last 24 hours
aws dynamodb query \
  --table-name DeviceEvents \
  --key-condition-expression "DeviceID = :did AND EventTime BETWEEN :start AND :end" \
  --expression-attribute-values '{
    ":did": {"S": "device-001"},
    ":start": {"S": "2024-01-15T00:00:00Z"},
    ":end": {"S": "2024-01-15T23:59:59Z"}
  }' \
  --region us-east-1
```

---

## 4. Single-Table Design

Single-table design is the practice of storing multiple entity types (Users, Orders, Products, etc.) in a single DynamoDB table, differentiated by key prefixes. It is the dominant pattern for production DynamoDB applications.

### Why Single-Table Design

In SQL, you normalize data into separate tables and JOIN at query time. In DynamoDB, JOINs do not exist. If your application needs data from multiple entities in one response, those entities must be retrievable in a single `Query` operation. Putting related data in the same table with a shared Partition Key enables this.

**Multi-table approach (problematic in DynamoDB):**
```
Table: Users    (PK: UserID)
Table: Orders   (PK: OrderID)
Table: Products (PK: ProductID)

# To show a user's order history, you need:
# 1. GetItem on Users table
# 2. Query on Orders table
# 3. Multiple GetItem on Products table
# = 3+ separate API calls
```

**Single-table approach:**
```
Table: AppTable  (PK: PK, SK: SK)

# To show a user's order history:
# Query PK="USER#alice" → returns user profile + all orders in one call
# = 1 API call
```

### Single-Table Naming Conventions

Use a consistent prefix convention for all entity types:

| Entity | PK | SK | Data |
|---|---|---|---|
| User profile | `USER#<id>` | `PROFILE` | name, email, created_at |
| User's order | `USER#<id>` | `ORDER#<date>#<order_id>` | status, total |
| Order items | `ORDER#<id>` | `ITEM#<product_id>` | qty, price |
| Product | `PRODUCT#<id>` | `DETAILS` | name, price, category |

```bash
# Write a user profile
aws dynamodb put-item \
  --table-name AppTable \
  --item '{
    "PK": {"S": "USER#alice"},
    "SK": {"S": "PROFILE"},
    "EntityType": {"S": "USER"},
    "Email": {"S": "alice@example.com"},
    "DisplayName": {"S": "Alice Smith"},
    "CreatedAt": {"S": "2024-01-10T09:00:00Z"}
  }' \
  --region us-east-1

# Write an order belonging to alice
aws dynamodb put-item \
  --table-name AppTable \
  --item '{
    "PK": {"S": "USER#alice"},
    "SK": {"S": "ORDER#2024-01-15#ord-001"},
    "EntityType": {"S": "ORDER"},
    "OrderID": {"S": "ord-001"},
    "Status": {"S": "PENDING"},
    "TotalAmount": {"N": "89.99"}
  }' \
  --region us-east-1

# Query: get alice's profile AND all her orders in one call
aws dynamodb query \
  --table-name AppTable \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk": {"S": "USER#alice"}}' \
  --scan-index-forward false \
  --region us-east-1
```

The query returns all items under `PK="USER#alice"` — the profile item and all order items — in a single API call.

### When Multi-Table is Acceptable

Single-table design adds modeling complexity. Multi-table is acceptable when:
- Entity access patterns are fully independent (different services query different tables)
- You are building a microservice where each service owns its table
- The access patterns do not require combining data from multiple entity types

---

## 5. Modeling One-to-Many Relationships

A one-to-many relationship (one User → many Orders) is the most common pattern in DynamoDB. The standard approach uses a composite PK where the parent entity is the Partition Key.

### Pattern: Parent PK, Type-Prefixed SK

```
User has many Orders:

PK = "USER#<user_id>"
SK = "PROFILE"                      → user attributes
SK = "ORDER#<datetime>#<order_id>"  → each order (sortable by date)

# Read the user profile and recent orders together:
Query: PK = "USER#alice" AND SK begins_with "ORDER#2024"
```

### Modeling Related Items Together

When you need to retrieve an order and all its line items together:

```
PK = "ORDER#<order_id>"   SK = "METADATA"         → order header
PK = "ORDER#<order_id>"   SK = "ITEM#<product_id>" → each line item

# Get an order and all items in one query:
Query: PK = "ORDER#ord-001"
# Returns: METADATA item + all ITEM# items
```

```python
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('AppTable')

# Get order with all its items
response = table.query(
    KeyConditionExpression=Key('PK').eq('ORDER#ord-001')
)

order_data = {}
items = []
for record in response['Items']:
    if record['SK'] == 'METADATA':
        order_data = record
    elif record['SK'].startswith('ITEM#'):
        items.append(record)
```

---

## 6. Modeling Many-to-Many Relationships

Many-to-many (e.g., Students → Courses, Users → Groups) requires an **adjacency list** pattern.

### Adjacency List Pattern

Store the relationship items alongside entity items in the same table. Each relationship is represented by a record in both directions.

```
PK = "STUDENT#s-001"   SK = "PROFILE"         → student attributes
PK = "STUDENT#s-001"   SK = "ENROLL#c-math"   → enrolled in math course
PK = "STUDENT#s-001"   SK = "ENROLL#c-phys"   → enrolled in physics course

PK = "COURSE#c-math"   SK = "DETAILS"         → course attributes
PK = "COURSE#c-math"   SK = "ENROLL#s-001"    → student s-001 enrolled
PK = "COURSE#c-math"   SK = "ENROLL#s-002"    → student s-002 enrolled
```

Queries:
- "What courses is student s-001 enrolled in?" → Query `PK="STUDENT#s-001" AND SK begins_with "ENROLL#"`
- "Who is enrolled in course c-math?" → Query `PK="COURSE#c-math" AND SK begins_with "ENROLL#"`

Each `ENROLL#` item in either direction contains the IDs needed to fetch the full entity from the other side.

---

## 7. Hierarchical Data Patterns

For deeply nested hierarchies (Country → Region → City → Store), use the Sort Key to represent depth levels.

### Pattern: Composite Sort Key with Separators

```
PK = "GEO"   SK = "US"                        → Country: United States
PK = "GEO"   SK = "US#CA"                     → State: California
PK = "GEO"   SK = "US#CA#SF"                  → City: San Francisco
PK = "GEO"   SK = "US#CA#SF#STORE-001"        → Store 001 in SF
PK = "GEO"   SK = "US#CA#LA"                  → City: Los Angeles
PK = "GEO"   SK = "US#CA#LA#STORE-002"        → Store 002 in LA
```

Queries:
- All US data: `PK = "GEO" AND SK begins_with "US"`
- All California data: `PK = "GEO" AND SK begins_with "US#CA"`
- All San Francisco stores: `PK = "GEO" AND SK begins_with "US#CA#SF#STORE"`

This pattern scales to arbitrary depth without additional API calls.

---

## 8. Avoiding Hot Partitions

### The Hot Partition Problem

When all traffic concentrates on a single Partition Key value, that partition becomes a bottleneck. The partition's throughput limits (3,000 RCU, 1,000 WCU) apply regardless of how much total capacity the table has.

**Scenario:** You have a social media app. A celebrity posts something, and 50,000 users try to like it simultaneously. All writes go to `PK="POST#celebrity-post-id"` — a hot partition.

### Solution 1: Write Sharding

Append a random suffix to the Partition Key to distribute writes across multiple partitions.

```python
import random

def write_post_interaction(post_id: str, user_id: str, action: str):
    shard = random.randint(0, 9)  # 10 shards
    sharded_pk = f"POST#{post_id}#SHARD{shard}"
    
    table.put_item(Item={
        'PK': sharded_pk,
        'SK': f"USER#{user_id}",
        'Action': action,
        'Timestamp': datetime.utcnow().isoformat()
    })

def get_post_interaction_count(post_id: str) -> int:
    # Must query all shards to get total count
    total = 0
    for shard in range(10):
        response = table.query(
            KeyConditionExpression=Key('PK').eq(f'POST#{post_id}#SHARD{shard}')
        )
        total += response['Count']
    return total
```

**Trade-off:** Reading requires querying all shards and aggregating results. Use this only for write-heavy hot items where write throughput is more critical than read simplicity.

### Solution 2: Calculated Sharding

For time-series data with a known key space, calculate a deterministic shard:

```python
import hashlib

def get_shard(key: str, num_shards: int = 10) -> int:
    return int(hashlib.md5(key.encode()).hexdigest(), 16) % num_shards

def write_event(user_id: str, event_data: dict):
    shard = get_shard(user_id)
    pk = f"EVENTS#SHARD{shard}"
    sk = f"{datetime.utcnow().isoformat()}#{user_id}"
    
    table.put_item(Item={
        'PK': pk,
        'SK': sk,
        **event_data
    })
```

This distributes the same user's events to the same shard deterministically, which helps with some read patterns.

---

## 9. Overloading Keys: Generic PK and SK Names

A common convention in single-table design is to name the primary key attributes generically (`PK` and `SK` instead of `UserID` and `OrderDate`). This allows the same table to store many different entity types without implying that the keys always mean the same thing.

```bash
# Create table with generic key names
aws dynamodb create-table \
  --table-name AppTable \
  --attribute-definitions \
      AttributeName=PK,AttributeType=S \
      AttributeName=SK,AttributeType=S \
  --key-schema \
      AttributeName=PK,KeyType=HASH \
      AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Each entity type decides how to use PK and SK:
- User: `PK = "USER#alice"`, `SK = "PROFILE"`
- Order: `PK = "USER#alice"`, `SK = "ORDER#2024-01-15"`
- Product: `PK = "PRODUCT#p-001"`, `SK = "DETAILS"`

The `EntityType` attribute (a plain non-key attribute) documents what type of entity each item represents:
```json
{
  "PK": "USER#alice",
  "SK": "PROFILE",
  "EntityType": "USER",
  "Email": "alice@example.com"
}
```

---

## 10. Access Pattern Planning Template

Use this template before creating any DynamoDB table.

```
Table Name: _______________
Primary Use Case: _______________

Entities Stored:
  1. _______________
  2. _______________

Access Patterns:
┌────┬─────────────────────────────┬──────────────────┬─────────────────┬──────────────────────┐
│ #  │ Access Pattern              │ Keys Available   │ Sort/Filter     │ Served By            │
├────┼─────────────────────────────┼──────────────────┼─────────────────┼──────────────────────┤
│ 1  │ Get [entity] by [id]        │ entity_id        │ None            │ Base table (GetItem) │
│ 2  │ Get all [children] of [par] │ parent_id        │ By date         │ Base table (Query)   │
│ 3  │ Get [entity] by [alt attr]  │ email            │ None            │ GSI on email         │
│ 4  │ Get all [status=X] items    │ status           │ By created_at   │ GSI or sparse index  │
└────┴─────────────────────────────┴──────────────────┴─────────────────┴──────────────────────┘

Table Key Design:
  PK: _______________ (format: PREFIX#value)
  SK: _______________ (format: PREFIX#value or timestamp)

GSIs Needed:
  GSI 1: PK=___, SK=___, Projection=___, Covers patterns: ___
  GSI 2: PK=___, SK=___, Projection=___, Covers patterns: ___

Estimated Item Count: _______________
Estimated Item Size: _______________
Estimated Traffic: ___ reads/sec, ___ writes/sec
Capacity Mode: Provisioned / On-Demand
```

---

**Next:** Part 3 covers Capacity Planning, Auto Scaling, and Pricing — including how to size provisioned capacity, configure auto scaling, interpret CloudWatch metrics, and optimize costs at scale.
