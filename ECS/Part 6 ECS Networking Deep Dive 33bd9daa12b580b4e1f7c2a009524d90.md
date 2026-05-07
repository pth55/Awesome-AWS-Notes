# Part 6: ECS Networking Deep Dive

## Quick Summary
Comprehensive guide to ECS networking covering network modes, security groups, service discovery, load balancing, and troubleshooting.

## Network Modes Comparison

| Mode | Used By | Security Groups | IP Address | Port Mapping |
|:-----|:--------|:----------------|:-----------|:-------------|
| **awsvpc** | Fargate (required), EC2 | Per-task | Task gets own IP | Static |
| **bridge** | EC2 only | Per-instance | Container uses bridge IP | Dynamic (32768-65535) |
| **host** | EC2 only | Per-instance | Uses host IP directly | Static (no isolation) |
| **none** | EC2 only | None | No network | N/A |

## awsvpc Mode Deep Dive

Each task gets its own Elastic Network Interface (ENI).

**Configuration:**
```bash
aws ecs create-service \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-private-1", "subnet-private-2"],
      "securityGroups": ["sg-task"],
      "assignPublicIp": "DISABLED"
    }
  }'
```

**Benefits:**
- Task-level security groups (fine-grained control)
- Each task has unique private IP
- VPC Flow Logs per task
- Required for Fargate

**Limitations:**
- ENI quota limits (t3.medium = 3 ENIs max)
- Consumes IPs from subnet

## Security Groups for Tasks

### Layered Security Model

```
Internet
  ↓
ALB Security Group (sg-alb)
  → Allow: 0.0.0.0/0 on port 80/443
  ↓
Web Task Security Group (sg-web)
  → Allow: sg-alb on port 80
  ↓
API Task Security Group (sg-api)
  → Allow: sg-web on port 3000
  ↓
Database Task Security Group (sg-db)
  → Allow: sg-api on port 5432
```

**Creating task security group:**
```bash
aws ec2 create-security-group \
  --group-name task-sg \
  --description "Security group for ECS tasks" \
  --vpc-id vpc-abc123

# Allow traffic from ALB
aws ec2 authorize-security-group-ingress \
  --group-id sg-task \
  --protocol tcp \
  --port 80 \
  --source-group sg-alb
```

## Service Discovery

AWS Cloud Map provides DNS-based service discovery.

**Setup:**
```bash
# 1. Create private namespace
aws servicediscovery create-private-dns-namespace \
  --name internal.local \
  --vpc vpc-abc123

# 2. Create ECS service with discovery
aws ecs create-service \
  --service-registries '[{
    "registryArn": "arn:aws:servicediscovery:...:service/api"
  }]'
```

**Result:**
```
Tasks automatically register as:
  api.internal.local → 10.0.1.10, 10.0.1.11, 10.0.1.12

From other tasks:
  curl http://api.internal.local:3000
```

## Load Balancing

### Application Load Balancer (ALB)

**For**: HTTP/HTTPS traffic

```bash
# Create target group (IP type for awsvpc)
aws elbv2 create-target-group \
  --name web-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-abc \
  --target-type ip \
  --health-check-path /health
```

### Network Load Balancer (NLB)

**For**: TCP/UDP, high performance, static IP

```bash
aws elbv2 create-target-group \
  --name tcp-tg \
  --protocol TCP \
  --port 8080 \
  --target-type ip
```

## Private vs Public Tasks

### Private Tasks (Recommended)

**Architecture:**
```
Internet → ALB (public subnets) → Tasks (private subnets)
         → NAT Gateway → Internet (for outbound)
```

**Benefits:**
- Tasks not directly exposed to internet
- Better security posture
- Access only through ALB

**Configuration:**
```bash
--network-configuration '{
  "awsvpcConfiguration": {
    "subnets": ["subnet-private-1", "subnet-private-2"],
    "assignPublicIp": "DISABLED"
  }
}'
```

## Troubleshooting

### Cannot Pull Images

**Issue**: CannotPullContainerError

**Solutions:**
1. Private subnet needs NAT Gateway OR VPC Endpoints
2. Task Execution Role needs ECR permissions
3. Check security groups allow outbound HTTPS

### Health Checks Failing

**Check:**
```bash
# 1. Security group allows ALB → Task
# 2. Application listens on correct port
# 3. Health check path returns 200 OK

aws elbv2 describe-target-health --target-group-arn arn:...
```

### Service Discovery Not Working

**Verify:**
```bash
# From inside task container
nslookup api.internal.local
dig api.internal.local
```

**Common causes:**
- DNS not enabled in VPC
- Security groups blocking traffic
- Service not registered yet

---

**See Also:**
- Part 4: EC2 Launch Type (bridge mode examples)
- Part 5: Fargate (awsvpc required)
- Part 7: Storage and Persistence
