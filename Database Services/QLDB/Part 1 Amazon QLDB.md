# Part 1 — Amazon QLDB (Quantum Ledger Database)

## Table of Contents

1. [What is QLDB](#1-what-is-qldb)
2. [Core Concepts — Journal, Tables, and Documents](#2-core-concepts--journal-tables-and-documents)
3. [PartiQL — Query Language](#3-partiql--query-language)
4. [Cryptographic Verification](#4-cryptographic-verification)
5. [Architecture and Storage](#5-architecture-and-storage)
6. [CRUD Operations with PartiQL](#6-crud-operations-with-partiql)
7. [History Queries](#7-history-queries)
8. [Security](#8-security)
9. [Streams and Integrations](#9-streams-and-integrations)
10. [Pricing and When to Use](#10-pricing-and-when-to-use)

---

## 1. What is QLDB

Amazon QLDB (Quantum Ledger Database) is a **fully managed ledger database** with a built-in, cryptographically verifiable transaction log. Every data change — insert, update, delete — is recorded in an **immutable journal**. No change can be deleted, altered, or backdated without detection.

QLDB solves a specific problem: **proving that a historical record has not been tampered with**. Traditional databases allow data to be updated or deleted. Even with audit tables or change-data-capture, a sufficiently privileged administrator can alter both the data and the audit trail. QLDB makes this structurally impossible by using a cryptographic hash chain (similar to a blockchain) in the journal.

### The Core Guarantee

Every transaction written to QLDB generates a **cryptographic digest** — a hash of all preceding data changes. You can provide this digest to auditors as proof that the historical record is intact. If anyone modifies the journal, the digest will not match, and the tampering is detectable.

### QLDB vs Blockchain

QLDB is **not** a decentralized blockchain. It is a centralized ledger managed by AWS. The difference:
- **Blockchain**: Decentralized, trustless, consensus among multiple parties.
- **QLDB**: Centralized, trusted authority (AWS + your account), cryptographic tamper-evidence.

QLDB is appropriate when **one organization** needs to prove data integrity to auditors, regulators, or partners — without requiring a multi-party consensus mechanism. If you need multiple untrusted parties to agree on a shared ledger, use Amazon Managed Blockchain.

---

## 2. Core Concepts — Journal, Tables, and Documents

### Journal

The journal is the core of QLDB. It is an **append-only, cryptographically chained** sequence of all committed transactions. Every data change creates a journal entry. Journal entries can never be modified, deleted, or backdated.

The journal stores:
- The transaction ID
- The timestamp
- The hash of the transaction payload
- The hash of the previous journal entry (creating the chain)

### Tables and Documents

QLDB organizes data in **tables** containing **documents** (similar to document databases). Documents are stored in **Amazon Ion** format — a superset of JSON that adds types like `timestamp`, `decimal`, `blob`, and `clob`.

```ion
{
  vehicleId: "VIN-001",
  make: "Toyota",
  model: "Camry",
  year: 2022,
  registrationDate: 2022-03-15T,
  owner: {
    name: "Alice Johnson",
    licenseNumber: "WA-DL-12345"
  },
  status: "active"
}
```

### Revisions

Every document change (insert, update) creates a new **revision**. Previous revisions are retained in the journal. A document's current state is its latest revision. The full history of all revisions is always accessible.

Each revision has system-controlled metadata:
- `version`: Monotonically incrementing revision number (starts at 0)
- `txId`: Transaction ID that created this revision
- `txTime`: Timestamp of the transaction
- `hash`: Hash of this revision's content

---

## 3. PartiQL — Query Language

QLDB uses **PartiQL** — a SQL-compatible query language for semi-structured data (Amazon Ion documents). PartiQL extends SQL to handle nested structures and arrays without losing SQL's familiar syntax.

```sql
-- Standard SQL syntax works for flat fields
SELECT vehicleId, make, model FROM Vehicle WHERE year >= 2020;

-- Nested field access with dot notation
SELECT vehicleId, owner.name AS ownerName
FROM Vehicle
WHERE owner.licenseNumber = 'WA-DL-12345';

-- Array handling (unnesting)
SELECT v.vehicleId, t.infraction
FROM Vehicle v, v.trafficInfractions t
WHERE t.severity = 'major';
```

---

## 4. Cryptographic Verification

QLDB's verification mechanism uses a **Merkle tree** hash structure over the journal. You can verify that a specific document revision was part of the committed journal without trusting any intermediary.

### The Verification Process

1. **Get a digest**: A digest is a cryptographic hash representing the state of the entire journal up to a point in time.
2. **Get a proof**: For a specific document revision, request a proof — a list of hashes that forms a path in the Merkle tree from the document to the digest root.
3. **Verify**: Compute the hash path using the document content and the proof. If the final hash matches the digest, the document's revision is proven to be part of the original journal.

```python
import boto3
import hashlib
import base64

qldb_client = boto3.client('qldb', region_name='us-east-1')

# Step 1: Get a digest (this is what you give to auditors as proof of state)
digest_response = qldb_client.get_digest(Name='vehicle-registration')
digest = digest_response['Digest']
digest_tip_address = digest_response['DigestTipAddress']

print(f"Digest (give to auditor): {base64.b64encode(digest).decode()}")
print(f"At block position: {digest_tip_address}")

# Step 2: Get a revision (specific document version)
# You need the document ID and block address from the document metadata
revision_response = qldb_client.get_revision(
    Name='vehicle-registration',
    BlockAddress={'IonText': str(block_address)},
    DocumentId='document-id-here',
    DigestTipAddress={'IonText': str(digest_tip_address)}
)

proof = revision_response['Proof']

# Step 3: Verify (typically done with the QLDB verification library)
# The Python QLDB driver includes verification utilities
# pyqldb includes verify_document function
```

### Practical Verification Workflow

In regulated industries (finance, healthcare, government), you:
1. After each audit period, call `GetDigest` and store the digest in a secure, separate system (or give it to auditors/regulators).
2. When an audit requires proof that a specific transaction occurred as claimed, provide the digest + a proof generated by `GetRevision`.
3. The auditor (or automated system) computes the verification and confirms the transaction was part of the original journal.

---

## 5. Architecture and Storage

QLDB is fully managed and serverless. There are no instances to provision, no clusters, no storage sizing. AWS handles all scaling automatically.

### Internal Structure

```
QLDB Ledger
  │
  ├── Journal (immutable, append-only, cryptographic hash chain)
  │     └── All transactions, sequenced and hashed
  │
  └── Indexed Storage (current document states + queryable indexes)
        ├── Table: Vehicle
        ├── Table: Registration
        └── Table: TitleHistory
```

The journal is the source of truth. The indexed storage is a queryable view derived from the journal. If you delete a document (mark it as removed), the deletion is a new journal entry — the document's history remains in the journal permanently.

### Concurrency Control

QLDB uses **optimistic concurrency control (OCC)**. Transactions read data, make changes, and commit. At commit time, QLDB checks if any of the read data changed since the transaction started. If so, the transaction is rejected with an `OccConflictException` and must be retried.

This means:
- No row-level locking.
- High read concurrency.
- Write conflicts on the same document require retry logic.

---

## 6. CRUD Operations with PartiQL

### Setup

```python
from pyqldb.driver.qldb_driver import QldbDriver

driver = QldbDriver(
    ledger_name='vehicle-registration',
    region_name='us-east-1'
)
```

### Insert

```python
def register_vehicle(vin, make, model, year, owner_name, license_number):
    def transaction(executor):
        executor.execute_statement(
            "INSERT INTO Vehicle VALUE ?",
            {
                'vehicleId': vin,
                'make': make,
                'model': model,
                'year': year,
                'status': 'active',
                'owner': {
                    'name': owner_name,
                    'licenseNumber': license_number
                }
            }
        )

    driver.execute_lambda(transaction)

register_vehicle('VIN-001', 'Toyota', 'Camry', 2022, 'Alice Johnson', 'WA-DL-12345')
```

### Query

```python
def get_vehicle(vin):
    def transaction(executor):
        cursor = executor.execute_statement(
            "SELECT * FROM Vehicle WHERE vehicleId = ?", vin
        )
        results = list(cursor)
        return results[0] if results else None

    return driver.execute_lambda(transaction)

vehicle = get_vehicle('VIN-001')
print(vehicle)
```

### Update

```python
def transfer_ownership(vin, new_owner_name, new_license):
    def transaction(executor):
        # QLDB creates a new revision — old owner data is preserved in journal
        executor.execute_statement(
            """
            UPDATE Vehicle
            SET owner.name = ?, owner.licenseNumber = ?
            WHERE vehicleId = ?
            """,
            new_owner_name, new_license, vin
        )

    driver.execute_lambda(transaction)

transfer_ownership('VIN-001', 'Bob Smith', 'WA-DL-67890')
```

### Delete

```python
def deregister_vehicle(vin):
    def transaction(executor):
        # DELETE creates a tombstone revision in the journal
        # Document history remains accessible in the journal permanently
        executor.execute_statement(
            "DELETE FROM Vehicle WHERE vehicleId = ?", vin
        )

    driver.execute_lambda(transaction)
```

Deleting a document removes it from the current state (it will not appear in normal queries). The deletion event is written to the journal as a final revision. Historical queries via `history()` still return all previous revisions including the delete event.

### Handling OCC Conflicts

```python
from pyqldb.errors import is_occ_conflict_exception

def safe_update(vin, new_status, max_retries=3):
    for attempt in range(max_retries):
        try:
            def transaction(executor):
                executor.execute_statement(
                    "UPDATE Vehicle SET status = ? WHERE vehicleId = ?",
                    new_status, vin
                )
            driver.execute_lambda(transaction)
            return  # Success
        except Exception as e:
            if is_occ_conflict_exception(e) and attempt < max_retries - 1:
                continue  # Retry
            raise  # Give up after max retries
```

The QLDB Python driver also handles OCC retries automatically when using `execute_lambda`. The built-in retry logic is sufficient for most use cases.

---

## 7. History Queries

The `history()` function returns all revisions of a document, including deleted documents.

```sql
-- All revisions of vehicle VIN-001
SELECT h.data.vehicleId,
       h.data.owner.name AS ownerName,
       h.metadata.version AS revision,
       h.metadata.txTime AS changedAt
FROM history(Vehicle) AS h
WHERE h.data.vehicleId = 'VIN-001'
ORDER BY h.metadata.version ASC;
```

Output:

```
vehicleId | ownerName     | revision | changedAt
VIN-001   | Alice Johnson | 0        | 2022-03-15T10:00:00Z  ← initial registration
VIN-001   | Bob Smith     | 1        | 2024-06-01T14:30:00Z  ← ownership transfer
VIN-001   | Bob Smith     | 2        | 2024-12-01T09:00:00Z  ← deletion (tombstone)
```

You can also bound the history query to a time range:

```sql
SELECT h.data, h.metadata
FROM history(Vehicle, `2024-01-01T`, `2024-12-31T`) AS h
WHERE h.data.vehicleId = 'VIN-001';
```

---

## 8. Security

### IAM

All QLDB access is controlled by IAM. Common actions:

| Action | Description |
|---|---|
| `qldb:SendCommand` | Execute PartiQL statements (required for all queries) |
| `qldb:GetDigest` | Retrieve journal digest for verification |
| `qldb:GetRevision` | Retrieve specific document revision for verification |
| `qldb:CreateLedger` | Create a new ledger |
| `qldb:DeleteLedger` | Delete a ledger |
| `qldb:ExportJournalToS3` | Export journal to S3 |

```json
{
  "Effect": "Allow",
  "Action": ["qldb:SendCommand"],
  "Resource": "arn:aws:qldb:us-east-1:123456789012:ledger/vehicle-registration"
}
```

### Encryption

All QLDB data is encrypted at rest. You can use AWS-owned keys (default) or specify a customer-managed KMS key when creating the ledger:

```bash
aws qldb create-ledger \
  --name vehicle-registration \
  --permissions-mode STANDARD \
  --kms-key alias/my-qldb-key
```

### Deletion Protection

QLDB ledgers have deletion protection enabled by default. You must explicitly disable it before deleting:

```bash
aws qldb update-ledger \
  --name vehicle-registration \
  --no-deletion-protection

aws qldb delete-ledger --name vehicle-registration
```

---

## 9. Streams and Integrations

### QLDB Streams

QLDB Streams exports a continuous, real-time stream of journal data to **Amazon Kinesis Data Streams**. Every committed transaction appears in the stream in journal order.

```bash
aws qldb create-ledger-stream \
  --ledger-name vehicle-registration \
  --inclusive-start-time 2024-12-01T00:00:00Z \
  --stream-name vehicle-changes \
  --role-arn arn:aws:iam::123456789012:role/QLDBStreamRole \
  --kinesis-configuration ShardCount=2,AggregationEnabled=true
```

Downstream consumers of the Kinesis stream can:
- Maintain a replica of the data in another store (DynamoDB, ElastiCache, OpenSearch).
- Trigger workflows on data changes (Lambda).
- Stream changes to data warehouses (Kinesis → Firehose → Redshift/S3).

### Journal Export to S3

For archival and offline audit, export the raw journal to S3:

```bash
aws qldb export-journal-to-s3 \
  --name vehicle-registration \
  --inclusive-start-time 2024-01-01T00:00:00Z \
  --exclusive-end-time 2024-12-31T23:59:59Z \
  --s3-export-configuration '{
    "Bucket": "my-qldb-archives",
    "Prefix": "vehicle-reg-2024/",
    "EncryptionConfiguration": {"ObjectEncryptionType": "SSE_S3"}
  }' \
  --role-arn arn:aws:iam::123456789012:role/QLDBExportRole
```

The exported journal is in Amazon Ion format and can be queried with Athena or processed offline.

---

## 10. Pricing and When to Use

### Pricing (us-east-1)

| Component | Price |
|---|---|
| Journal storage | $0.03/GB-month |
| Indexed storage | $0.30/GB-month |
| I/O requests (journal) | $0.04/million |
| Data transfer | Standard AWS rates |

### When to Use QLDB

Use QLDB when:
- You need **immutable audit history** and the ability to cryptographically prove that records have not been tampered with.
- Use cases: **financial transaction ledgers**, **vehicle registration and title history**, **supply chain provenance tracking**, **healthcare record audit trails**, **legal document versioning**, **regulatory compliance logs**.
- You need to answer "prove that this is what the record said at time X and it has not changed."

Avoid QLDB when:
- You need a **high-throughput transactional database** — QLDB is not optimized for bulk write throughput.
- Your data changes **rarely and verification is not required** — a standard database with CloudTrail logging is simpler.
- You need **cross-table JOINs** or **complex aggregations** — QLDB's PartiQL support has limitations compared to full SQL.
- You need **distributed ledger with multiple untrusted parties** — use Amazon Managed Blockchain.
- Your data structure is **highly relational** — use RDS.

### QLDB vs Traditional Audit Tables

| Aspect | Traditional Audit Table | QLDB Journal |
|---|---|---|
| Tamper-evident | No — admin can modify audit table | Yes — hash chain makes modification detectable |
| Cryptographic proof | No | Yes (Merkle tree + digest) |
| Performance overhead | Separate write per change | Integrated journal, no extra writes |
| Query history | Custom SQL on audit table | Built-in `history()` function |
| Storage cost | Standard database rates | $0.03/GB-month journal + $0.30/GB-month indexed |

---

## Key Takeaways

- QLDB is for **tamper-evident audit history**, not general-purpose database workloads.
- The journal is append-only and cryptographically chained. Even AWS cannot alter the journal without the digest hash changing.
- Every data change creates a new document revision. History is always accessible via `history()`.
- PartiQL is SQL-compatible with Ion document support — familiar to any SQL developer.
- Use `GetDigest` to get a snapshot of the journal state to give to auditors. Use `GetRevision` + `Proof` to prove a specific transaction was part of that journal.
- QLDB Streams exports the journal to Kinesis for downstream consumers (search indexes, event processing, data warehouses).
