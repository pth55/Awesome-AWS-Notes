# Part 8: Monitoring, Logging, and Security Best Practices

## Table of Contents

1. [CloudWatch Container Insights](#1-cloudwatch-container-insights)
2. [Logging Strategies](#2-logging-strategies)
3. [IAM Roles for ECS](#3-iam-roles-for-ecs)
4. [Secrets Management](#4-secrets-management)
5. [Security Best Practices](#5-security-best-practices)
6. [Compliance and Auditing](#6-compliance-and-auditing)
7. [Performance Monitoring](#7-performance-monitoring)
8. [Alerting and Notifications](#8-alerting-and-notifications)
9. [Cost Monitoring](#9-cost-monitoring)
10. [Security Scanning](#10-security-scanning)

---

## 1. CloudWatch Container Insights

Container Insights collects, aggregates, and visualizes metrics and logs from your ECS clusters.

### Enable Container Insights

**For New Cluster:**
```bash
aws ecs create-cluster \
  --cluster-name production \
  --settings name=containerInsights,value=enabled
```

**For Existing Cluster:**
```bash
aws ecs update-cluster-settings \
  --cluster production \
  --settings name=containerInsights,value=enabled
```

### Metrics Collected

**Cluster-Level:**
- CPU utilization
- Memory utilization
- Network throughput
- Task count
- Service count

**Service-Level:**
- Desired vs running tasks
- Pending tasks
- Service CPU/memory

**Task-Level:**
- Per-task CPU/memory
- Network I/O
- Storage I/O

### Viewing Container Insights

```bash
# CloudWatch Console → Container Insights → ECS

# Or via CLI
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=web-service Name=ClusterName,Value=production \
  --start-time 2026-04-29T00:00:00Z \
  --end-time 2026-04-29T23:59:59Z \
  --period 3600 \
  --statistics Average
```

**Cost:** ~$0.30 per task per month

---

## 2. Logging Strategies

### CloudWatch Logs (Recommended)

**Task Definition:**
```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/my-app",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "web",
      "awslogs-create-group": "true"
    }
  }
}
```

**Create Log Group:**
```bash
aws logs create-log-group \
  --log-group-name /ecs/my-app \
  --tags Environment=production,Application=web

# Set retention
aws logs put-retention-policy \
  --log-group-name /ecs/my-app \
  --retention-in-days 30
```

### Viewing Logs

```bash
# Tail logs in real-time
aws logs tail /ecs/my-app --follow

# Filter logs
aws logs filter-log-events \
  --log-group-name /ecs/my-app \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s)000

# Query with CloudWatch Insights
aws logs start-query \
  --log-group-name /ecs/my-app \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
```

### Structured Logging

**Application logs in JSON:**
```json
{
  "timestamp": "2026-04-29T10:30:00Z",
  "level": "ERROR",
  "message": "Database connection failed",
  "error": "Connection timeout",
  "service": "api",
  "user_id": "12345"
}
```

**CloudWatch Insights query:**
```
fields @timestamp, level, message, service, user_id
| filter level = "ERROR"
| stats count() by service
```

### Centralized Logging with FireLens

Route logs to Datadog, Splunk, Elasticsearch:

```json
{
  "logConfiguration": {
    "logDriver": "awsfirelens",
    "options": {
      "Name": "datadog",
      "Host": "http-intake.logs.datadoghq.com",
      "TLS": "on",
      "dd_service": "my-app",
      "dd_source": "ecs",
      "dd_tags": "env:production",
      "provider": "ecs"
    }
  }
}
```

---

## 3. IAM Roles for ECS

### Three Types of Roles

```
┌─────────────────────────────────────────────────────┐
│                 ECS Security Model                  │
│                                                     │
│  1. ECS Instance Role (EC2 only)                   │
│     → Allows EC2 instance to register with ECS     │
│     → Required for EC2 launch type                 │
│                                                     │
│  2. ECS Task Execution Role                        │
│     → Allows ECS to pull images from ECR           │
│     → Write logs to CloudWatch                     │
│     → Get secrets from Secrets Manager             │
│     → Required for ALL launch types                │
│                                                     │
│  3. ECS Task Role                                  │
│     → Allows application code to access AWS        │
│     → S3, DynamoDB, SQS, etc.                      │
│     → Optional (only if app needs AWS access)      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Task Execution Role (Required)

```bash
# Create role
cat > task-execution-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://task-execution-trust-policy.json

# Attach managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

**Permissions included:**
- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:GetDownloadUrlForLayer`
- `ecr:BatchGetImage`
- `logs:CreateLogStream`
- `logs:PutLogEvents`

### Task Role (Application Permissions)

```bash
# Create task role
aws iam create-role \
  --role-name myAppTaskRole \
  --assume-role-policy-document file://task-trust-policy.json

# Create custom policy
cat > app-permissions.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name myAppTaskRole \
  --policy-name AppPermissions \
  --policy-document file://app-permissions.json
```

**In task definition:**
```json
{
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/myAppTaskRole"
}
```

---

## 4. Secrets Management

### AWS Secrets Manager

**Create Secret:**
```bash
aws secretsmanager create-secret \
  --name prod/myapp/db-password \
  --description "Database password for production" \
  --secret-string "SuperSecurePassword123!"
```

**Use in Task Definition:**
```json
{
  "containerDefinitions": [{
    "name": "app",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/db-password"
      }
    ]
  }]
}
```

**Task Execution Role needs:**
```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/*"
}
```

### Systems Manager Parameter Store

**Create Parameter:**
```bash
aws ssm put-parameter \
  --name /prod/myapp/api-key \
  --value "api-key-value" \
  --type SecureString \
  --key-id alias/aws/ssm
```

**Use in Task Definition:**
```json
{
  "secrets": [
    {
      "name": "API_KEY",
      "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/prod/myapp/api-key"
    }
  ]
}
```

### Environment Variables (Not Recommended for Secrets)

```json
{
  "environment": [
    {
      "name": "APP_ENV",
      "value": "production"
    }
  ]
}
```

**Never store secrets in environment variables!** They are:
- Visible in task definitions
- Visible in CloudWatch logs
- Stored in plaintext in API responses

---

## 5. Security Best Practices

### Container Security

**1. Use Minimal Base Images**
```dockerfile
# ❌ Bad: Large attack surface
FROM ubuntu:20.04

# ✅ Good: Minimal image
FROM alpine:3.18

# ✅ Better: Distroless (no shell, no package manager)
FROM gcr.io/distroless/static-debian11
```

**2. Run as Non-Root User**
```dockerfile
FROM node:18-alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

USER nodejs

CMD ["node", "server.js"]
```

**3. Scan Images for Vulnerabilities**
```bash
# Enable ECR scanning
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true

# Or use Trivy locally
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image my-app:latest
```

### Network Security

**1. Use Private Subnets**
```bash
# Place tasks in private subnets
--network-configuration '{
  "awsvpcConfiguration": {
    "subnets": ["subnet-private-1", "subnet-private-2"],
    "assignPublicIp": "DISABLED"
  }
}'
```

**2. Principle of Least Privilege (Security Groups)**
```bash
# Allow only necessary traffic
aws ec2 authorize-security-group-ingress \
  --group-id sg-task \
  --protocol tcp \
  --port 80 \
  --source-group sg-alb  # Only from ALB, not 0.0.0.0/0
```

**3. Enable VPC Flow Logs**
```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-abc123 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs
```

### IAM Security

**1. Separate Roles per Service**
```
Don't: One task role for all services
Do: web-service-role, api-service-role, worker-service-role
```

**2. Use IAM Policies with Conditions**
```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "IpAddress": {
      "aws:SourceIp": "10.0.0.0/16"
    }
  }
}
```

**3. Rotate Credentials**
```bash
# Use Secrets Manager auto-rotation
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/db-password \
  --rotation-lambda-arn arn:aws:lambda:...:function:RotateSecret
```

---

## 6. Compliance and Auditing

### AWS CloudTrail

Log all ECS API calls:

```bash
aws cloudtrail create-trail \
  --name ecs-audit-trail \
  --s3-bucket-name my-audit-logs

aws cloudtrail start-logging --name ecs-audit-trail
```

**Events tracked:**
- `CreateCluster`
- `CreateService`
- `RegisterTaskDefinition`
- `UpdateService`
- `RunTask`

### AWS Config

Monitor ECS resource compliance:

```bash
# Enable Config for ECS
aws configservice put-configuration-recorder \
  --configuration-recorder name=ecs-config,roleARN=arn:aws:iam::123456789012:role/ConfigRole

# Create rule: Tasks must use task roles
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ecs-task-role-required",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "ECS_TASK_DEFINITION_NONROOT_USER"
    }
  }'
```

### Logging Best Practices

1. **Enable CloudTrail** for all API calls
2. **Store logs in S3** with lifecycle policies
3. **Encrypt log data** at rest and in transit
4. **Set retention policies** (30-90 days for CloudWatch)
5. **Monitor access** to logs (CloudTrail on CloudWatch Logs)

---

## 7. Performance Monitoring

### Key Metrics to Monitor

**Cluster Metrics:**
- `CPUUtilization`
- `MemoryUtilization`
- `CPUReservation`
- `MemoryReservation`

**Service Metrics:**
- `ServiceCPU`
- `ServiceMemory`
- `RunningTaskCount`
- `TargetResponseTime` (ALB)

**Custom Metrics:**
```bash
# Publish custom metric from application
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name RequestsProcessed \
  --value 150 \
  --dimensions Service=api,Environment=production
```

### Performance Tuning

**1. Right-Size Tasks**
```bash
# Monitor actual usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=web-service

# Adjust task definition
# If average CPU < 50%, reduce CPU allocation
# If average CPU > 80%, increase CPU allocation
```

**2. Enable Container Insights**
Identify bottlenecks and optimize resource allocation.

**3. Use Application Performance Monitoring (APM)**
- AWS X-Ray for distributed tracing
- New Relic, Datadog, Dynatrace

---

## 8. Alerting and Notifications

### CloudWatch Alarms

**CPU Alarm:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name ecs-high-cpu \
  --alarm-description "Alert when CPU > 80%" \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=web-service Name=ClusterName,Value=production \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ecs-alerts
```

**Service Health Alarm:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name ecs-unhealthy-tasks \
  --namespace AWS/ECS \
  --metric-name RunningTaskCount \
  --dimensions Name=ServiceName,Value=web-service \
  --statistic Average \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ecs-alerts
```

### SNS Notifications

```bash
# Create SNS topic
aws sns create-topic --name ecs-alerts

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:ecs-alerts \
  --protocol email \
  --notification-endpoint ops-team@company.com
```

### EventBridge Rules

**Alert on task failures:**
```bash
aws events put-rule \
  --name ecs-task-stopped \
  --event-pattern '{
    "source": ["aws.ecs"],
    "detail-type": ["ECS Task State Change"],
    "detail": {
      "lastStatus": ["STOPPED"],
      "stoppedReason": [{"prefix": ""}]
    }
  }'

aws events put-targets \
  --rule ecs-task-stopped \
  --targets "Id=1,Arn=arn:aws:sns:us-east-1:123456789012:ecs-alerts"
```

---

## 9. Cost Monitoring

### Enable Cost Allocation Tags

```bash
# Tag all resources
aws ecs create-service \
  --tags "key=Environment,value=production" "key=CostCenter,value=engineering" \
  ...
```

### View Costs in Cost Explorer

- Filter by: Service = ECS
- Group by: Tag (Environment, CostCenter)
- Time range: Last 30 days

### Cost Optimization

**1. Use Fargate Spot**
Save 70% on non-critical workloads.

**2. Right-Size Tasks**
Don't over-provision CPU/memory.

**3. Use EC2 Spot for Long-Running Workloads**
Save 70-90% compared to On-Demand.

**4. Delete Unused Resources**
- Old task definitions
- Stopped tasks
- Unused ECR images

**5. Set CloudWatch Logs Retention**
```bash
aws logs put-retention-policy \
  --log-group-name /ecs/my-app \
  --retention-in-days 7  # Instead of "Never expire"
```

---

## 10. Security Scanning

### ECR Image Scanning

```bash
# Enable scan on push
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true

# View scan results
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=latest
```

### Third-Party Scanners

**Trivy:**
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image my-app:latest
```

**Clair:**
```bash
clairctl analyze my-app:latest
```

### Runtime Security

**AWS GuardDuty for ECS:**
Monitors runtime behavior for threats.

**Falco (Open Source):**
Runtime security monitoring for containers.

---

## Quick Security Checklist

- [ ] Use IAM roles (no hardcoded credentials)
- [ ] Store secrets in Secrets Manager
- [ ] Run containers as non-root
- [ ] Use minimal base images
- [ ] Enable ECR image scanning
- [ ] Place tasks in private subnets
- [ ] Use task-level security groups (awsvpc)
- [ ] Enable CloudTrail logging
- [ ] Enable VPC Flow Logs
- [ ] Set CloudWatch Logs retention
- [ ] Enable Container Insights
- [ ] Configure CloudWatch Alarms
- [ ] Tag all resources for cost tracking
- [ ] Regularly update base images
- [ ] Review IAM policies (least privilege)

---

**End of Part 8 — Monitoring, Logging, and Security Best Practices**

**You've completed all 8 parts of the ECS/ECR guide!**
