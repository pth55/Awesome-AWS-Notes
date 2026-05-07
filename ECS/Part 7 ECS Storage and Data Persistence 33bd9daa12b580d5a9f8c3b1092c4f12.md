# Part 7: ECS Storage and Data Persistence

## Storage Options Overview

| Storage Type | Persistence | Shared | Performance | Use Case |
|:-------------|:------------|:-------|:------------|:---------|
| **Ephemeral** | Task lifetime only | No | Fast (local) | Temporary data, caches |
| **EFS** | Permanent | Yes (across tasks) | Good (network) | Shared files, content |
| **EBS** | Permanent | No | Fast (block) | Databases (EC2 only) |
| **Bind Mounts** | Host lifetime | No | Fast (local) | Container-to-container |

## Ephemeral Storage

Default storage in every container. Deleted when task stops.

**Fargate:**
- Default: 20 GB
- Maximum: 200 GB (requires platform version 1.4.0+)

```json
{
  "family": "my-app",
  "ephemeralStorage": {
    "sizeInGiB": 50
  }
}
```

**EC2 Launch Type:**
- Uses instance root volume
- Size depends on instance type

**Use for:**
- Application temporary files
- Build artifacts
- Caches that can be regenerated

## Amazon EFS Integration

Fully managed NFS file system, shared across multiple tasks.

### Why Use EFS with ECS?

```
Traditional containers:
  Task 1 → Ephemeral storage → Data lost on restart
  Task 2 → Ephemeral storage → Cannot share data with Task 1

With EFS:
  Task 1 → EFS → /shared
  Task 2 → EFS → /shared (same data)
  Task 3 → EFS → /shared (same data)
```

### Setup EFS for ECS

**1. Create EFS File System**
```bash
# Create EFS
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=ecs-shared-storage \
  --query 'FileSystemId' \
  --output text)

echo "EFS ID: $EFS_ID"

# Create mount targets in each AZ
aws efs create-mount-target \
  --file-system-id $EFS_ID \
  --subnet-id subnet-private-1 \
  --security-groups sg-efs

aws efs create-mount-target \
  --file-system-id $EFS_ID \
  --subnet-id subnet-private-2 \
  --security-groups sg-efs
```

**2. Create Security Group for EFS**
```bash
# EFS security group
aws ec2 create-security-group \
  --group-name efs-sg \
  --description "Allow NFS from ECS tasks" \
  --vpc-id vpc-abc123

# Allow NFS (port 2049) from task security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-efs \
  --protocol tcp \
  --port 2049 \
  --source-group sg-task
```

**3. Task Definition with EFS**
```json
{
  "family": "app-with-efs",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "volumes": [
    {
      "name": "shared-storage",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-abc123",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "app",
      "image": "myapp:latest",
      "mountPoints": [
        {
          "sourceVolume": "shared-storage",
          "containerPath": "/mnt/efs",
          "readOnly": false
        }
      ]
    }
  ]
}
```

**Register and run:**
```bash
aws ecs register-task-definition --cli-input-json file://task-with-efs.json

aws ecs run-task \
  --cluster production \
  --task-definition app-with-efs:1 \
  --launch-type FARGATE \
  --network-configuration '...'
```

### EFS Use Cases

**Content Management:**
```
Web Tasks (3 replicas) → EFS → /var/www/uploads
  └── All tasks see same uploaded files
```

**Shared Configuration:**
```
API Tasks → EFS → /config
  └── Update config file, all tasks see changes
```

**Machine Learning:**
```
Training Task → EFS → /models
Inference Tasks → EFS → /models (read-only)
```

## Docker Volumes

### Bind Mounts

Share data between containers in the same task.

```json
{
  "volumes": [
    {
      "name": "shared-data",
      "host": {}
    }
  ],
  "containerDefinitions": [
    {
      "name": "app",
      "mountPoints": [{
        "sourceVolume": "shared-data",
        "containerPath": "/data"
      }]
    },
    {
      "name": "sidecar",
      "mountPoints": [{
        "sourceVolume": "shared-data",
        "containerPath": "/data"
      }]
    }
  ]
}
```

**Use case**: App container writes logs → Sidecar container ships logs

### Docker Volume Drivers

For EC2 launch type, you can use Docker volume plugins:

**Example with Portworx:**
```json
{
  "volumes": [
    {
      "name": "portworx-volume",
      "dockerVolumeConfiguration": {
        "scope": "shared",
        "autoprovision": true,
        "driver": "pxd"
      }
    }
  ]
}
```

## EBS Volumes (EC2 Only)

Attach EBS volumes to EC2 instances, then mount in containers.

**Limitations:**
- Fargate does not support EBS
- Volume tied to single AZ (not shared)
- Requires custom AMI or user data script

**Setup:**
```bash
# In Launch Template user data:
#!/bin/bash
# Mount EBS volume
mkfs -t ext4 /dev/xvdf
mkdir -p /data
mount /dev/xvdf /data
```

**Task definition:**
```json
{
  "volumes": [{
    "name": "ebs-volume",
    "host": {
      "sourcePath": "/data"
    }
  }],
  "containerDefinitions": [{
    "mountPoints": [{
      "sourceVolume": "ebs-volume",
      "containerPath": "/app/data"
    }]
  }]
}
```

**Use case**: Single-task database (PostgreSQL, MySQL)

## Data Persistence Patterns

### Pattern 1: Stateless with External Storage

```
Tasks (stateless) → RDS/DynamoDB (persistent data)
                  → S3 (file storage)
```

**Best for**: Web apps, APIs, microservices

### Pattern 2: Shared File System

```
Multiple Tasks → EFS → Shared data
```

**Best for**: Content management, shared uploads

### Pattern 3: Task-Local Persistent Storage

```
Single Task → EBS/Local Volume → Database files
```

**Best for**: Development databases, single-instance workloads

### Pattern 4: Backup to S3

```
Task → Ephemeral storage → Periodic backup to S3
```

**Best for**: Batch processing, build outputs

## Storage Performance

### EFS Performance Modes

| Mode | Throughput | Latency | Use Case |
|:-----|:-----------|:--------|:---------|
| **General Purpose** | Baseline + burst | Low | Most workloads |
| **Max I/O** | Higher throughput | Higher latency | Big data, analytics |

### EFS Throughput Modes

| Mode | Description | Cost |
|:-----|:------------|:-----|
| **Bursting** | Scales with size | Lower |
| **Provisioned** | Fixed throughput | Higher |
| **Elastic** | Auto-scales | Pay for use |

**Example:**
```bash
# Create EFS with provisioned throughput
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode provisioned \
  --provisioned-throughput-in-mibps 100
```

## Backup and Recovery

### EFS Backups

**AWS Backup:**
```bash
# Create backup plan
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "efs-daily-backup",
    "Rules": [{
      "RuleName": "DailyBackup",
      "TargetBackupVaultName": "Default",
      "ScheduleExpression": "cron(0 2 * * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 120,
      "Lifecycle": {
        "DeleteAfterDays": 30
      }
    }]
  }'
```

### Ephemeral Data Backup

For critical ephemeral data, sync to S3:

```bash
# In container startup script
aws s3 sync /data s3://my-bucket/backup/ --delete
```

## Troubleshooting

### EFS Mount Fails

**Error**: "Connection timed out"

**Check:**
```bash
# 1. Security group allows NFS (port 2049)
aws ec2 describe-security-groups --group-ids sg-efs

# 2. Mount targets exist in task subnets
aws efs describe-mount-targets --file-system-id fs-abc

# 3. DNS resolution works
nslookup fs-abc.efs.us-east-1.amazonaws.com
```

### Slow EFS Performance

**Solutions:**
1. Switch to Provisioned or Elastic throughput mode
2. Use Max I/O performance mode for parallel access
3. Enable EFS Intelligent-Tiering for frequently accessed files

### Out of Disk Space

**Ephemeral storage full:**
```bash
# Exec into container
aws ecs execute-command --cluster prod --task task-id --container app --command "/bin/bash"

# Check disk usage
df -h
du -sh /data/*
```

**Solution:**
- Increase ephemeral storage size
- Clean up temporary files
- Move to EFS for large files

---

**Part 7 Complete. Next: Part 8 covers monitoring, logging, and security.**
