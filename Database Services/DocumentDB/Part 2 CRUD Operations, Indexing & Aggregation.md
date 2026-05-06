# Part 2 — DocumentDB CRUD Operations, Indexing & Aggregation

## Table of Contents

1. [Creating a Cluster and Connecting](#1-creating-a-cluster-and-connecting)
2. [CRUD Operations](#2-crud-operations)
3. [Query Operators](#3-query-operators)
4. [Indexing](#4-indexing)
5. [Aggregation Pipeline](#5-aggregation-pipeline)
6. [Update Operators](#6-update-operators)
7. [Bulk Operations](#7-bulk-operations)
8. [Explain Plans and Query Optimization](#8-explain-plans-and-query-optimization)

---

## 1. Creating a Cluster and Connecting

### Create a Cluster via CLI

```bash
# Create subnet group
aws docdb create-db-subnet-group \
  --db-subnet-group-name docdb-subnet-group \
  --db-subnet-group-description "DocumentDB subnet group" \
  --subnet-ids subnet-aaa111 subnet-bbb222 subnet-ccc333

# Create the cluster
aws docdb create-db-cluster \
  --db-cluster-identifier my-docdb-cluster \
  --engine docdb \
  --engine-version 4.0.0 \
  --master-username admin \
  --master-user-password "MyPassword123!" \
  --db-subnet-group-name docdb-subnet-group \
  --vpc-security-group-ids sg-xxxxxxxx \
  --storage-encrypted \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:05:00-sun:06:00"

# Add the primary instance
aws docdb create-db-instance \
  --db-instance-identifier my-docdb-primary \
  --db-cluster-identifier my-docdb-cluster \
  --db-instance-class db.r6g.large \
  --engine docdb

# Add a read replica
aws docdb create-db-instance \
  --db-instance-identifier my-docdb-replica-1 \
  --db-cluster-identifier my-docdb-cluster \
  --db-instance-class db.r6g.large \
  --engine docdb \
  --availability-zone us-east-1b \
  --promotion-tier 1  # Failover priority (0=highest)
```

### Connection String Format

```
mongodb://username:password@cluster-endpoint:27017/?tls=true&tlsCAFile=rds-combined-ca-bundle.pem&retryWrites=false
```

For applications using connection strings (Node.js, etc.):

```javascript
const { MongoClient } = require('mongodb');

const uri = 'mongodb://admin:password@mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com:27017/?tls=true&tlsCAFile=rds-combined-ca-bundle.pem&retryWrites=false';

const client = new MongoClient(uri);
await client.connect();
const db = client.db('myapp');
```

---

## 2. CRUD Operations

### Insert

```python
from pymongo import MongoClient
import certifi

client = MongoClient(
    'mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com',
    port=27017,
    username='admin', password='password',
    tls=True, tlsCAFile=certifi.where(),
    retryWrites=False
)
db = client['ecommerce']
orders = db['orders']

# Insert one
result = orders.insert_one({
    '_id': 'ORD-001',
    'customerId': 'CUST-123',
    'status': 'pending',
    'items': [
        {'productId': 'PROD-A', 'qty': 2, 'price': 29.99},
        {'productId': 'PROD-B', 'qty': 1, 'price': 49.99}
    ],
    'total': 109.97,
    'createdAt': '2024-12-01T10:00:00Z'
})
print(result.inserted_id)  # ORD-001

# Insert many
results = orders.insert_many([
    {'_id': 'ORD-002', 'customerId': 'CUST-456', 'status': 'pending', 'total': 25.00},
    {'_id': 'ORD-003', 'customerId': 'CUST-789', 'status': 'shipped', 'total': 75.00}
])
print(results.inserted_ids)  # ['ORD-002', 'ORD-003']
```

### Read

```python
# Find one
order = orders.find_one({'_id': 'ORD-001'})

# Find many with filter
pending = list(orders.find({'status': 'pending'}))

# Find with projection (include only specific fields)
result = orders.find_one(
    {'_id': 'ORD-001'},
    projection={'customerId': 1, 'status': 1, '_id': 0}
)
# Returns: {'customerId': 'CUST-123', 'status': 'pending'}

# Find with sort and limit
recent_orders = list(
    orders.find({'status': 'shipped'})
          .sort('createdAt', -1)  # -1 = descending
          .limit(10)
)

# Count documents
count = orders.count_documents({'status': 'pending'})

# Check existence
exists = orders.count_documents({'_id': 'ORD-001'}, limit=1) > 0
```

### Update

```python
# Update one — set a field
orders.update_one(
    {'_id': 'ORD-001'},
    {'$set': {'status': 'shipped', 'shippedAt': '2024-12-02T08:00:00Z'}}
)

# Update many
orders.update_many(
    {'status': 'pending', 'total': {'$lt': 20}},
    {'$set': {'status': 'cancelled'}}
)

# Upsert — insert if not found
orders.update_one(
    {'_id': 'ORD-099'},
    {'$set': {'status': 'pending', 'total': 50.00}},
    upsert=True
)

# Find and update (returns original document by default)
old_doc = orders.find_one_and_update(
    {'_id': 'ORD-001'},
    {'$set': {'status': 'delivered'}},
    return_document=False  # True = return updated document
)
```

### Delete

```python
# Delete one
result = orders.delete_one({'_id': 'ORD-001'})
print(result.deleted_count)  # 1

# Delete many
result = orders.delete_many({'status': 'cancelled'})
print(result.deleted_count)

# Find and delete (returns the deleted document)
deleted = orders.find_one_and_delete({'_id': 'ORD-002'})
```

---

## 3. Query Operators

### Comparison Operators

```python
# $eq, $ne, $gt, $gte, $lt, $lte
orders.find({'total': {'$gte': 50, '$lte': 200}})
orders.find({'status': {'$ne': 'cancelled'}})

# $in, $nin — match against a list
orders.find({'status': {'$in': ['pending', 'processing']}})
orders.find({'customerId': {'$nin': ['CUST-001', 'CUST-002']}})
```

### Logical Operators

```python
# $and (implicit with multiple conditions)
orders.find({'status': 'pending', 'total': {'$gt': 100}})

# $or
orders.find({'$or': [
    {'status': 'pending'},
    {'status': 'processing'}
]})

# $not
orders.find({'total': {'$not': {'$gt': 1000}}})

# $nor — matches documents that fail all conditions
orders.find({'$nor': [
    {'status': 'cancelled'},
    {'total': {'$lt': 10}}
]})
```

### Array Operators

```python
# $elemMatch — match documents where at least one array element meets all conditions
orders.find({'items': {'$elemMatch': {'productId': 'PROD-A', 'qty': {'$gte': 2}}}})

# $size — match arrays of a specific length
orders.find({'items': {'$size': 3}})

# $all — array contains all specified values
products.find({'tags': {'$all': ['electronics', 'sale']}})

# Query on array element by index
orders.find({'items.0.productId': 'PROD-A'})  # First item's productId
```

### Element Operators

```python
# $exists — check if field exists
orders.find({'shippedAt': {'$exists': True}})
orders.find({'couponCode': {'$exists': False}})

# $type — filter by BSON type
orders.find({'total': {'$type': 'double'}})
```

### Embedded Document Queries

```python
# Exact match on embedded document (field order matters — use dot notation instead)
orders.find({'shippingAddress.city': 'Seattle'})
orders.find({'shippingAddress.state': 'WA', 'shippingAddress.zip': '98101'})

# Nested array of embedded documents
orders.find({'items.productId': 'PROD-A'})  # Any item with this productId
```

---

## 4. Indexing

Indexes are critical for DocumentDB performance. Without an index, every query requires a full collection scan.

### Index Types

| Index Type | Use Case |
|---|---|
| Single-field | Filter or sort on one field |
| Compound | Filter/sort on multiple fields together |
| Multi-key | Fields that contain arrays — one index entry per array element |
| Sparse | Only index documents that have the indexed field |
| Unique | Enforce uniqueness on field values |
| Partial | Index only documents matching a filter expression |
| TTL | Automatically delete documents after a time period |

### Creating Indexes

```python
from pymongo import ASCENDING, DESCENDING

# Single-field index
orders.create_index('status')

# Compound index — order matters for query coverage
orders.create_index([('status', ASCENDING), ('createdAt', DESCENDING)])

# Unique index
customers.create_index('email', unique=True)

# Sparse index — only index documents where 'shippedAt' exists
orders.create_index('shippedAt', sparse=True)

# Partial index — only index orders with status != 'cancelled'
orders.create_index(
    'total',
    partialFilterExpression={'status': {'$ne': 'cancelled'}}
)

# TTL index — auto-delete documents after 30 days
orders.create_index('createdAt', expireAfterSeconds=2592000)

# Text index for basic string search (limited — use OpenSearch for full-text)
products.create_index([('name', 'text'), ('description', 'text')])
```

### Creating Indexes via CLI

```bash
# Use mongosh or the mongo shell via bastion/EC2
mongo --ssl --host mycluster.cluster-xyz.us-east-1.docdb.amazonaws.com:27017 \
  --sslCAFile rds-combined-ca-bundle.pem \
  -u admin -p password \
  --eval 'db.orders.createIndex({"status": 1, "createdAt": -1})'
```

### Listing and Dropping Indexes

```python
# List all indexes on a collection
for index in orders.list_indexes():
    print(index['name'], index['key'])

# Drop a specific index
orders.drop_index('status_1')

# Drop all indexes except _id
orders.drop_indexes()
```

### Index Design Guidelines

- **Leading field matters**: A compound index on `(status, createdAt)` supports queries filtering on `status` alone or `status + createdAt`. It does not help queries filtering only on `createdAt`.
- **Avoid over-indexing**: Each index adds write overhead and storage cost. Index only fields used in frequent query filters or sorts.
- **Covered queries**: If all fields in a query (filter + projection) are covered by an index, DocumentDB can answer the query from the index alone without reading the document.
- **Multi-key index cost**: Indexing an array field creates one index entry per array element. A document with 100 items in an array creates 100 index entries. Large arrays with multi-key indexes are expensive.

---

## 5. Aggregation Pipeline

The aggregation pipeline processes documents through a sequence of stages, each transforming the data. Stages execute in order, and each stage receives the output of the previous stage.

### Common Pipeline Stages

```python
# Sales summary by customer
pipeline = [
    # Stage 1: Filter
    {'$match': {'status': 'shipped', 'createdAt': {'$gte': '2024-01-01'}}},

    # Stage 2: Group by customerId, sum totals
    {'$group': {
        '_id': '$customerId',
        'orderCount': {'$sum': 1},
        'totalSpent': {'$sum': '$total'},
        'avgOrder': {'$avg': '$total'}
    }},

    # Stage 3: Filter groups
    {'$match': {'totalSpent': {'$gte': 500}}},

    # Stage 4: Sort descending by totalSpent
    {'$sort': {'totalSpent': -1}},

    # Stage 5: Take top 10
    {'$limit': 10},

    # Stage 6: Reshape output
    {'$project': {
        'customerId': '$_id',
        'orderCount': 1,
        'totalSpent': {'$round': ['$totalSpent', 2]},
        'avgOrder': {'$round': ['$avgOrder', 2]},
        '_id': 0
    }}
]

top_customers = list(orders.aggregate(pipeline))
```

### $unwind — Flatten Arrays

```python
# Flatten the items array to analyze per-product revenue
pipeline = [
    {'$unwind': '$items'},  # Creates one document per item
    {'$group': {
        '_id': '$items.productId',
        'totalRevenue': {'$sum': {'$multiply': ['$items.qty', '$items.price']}},
        'unitsSold': {'$sum': '$items.qty'}
    }},
    {'$sort': {'totalRevenue': -1}}
]

product_revenue = list(orders.aggregate(pipeline))
```

### $lookup — Join Between Collections

```python
# Join orders with customer details
pipeline = [
    {'$match': {'status': 'pending'}},
    {'$lookup': {
        'from': 'customers',        # Collection to join
        'localField': 'customerId', # Field in orders
        'foreignField': '_id',      # Field in customers
        'as': 'customerDetails'     # Output array field name
    }},
    {'$unwind': '$customerDetails'},  # Flatten the joined array
    {'$project': {
        'orderId': '$_id',
        'customerName': '$customerDetails.name',
        'customerEmail': '$customerDetails.email',
        'total': 1,
        '_id': 0
    }}
]

orders_with_customers = list(orders.aggregate(pipeline))
```

`$lookup` in DocumentDB performs an in-memory join on the instance. For large joins, ensure indexes exist on the `foreignField`. Avoid `$lookup` on very large collections — consider denormalizing data instead.

### $facet — Multiple Aggregations in One Pass

```python
# Compute multiple summaries in a single query
pipeline = [
    {'$match': {'status': {'$ne': 'cancelled'}}},
    {'$facet': {
        'byStatus': [
            {'$group': {'_id': '$status', 'count': {'$sum': 1}}}
        ],
        'totalRevenue': [
            {'$group': {'_id': None, 'total': {'$sum': '$total'}}}
        ],
        'avgOrderValue': [
            {'$group': {'_id': None, 'avg': {'$avg': '$total'}}}
        ]
    }}
]

result = list(orders.aggregate(pipeline))[0]
```

---

## 6. Update Operators

```python
# $inc — increment a numeric field
orders.update_one({'_id': 'ORD-001'}, {'$inc': {'retryCount': 1}})

# $mul — multiply a field
orders.update_one({'_id': 'ORD-001'}, {'$mul': {'total': 1.10}})  # Apply 10% tax

# $rename — rename a field
orders.update_many({}, {'$rename': {'qty': 'quantity'}})

# $unset — remove a field
orders.update_one({'_id': 'ORD-001'}, {'$unset': {'temporaryFlag': ''}})

# $push — add element to array
orders.update_one(
    {'_id': 'ORD-001'},
    {'$push': {'statusHistory': {'status': 'shipped', 'at': '2024-12-02T08:00:00Z'}}}
)

# $push with $each and $slice — maintain a bounded array (keep last 10 events)
orders.update_one(
    {'_id': 'ORD-001'},
    {'$push': {
        'events': {
            '$each': [{'type': 'viewed', 'at': '2024-12-01'}],
            '$slice': -10  # Keep only the last 10 elements
        }
    }}
)

# $pull — remove matching elements from array
orders.update_one(
    {'_id': 'ORD-001'},
    {'$pull': {'items': {'productId': 'PROD-REMOVED'}}}
)

# $addToSet — add to array only if not already present
products.update_one(
    {'_id': 'PROD-A'},
    {'$addToSet': {'tags': 'clearance'}}
)

# Array element update by position ($ positional operator)
orders.update_one(
    {'_id': 'ORD-001', 'items.productId': 'PROD-A'},
    {'$set': {'items.$.price': 24.99}}  # Update the matched item's price
)
```

---

## 7. Bulk Operations

Use `bulk_write` for batch inserts, updates, and deletes to reduce round trips and improve throughput.

```python
from pymongo import InsertOne, UpdateOne, DeleteOne

operations = [
    InsertOne({'_id': 'ORD-100', 'status': 'pending', 'total': 50.0}),
    UpdateOne({'_id': 'ORD-001'}, {'$set': {'status': 'shipped'}}),
    UpdateOne({'_id': 'ORD-002'}, {'$set': {'status': 'shipped'}}),
    DeleteOne({'_id': 'ORD-OLD'}),
]

result = orders.bulk_write(operations, ordered=False)  # ordered=False for parallel execution
print(f"Inserted: {result.inserted_count}")
print(f"Modified: {result.modified_count}")
print(f"Deleted: {result.deleted_count}")
```

`ordered=False` allows DocumentDB to execute operations in any order and continue even if individual operations fail. Use this for bulk imports where order does not matter. Use `ordered=True` (default) when operations depend on each other.

---

## 8. Explain Plans and Query Optimization

### Using Explain

```javascript
// In mongosh or mongo shell
db.orders.find({ status: "pending", total: { $gt: 100 } })
         .explain("executionStats")
```

Key fields in the explain output:

| Field | What to Look For |
|---|---|
| `stage` | `IXSCAN` = index used (good). `COLLSCAN` = full collection scan (bad for large collections). |
| `nReturned` | Documents returned by the query |
| `totalDocsExamined` | Documents examined. If >> `nReturned`, the query scans many documents it discards — improve with a better index |
| `totalKeysExamined` | Index entries scanned |
| `executionTimeMillis` | Query execution time |

### Common Optimization Patterns

**Problem: Query uses COLLSCAN**
```python
# Bad: no index on status
orders.find({'status': 'pending'})

# Fix: create an index
orders.create_index('status')
```

**Problem: Sort is not covered by index**
```python
# Bad: index on status only, sort on createdAt requires in-memory sort
orders.find({'status': 'pending'}).sort('createdAt', -1)

# Fix: compound index matching both filter and sort
orders.create_index([('status', 1), ('createdAt', -1)])
```

**Problem: Low selectivity index**
A field with only 2-3 unique values (like `status`) has low cardinality. DocumentDB may choose a COLLSCAN over an index if the index does not reduce the result set enough. Combine with a high-cardinality field in a compound index:

```python
# Better: compound with high-cardinality field first
orders.create_index([('customerId', 1), ('status', 1), ('createdAt', -1)])
```

---

## Key Takeaways

- Use dot notation for querying nested fields and array elements. `$elemMatch` is required when matching multiple conditions on a single array element.
- Always explain your queries during development. `COLLSCAN` on large collections will degrade in production. Target `IXSCAN` with low `totalDocsExamined`.
- Design compound indexes to match query patterns: leading fields should be equality filters, followed by range filters, followed by sort fields.
- `$lookup` performs in-memory joins. For large joins, consider embedding related data into the document or using a separate query.
- Use `bulk_write` with `ordered=False` for high-throughput batch operations.
