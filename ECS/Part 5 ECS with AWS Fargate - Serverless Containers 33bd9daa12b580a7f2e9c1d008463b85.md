# Part 5: ECS with AWS Fargate - Serverless Containers

## Quick Summary

**Part 5 Complete Coverage:**
- Fargate fundamentals and architecture
- Task definitions for Fargate (awsvpc network mode required)
- Fargate vs EC2 cost comparison and when to use each
- Platform versions and what they mean
- Fargate Spot for 70% cost savings
- Complete hands-on: Deploy app to Fargate with ALB
- Networking configuration (ENI per task)
- Storage options (ephemeral storage up to 200GB)
- Monitoring and logging best practices
- Troubleshooting Fargate-specific issues

## Key Concepts Covered

### What is Fargate?
AWS Fargate is a **serverless compute engine** for containers. You define your application in a task definition, and Fargate handles all the infrastructure.

**Traditional EC2 Launch Type:**
```
You manage: EC2 instances, patching, scaling, capacity planning
AWS manages: Container orchestration
```

**Fargate:**
```
You manage: Task definitions (what to run)
AWS manages: Everything else (compute, scaling, patching, capacity)
```

### Fargate Architecture

```
┌──────────────────────────────────────────────────────┐
│                  AWS Account                         │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │             ECS Cluster (Fargate)              │ │
│  │                                                │ │
│  │  ┌──────────────┐      ┌──────────────┐       │ │
│  │  │ Fargate Task │      │ Fargate Task │       │ │
│  │  │              │      │              │       │ │
│  │  │  ┌────────┐  │      │  ┌────────┐  │       │ │
│  │  │  │Container│ │      │  │Container│ │       │ │
│  │  │  │  (App)  │  │      │  │  (App)  │  │       │ │
│  │  │  └────────┘  │      │  └────────┘  │       │ │
│  │  │              │      │              │       │ │
│  │  │  ENI         │      │  ENI         │       │ │
│  │  │  Private IP  │      │  Private IP  │       │ │
│  │  └──────────────┘      └──────────────┘       │ │
│  │                                                │ │
│  │  AWS-Managed Infrastructure (invisible to you)│ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

### Task Definition Requirements for Fargate

**Key Differences from EC2:**

1. **Network Mode**: MUST be `awsvpc`
2. **CPU/Memory**: Must use specific valid combinations
3. **Task-level** resources (not container-level)

Valid CPU/Memory combinations:
| CPU (vCPU) | Memory (GB) |
|:-----------|:------------|
| 0.25 | 0.5, 1, 2 |
| 0.5 | 1, 2, 3, 4 |
| 1 | 2, 3, 4, 5, 6, 7, 8 |
| 2 | 4-16 (1 GB increments) |
| 4 | 8-30 (1 GB increments) |

### Hands-On: Deploy to Fargate

```bash
# Create task definition for Fargate
cat > fargate-task.json <<EOF
{
  "family": "fargate-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "nginx:latest",
      "portMappings": [{
        "containerPort": 80,
        "protocol": "tcp"
      }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fargate-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ]
}
EOF

# Register task definition
aws ecs register-task-definition --cli-input-json file://fargate-task.json

# Create Fargate service
aws ecs create-service \
  --cluster production-cluster \
  --service-name fargate-web \
  --task-definition fargate-app:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc,subnet-def],securityGroups=[sg-xyz],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=web,containerPort=80"
```

### Fargate Spot

Save 70% by using spare AWS capacity:

```bash
# Use capacity provider strategy
aws ecs create-service \
  --cluster production-cluster \
  --service-name fargate-spot-web \
  --task-definition fargate-app:1 \
  --desired-count 4 \
  --capacity-provider-strategy \
    "capacityProvider=FARGATE_SPOT,weight=70,base=1" \
    "capacityProvider=FARGATE,weight=30,base=1"
```

This runs:
- Base: 1 task on Fargate + 1 on Fargate Spot (always)
- Remaining 2 tasks: 70% Spot, 30% On-Demand

### Cost Comparison

**Scenario**: Web app needing 1 vCPU, 2 GB RAM

**Fargate On-Demand:**
```
$0.04048/vCPU-hour + $0.004445/GB-hour
= $0.04048 + ($0.004445 × 2) = $0.0494/hour
= $36.04/month (24/7)
```

**Fargate Spot:**
```
70% discount = $0.0148/hour
= $10.81/month (24/7)
```

**EC2 t3.medium (2 vCPU, 4 GB):**
```
On-Demand: $30.37/month
Reserved: $20/month
Spot: ~$9/month
```

**When to use Fargate:**
- Variable workloads (scale to zero)
- Don't want infrastructure management
- Development/test environments
- Microservices with unpredictable traffic

**When to use EC2:**
- Large steady workloads (>10 tasks 24/7)
- GPU/specialized hardware needed
- Tighter control over instance types
- Cost optimization at scale

### Platform Versions

Fargate has platform versions (like 1.4.0, 1.3.0):

**Latest (1.4.0)** includes:
- Larger ephemeral storage (200 GB)
- Faster task launch times
- EFS integration
- Windows container support

Always use `LATEST` unless you need a specific version for compatibility.

### Storage in Fargate

**Ephemeral Storage:**
- Default: 20 GB
- Maximum: 200 GB (platform version 1.4.0+)
- Deleted when task stops

```json
{
  "ephemeralStorage": {
    "sizeInGiB": 50
  }
}
```

**Persistent Storage (EFS):**
See Part 7 for EFS integration.

### Monitoring Fargate

Enable Container Insights:
```bash
aws ecs update-cluster-settings \
  --cluster production-cluster \
  --settings name=containerInsights,value=enabled
```

Metrics:
- CPU/Memory utilization
- Network throughput
- Task count
- Target tracking

### Troubleshooting

**Issue: Task fails to start**
```bash
# Check stopped tasks
aws ecs describe-tasks --cluster CLUSTER --tasks TASK_ARN
```

Common causes:
- Invalid CPU/memory combination
- Missing IAM permissions
- Image pull failure
- Subnet has no available IPs

**Issue: Cannot reach task**
- Security group not allowing inbound traffic
- Task in private subnet without NAT Gateway
- Target group health checks failing

---

**Part 5 covers all Fargate essentials. For detailed networking, see Part 6. For storage options, see Part 7.**
