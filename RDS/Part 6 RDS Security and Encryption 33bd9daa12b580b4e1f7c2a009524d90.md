# Part 6: RDS Security and Encryption

---

## Table of Contents

1. [RDS Security Overview](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
2. [Network Security (VPC, Security Groups, NACLs)](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
3. [Database Authentication Methods](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
4. [IAM Database Authentication (Hands-On)](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
5. [Encryption at Rest](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
6. [Encryption in Transit (SSL/TLS)](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
7. [Secrets Manager Integration](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
8. [Audit Logging](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
9. [Database Activity Streams](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)
10. [Security Best Practices](Part%206%20RDS%20Security%20and%20Encryption%2033bd9daa12b580b4e1f7c2a009524d90.md)

---

## 1. RDS Security Overview

RDS security operates at multiple layers:

```
Security Layers
│
├── Network Security
│   ├── VPC (isolation)
│   ├── Security Groups (firewall)
│   ├── NACLs (subnet-level firewall)
│   └── Private subnets (no public internet)
│
├── Authentication
│   ├── Master username/password
│   ├── IAM database authentication
│   └── Kerberos (SQL Server, Oracle)
│
├── Encryption
│   ├── At rest (storage, backups)
│   └── In transit (SSL/TLS)
│
├── Access Control
│   ├── Database permissions (GRANT/REVOKE)
│   ├── IAM policies (who can manage RDS)
│   └── Resource tags
│
└── Auditing
    ├── CloudTrail (AWS API calls)
    ├── Database logs (error, slow query, general)
    └── Database Activity Streams (real-time)
```

### Defense in Depth Security Architecture

![RDS Security Layers](images/rds-security-layers.svg)

---

## 2. Network Security (VPC, Security Groups, NACLs)

### VPC Isolation

**Always place RDS in a VPC** (this is mandatory since 2013).

```
┌──────────────────────────────────────────────────────────┐
│                          VPC                              │
│                      10.0.0.0/16                          │
│                                                           │
│  ┌────────────────────┐      ┌────────────────────┐      │
│  │  Public Subnet     │      │  Private Subnet    │      │
│  │  10.0.1.0/24       │      │  10.0.10.0/24      │      │
│  │                    │      │                    │      │
│  │  ┌──────────┐      │      │  ┌──────────────┐ │      │
│  │  │  Bastion │      │      │  │  RDS Instance│ │      │
│  │  │  Host    │      │      │  │              │ │      │
│  │  └──────────┘      │      │  └──────────────┘ │      │
│  │       │            │      │                    │      │
│  └───────┼────────────┘      └────────────────────┘      │
│          │                                                │
│          │   SSH tunnel for admin access                 │
└──────────┼────────────────────────────────────────────────┘
           │
     Internet Gateway
```

**Best practices:**
- ✅ Use private subnets for RDS (no public access)
- ✅ Access via bastion host or VPN
- ❌ Never enable "Public access" in production

---

### Security Groups (Instance-Level Firewall)

Security groups act as a **stateful firewall** for your RDS instance.

**Example security group rules:**

```
Inbound Rules:
┌──────────┬──────────┬──────┬────────────────────────────┐
│   Type   │ Protocol │ Port │          Source            │
├──────────┼──────────┼──────┼────────────────────────────┤
│ MySQL    │   TCP    │ 3306 │ sg-app-servers (EC2 SG)    │
│ MySQL    │   TCP    │ 3306 │ sg-lambda (Lambda SG)      │
│ MySQL    │   TCP    │ 3306 │ sg-bastion (admin access)  │
└──────────┴──────────┴──────┴────────────────────────────┘

Outbound Rules:
┌──────────┬──────────┬──────┬────────────────────────────┐
│   Type   │ Protocol │ Port │       Destination          │
├──────────┼──────────┼──────┼────────────────────────────┤
│ All      │   All    │ All  │ 0.0.0.0/0 (allow all)      │
└──────────┴──────────┴──────┴────────────────────────────┘
```

**Security group best practices:**
- ✅ Use security groups as sources (not IP addresses)
- ✅ Principle of least privilege (only allow required sources)
- ✅ Separate security groups for different application tiers
- ❌ Don't allow 0.0.0.0/0 (entire internet) on inbound rules

---

### NACLs (Network Access Control Lists)

NACLs are **stateless firewalls** at the subnet level.

**When to use NACLs:**
- Additional layer of defense (defense in depth)
- Block specific IP ranges at subnet level
- Explicit deny rules (security groups can only allow)

**Most RDS deployments:** Default NACL (allow all) + restrictive security groups

---

## 3. Database Authentication Methods

### Method 1: Master Username and Password (Default)

```sql
mysql -h myapp-db.xxxxx.rds.amazonaws.com -u admin -p
Enter password: ********
```

**Pros:**
- Simple, works with all database engines
- Standard database authentication

**Cons:**
- Password stored in application config (risk of exposure)
- Password rotation requires application updates
- No centralized access control

---

### Method 2: IAM Database Authentication

Use **IAM credentials** instead of database passwords.

```
How it works:
1. Application assumes IAM role
2. Generates temporary password (auth token) using AWS STS
3. Connects to RDS using auth token (valid 15 minutes)
4. RDS validates token with IAM
```

**Pros:**
- No passwords stored in application code
- Centralized access control via IAM
- Automatic password rotation (tokens expire after 15 min)
- Integrates with IAM policies, roles, MFA

**Cons:**
- Only supports MySQL, PostgreSQL, MariaDB (not Oracle, SQL Server)
- Slightly more complex setup
- Token generation adds ~50ms latency per connection

---

### Method 3: Kerberos Authentication

**Only for:** SQL Server and Oracle  
**Use case:** Integrate with Active Directory

---

## 4. IAM Database Authentication (Hands-On)

### Step 1: Enable IAM Database Authentication

```
AWS Console → RDS → Databases → Modify
```

```
Database authentication:
☑ Password authentication
☑ Password and IAM database authentication
```

Apply changes.

---

### Step 2: Create IAM Policy

Create an IAM policy that allows connecting to RDS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds-db:connect"
      ],
      "Resource": [
        "arn:aws:rds-db:us-east-1:123456789012:dbuser:db-ABCDEFGHIJK/iamuser"
      ]
    }
  ]
}
```

**Resource ARN format:**
```
arn:aws:rds-db:REGION:ACCOUNT:dbuser:DB-RESOURCE-ID/DATABASE-USERNAME
```

Get `DB-RESOURCE-ID` from RDS console (Configuration tab).

---

### Step 3: Attach Policy to IAM Role

```
AWS Console → IAM → Roles → Create role
```

```
Trusted entity: EC2 (or Lambda, ECS, etc.)
Policy:         Attach the policy created above
Role name:      RDSIAMAuthRole
```

Attach this role to your EC2 instance.

---

### Step 4: Create Database User

Connect to RDS using master credentials:

```bash
mysql -h myapp-db.xxxxx.rds.amazonaws.com -u admin -p
```

Create a user for IAM authentication:

```sql
CREATE USER iamuser IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydatabase.* TO iamuser;
FLUSH PRIVILEGES;
```

**Important:** `IDENTIFIED WITH AWSAuthenticationPlugin` enables IAM auth for this user.

---

### Step 5: Generate Authentication Token

On your EC2 instance (with IAM role attached):

```bash
export RDSHOST="myapp-db.xxxxx.us-east-1.rds.amazonaws.com"
export RDSPORT="3306"
export RDSUSER="iamuser"
export RDSREGION="us-east-1"

# Generate auth token (valid 15 minutes)
TOKEN=$(aws rds generate-db-auth-token \
    --hostname $RDSHOST \
    --port $RDSPORT \
    --region $RDSREGION \
    --username $RDSUSER)

echo $TOKEN
```

Output:
```
myapp-db.xxxxx.us-east-1.rds.amazonaws.com:3306/?Action=connect&DBUser=iamuser&X-Amz-Security-Token=...
```

---

### Step 6: Connect Using Token

```bash
mysql -h $RDSHOST \
      -P $RDSPORT \
      -u $RDSUSER \
      --password=$TOKEN \
      --enable-cleartext-plugin \
      --ssl-ca=/path/to/rds-ca-bundle.pem
```

**Download CA bundle:**
```bash
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

---

### Step 7: Application Code Example (Python)

```python
import boto3
import pymysql

# Generate auth token
client = boto3.client('rds', region_name='us-east-1')
token = client.generate_db_auth_token(
    DBHostname='myapp-db.xxxxx.us-east-1.rds.amazonaws.com',
    Port=3306,
    DBUsername='iamuser'
)

# Connect to RDS
connection = pymysql.connect(
    host='myapp-db.xxxxx.us-east-1.rds.amazonaws.com',
    user='iamuser',
    password=token,
    database='mydatabase',
    ssl_ca='/path/to/global-bundle.pem'
)

# Use connection
cursor = connection.cursor()
cursor.execute("SELECT * FROM users")
print(cursor.fetchall())
connection.close()
```

---

## 5. Encryption at Rest

**Encryption at rest** protects data stored on disk.

### What Gets Encrypted

When you enable encryption, AWS encrypts:
- Database storage (EBS volumes)
- Automated backups
- Manual snapshots
- Read replicas
- Temporary files

---

### Encryption Key Management

RDS uses **AWS KMS (Key Management Service)** for encryption.

**Two key options:**

```
1. AWS-managed key (default): aws/rds
   - Managed by AWS
   - Free
   - Automatic rotation

2. Customer-managed key (CMK): your own KMS key
   - You control key policies
   - Audit key usage in CloudTrail
   - Can disable/delete key
   - Cost: $1/month per key
```

---

### Enable Encryption (New Instance)

When creating a new RDS instance:

```
Encryption:
☑ Enable encryption
AWS KMS key:  (default) aws/rds  OR  (select your CMK)
```

**Important:** Encryption **cannot** be disabled after instance creation.

---

### Enable Encryption (Existing Unencrypted Instance)

You **cannot** directly enable encryption on an existing instance.

**Migration process:**

```
1. Take a snapshot of unencrypted instance
2. Copy snapshot with encryption enabled
3. Restore encrypted snapshot to new instance
4. Point application to new instance
5. Delete old unencrypted instance
```

**Hands-on steps:**

```
AWS Console → RDS → Snapshots → Select snapshot → Actions → Copy snapshot
```

```
☑ Enable encryption
AWS KMS key: (default) aws/rds
```

Then restore from encrypted snapshot.

---

### Performance Impact

**Encryption overhead:** Negligible (~1-3% CPU overhead on modern instances)

**Recommendation:** Always enable encryption, even for dev/test databases.

---

## 6. Encryption in Transit (SSL/TLS)

**Encryption in transit** protects data traveling between client and RDS.

### Force SSL Connections (MySQL)

Edit parameter group:

```
AWS Console → RDS → Parameter groups → Create new parameter group
```

```
Parameter: require_secure_transport
Value:     1 (ON)
```

Apply parameter group to your instance (requires reboot).

**After applying:** All connections must use SSL. Non-SSL connections are rejected.

---

### Connect with SSL (MySQL)

```bash
mysql -h myapp-db.xxxxx.rds.amazonaws.com \
      -u admin \
      -p \
      --ssl-ca=/path/to/global-bundle.pem \
      --ssl-mode=REQUIRED
```

---

### Verify SSL Connection

```sql
SHOW STATUS LIKE 'Ssl_cipher';
```

Output:
```
+---------------+---------------------------+
| Variable_name | Value                     |
+---------------+---------------------------+
| Ssl_cipher    | ECDHE-RSA-AES128-SHA256   |
+---------------+---------------------------+
```

If `Ssl_cipher` is empty, connection is **not** encrypted.

---

### Force SSL (PostgreSQL)

Edit parameter group:

```
Parameter: rds.force_ssl
Value:     1
```

Reboot instance to apply.

---

### Application Code (Python with SSL)

```python
import pymysql

connection = pymysql.connect(
    host='myapp-db.xxxxx.us-east-1.rds.amazonaws.com',
    user='admin',
    password='password',
    database='mydatabase',
    ssl={'ca': '/path/to/global-bundle.pem'}
)
```

---

## 7. Secrets Manager Integration

**AWS Secrets Manager** stores and rotates database passwords automatically.

### Why Use Secrets Manager

- ✅ No passwords in application code
- ✅ Automatic password rotation (30/60/90 days)
- ✅ Audit access to secrets in CloudTrail
- ✅ Fine-grained IAM access control

**Cost:** $0.40 per secret per month + $0.05 per 10,000 API calls

---

### Create Secret in Secrets Manager

```
AWS Console → Secrets Manager → Store a new secret
```

```
Secret type:           Credentials for RDS database
Username:              admin
Password:              <your-password>
Encryption key:        aws/secretsmanager
RDS database:          myapp-db (select your instance)
```

```
Secret name:           rds/myapp-db/admin
Description:           RDS master credentials
```

```
Automatic rotation:    ☑ Enable automatic rotation
Rotation schedule:     30 days
```

Click **Store**.

---

### Retrieve Secret in Application (Python)

```python
import boto3
import json

# Get secret from Secrets Manager
client = boto3.client('secretsmanager', region_name='us-east-1')
response = client.get_secret_value(SecretId='rds/myapp-db/admin')
secret = json.loads(response['SecretString'])

# Extract credentials
username = secret['username']
password = secret['password']
host = secret['host']
port = secret['port']

# Connect to RDS
import pymysql
connection = pymysql.connect(
    host=host,
    port=port,
    user=username,
    password=password,
    database='mydatabase'
)
```

---

## 8. Audit Logging

### CloudTrail (AWS API Activity)

**CloudTrail** logs all RDS API calls:
- Who created/modified/deleted instances
- Who took snapshots
- Parameter group changes
- Security group modifications

**Enable CloudTrail:**
```
AWS Console → CloudTrail → Create trail
```

---

### Database Logs

RDS publishes database logs to **CloudWatch Logs**.

**MySQL logs:**
- Error log
- Slow query log
- General log (all queries — very verbose, use sparingly)

**Enable logs:**

```
AWS Console → RDS → Databases → Modify
```

```
Log exports:
☑ Error log
☑ Slow query log
☐ General log (not recommended for production)
```

---

### View Logs in CloudWatch

```
AWS Console → CloudWatch → Log groups
```

Look for:
```
/aws/rds/instance/myapp-db/error
/aws/rds/instance/myapp-db/slowquery
```

---

### Slow Query Log Analysis

Slow query log shows queries exceeding `long_query_time` (default: 10 seconds).

**Enable slow query log:**

Edit parameter group:
```
Parameter: slow_query_log
Value:     1 (ON)

Parameter: long_query_time
Value:     2 (log queries > 2 seconds)
```

**View slow queries in CloudWatch:**
```
Filter pattern: "Query_time:"
```

---

## 9. Database Activity Streams

**Database Activity Streams** provide real-time audit log of all database activity.

**Supported engines:**
- Oracle (all editions)
- PostgreSQL
- Aurora MySQL, Aurora PostgreSQL

**Not supported:** MySQL (non-Aurora), MariaDB, SQL Server

---

### What Gets Logged

- Every SQL statement executed
- Who executed it (database user)
- When it was executed
- Connection details
- Changes to database objects

**Stream is tamper-proof** (cannot be modified by DBA or root user).

---

### Enable Database Activity Streams

```
AWS Console → RDS → Databases → Select instance → Actions → Start activity stream
```

```
AWS KMS key:     Select encryption key
Mode:            Asynchronous (recommended, no performance impact)
                 Synchronous (ensures every activity is logged before commit)
```

Click **Start**.

**Cost:** ~$0.135 per million database events

---

### Consume Activity Stream

Activity streams are published to **Amazon Kinesis Data Streams**.

**Process logs with:**
- Lambda function
- Kinesis Data Firehose → S3 → Athena
- Third-party SIEM tools (Splunk, Datadog)

---

## 10. Security Best Practices

### Network Security

- ✅ Always place RDS in **private subnets**
- ✅ Never enable **public access** in production
- ✅ Use **security groups** with least privilege (specific sources only)
- ✅ Access via bastion host, VPN, or AWS PrivateLink
- ❌ Don't allow 0.0.0.0/0 on inbound security group rules

### Authentication

- ✅ Use **IAM database authentication** for applications
- ✅ Store passwords in **Secrets Manager** (not application code)
- ✅ Use **strong master passwords** (16+ characters, complex)
- ✅ Rotate passwords regularly (30-90 days)
- ❌ Don't hardcode passwords in code or config files

### Encryption

- ✅ **Always enable encryption at rest** (storage, backups)
- ✅ **Force SSL/TLS** for all connections (encryption in transit)
- ✅ Use **customer-managed KMS keys** for sensitive data
- ✅ Verify SSL connections in application (`Ssl_cipher` check)
- ❌ Don't transmit data over unencrypted connections

### Access Control

- ✅ Follow **principle of least privilege** (grant only needed permissions)
- ✅ Create **separate database users** for each application/service
- ✅ Use **IAM policies** to control who can manage RDS instances
- ✅ Enable **deletion protection** for production databases
- ❌ Don't use master user for application connections

### Auditing and Monitoring

- ✅ Enable **CloudTrail** for AWS API audit logs
- ✅ Enable **slow query log** and monitor in CloudWatch
- ✅ Set up **CloudWatch alarms** for suspicious activity
- ✅ Enable **Database Activity Streams** for compliance requirements
- ✅ Review logs regularly (weekly or automated via SIEM)
- ❌ Don't ignore security events or failed login attempts

### Backup and DR

- ✅ Enable **automated backups** (minimum 7 days retention)
- ✅ **Encrypt backups** (automatic if instance is encrypted)
- ✅ **Copy snapshots** to another region for disaster recovery
- ✅ **Test restore procedures** quarterly
- ❌ Don't rely on backups without testing recovery

---