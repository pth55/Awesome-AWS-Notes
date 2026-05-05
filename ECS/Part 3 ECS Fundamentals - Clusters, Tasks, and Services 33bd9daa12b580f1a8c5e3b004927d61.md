# Part 3: ECS Fundamentals - Clusters, Tasks, and Services

---

## Table of Contents

1. [What is Amazon ECS](#1-what-is-amazon-ecs)
2. [ECS Architecture Overview](#2-ecs-architecture-overview)
3. [ECS Clusters Explained](#3-ecs-clusters-explained)
4. [Task Definitions — The Blueprint](#4-task-definitions--the-blueprint)
5. [Tasks vs Services — The Critical Difference](#5-tasks-vs-services--the-critical-difference)
6. [Launch Types: EC2 vs Fargate](#6-launch-types-ec2-vs-fargate)
7. [Container Instances (EC2 Launch Type)](#7-container-instances-ec2-launch-type)
8. [ECS Agent Deep Dive](#8-ecs-agent-deep-dive)
9. [Task Placement Strategies](#9-task-placement-strategies)
10. [Capacity Providers](#10-capacity-providers)
11. [ECS Namespaces and Service Discovery](#11-ecs-namespaces-and-service-discovery)
12. [Quick Reference Architecture Patterns](#12-quick-reference-architecture-patterns)

---

## 1. What is Amazon ECS

Amazon Elastic Container Service (ECS) is a fully managed container orchestration service that makes it easy to deploy, manage, and scale containerized applications.

### What ECS Does

```
Your Responsibilities:
─────────────────────
✓ Write application code
✓ Build Docker image
✓ Push to ECR
✓ Define how containers should run (task definition)
✓ Decide how many containers to run

AWS ECS Handles:
─────────────────────
✓ Scheduling containers onto compute resources
✓ Health monitoring and restart of failed containers
✓ Scaling up/down based on load
✓ Integration with load balancers
✓ Service discovery
✓ Rolling deployments
✓ Log aggregation
```

### Why Use ECS Instead of Managing Docker Manually?

```
Manual Docker on EC2:
─────────────────────
❌ You SSH into each EC2 instance
❌ Manually docker run commands
❌ Monitor containers yourself
❌ Restart failed containers manually
❌ Manually configure load balancer
❌ No automated scaling
❌ No deployment orchestration

With ECS:
─────────────────────
✓ Declare desired state → ECS maintains it
✓ Automatic container placement
✓ Automatic restarts on failure
✓ Integrated with ALB/NLB
✓ Auto scaling built-in
✓ Blue/green and rolling deployments
✓ CloudWatch integration
```

### ECS vs Other Container Orchestrators

| Feature | ECS | EKS (Kubernetes) | Docker Swarm |
|:--------|:----|:-----------------|:-------------|
| **Learning Curve** | Easy | Steep | Medium |
| **AWS Integration** | Native | Good | Manual |
| **Portability** | AWS-only | Cross-cloud | Cross-platform |
| **Complexity** | Low | High | Medium |
| **Community** | AWS docs | Massive | Declining |
| **Cost** | Free (pay for resources) | $0.10/hr per cluster | Free |
| **Use Case** | AWS-native apps | Kubernetes expertise, multi-cloud | Small teams, simple apps |

**When to use ECS:**
- You are all-in on AWS
- You want simplicity over portability
- You need deep AWS service integration (IAM, VPC, CloudWatch, Secrets Manager)
- Your team is small/medium and learning Kubernetes is overkill

**When to use EKS:**
- You need multi-cloud/hybrid cloud portability
- You already have Kubernetes expertise
- You need the Kubernetes ecosystem (Helm, Operators, etc.)
- You have complex orchestration needs

---

## 2. ECS Architecture Overview

### High-Level Components

```
┌──────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD                                 │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    ECS CLUSTER                             │ │
│  │                                                            │ │
│  │  ┌──────────────────────┐  ┌──────────────────────┐      │ │
│  │  │  Container Instance  │  │  Container Instance  │      │ │
│  │  │  (EC2 or Fargate)    │  │  (EC2 or Fargate)    │      │ │
│  │  │                      │  │                      │      │ │
│  │  │  ┌────────────────┐  │  │  ┌────────────────┐  │      │ │
│  │  │  │  Task          │  │  │  │  Task          │  │      │ │
│  │  │  │  ┌──────────┐  │  │  │  │  ┌──────────┐  │  │      │ │
│  │  │  │  │Container │  │  │  │  │  │Container │  │  │      │ │
│  │  │  │  │  (nginx) │  │  │  │  │  │  (node)  │  │  │      │ │
│  │  │  │  └──────────┘  │  │  │  │  └──────────┘  │  │      │ │
│  │  │  └────────────────┘  │  │  └────────────────┘  │      │ │
│  │  └──────────────────────┘  └──────────────────────┘      │ │
│  │                                                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   ECS CONTROL PLANE                        │ │
│  │  (Scheduling, Health Checks, API, State Management)       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │      Integration Layer                                     │ │
│  │  ┌──────┐  ┌──────┐  ┌───────┐  ┌──────────┐  ┌────────┐  │ │
│  │  │ ALB  │  │ ECR  │  │  IAM  │  │CloudWatch│  │Secrets │  │ │
│  │  └──────┘  └──────┘  └───────┘  └──────────┘  └────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### Key Components Explained

**1. Cluster** — Logical grouping of compute resources (EC2 instances or Fargate capacity)

**2. Task Definition** — JSON blueprint describing:
   - Which Docker images to run
   - CPU/memory requirements
   - Environment variables
   - IAM roles
   - Networking mode
   - Logging configuration

**3. Task** — Running instance of a task definition. A task runs one or more containers.

**4. Service** — Maintains a desired number of tasks running and handles:
   - Load balancer integration
   - Auto scaling
   - Rolling deployments
   - Health checks

**5. Container Instance** — EC2 instance running the ECS agent (EC2 launch type only)

**6. ECS Agent** — Software running on container instances that communicates with ECS control plane

**7. Control Plane** — AWS-managed component that handles scheduling, monitoring, and API requests

---

## 3. ECS Clusters Explained

### What is a Cluster?

A cluster is a **logical grouping** of tasks or services. It is NOT a physical resource — it is just a namespace/boundary.

```
Cluster = Named boundary that contains:
  ├── EC2 instances (if using EC2 launch type)
  ├── Fargate tasks (if using Fargate launch type)
  ├── Services
  └── Capacity providers
```

### Creating a Cluster

#### Console:

```
ECS Console → Clusters → Create Cluster

Settings:
─────────────────────────────────────
Cluster name:     production-cluster
Infrastructure:   AWS Fargate (serverless)
                  OR
                  Amazon EC2 instances

Monitoring:       CloudWatch Container Insights (optional)
Tags:             Environment: production
─────────────────────────────────────
```

#### AWS CLI:

```bash
# Create empty cluster (for Fargate)
aws ecs create-cluster \
  --cluster-name production-cluster \
  --region us-east-1

# Output:
{
  "cluster": {
    "clusterArn": "arn:aws:ecs:us-east-1:123456789012:cluster/production-cluster",
    "clusterName": "production-cluster",
    "status": "ACTIVE",
    "registeredContainerInstancesCount": 0,
    "runningTasksCount": 0,
    "pendingTasksCount": 0
  }
}
```

### Cluster Types

**1. Fargate Cluster**
```
No EC2 instances to manage
Just logical grouping for Fargate tasks
```

**2. EC2 Cluster**
```
You provision EC2 instances
Register them to the cluster
ECS schedules containers onto them
```

**3. Hybrid (Both EC2 + Fargate)**
```
Same cluster can run both EC2 and Fargate tasks
Use capacity providers to control placement
```

### Cluster Namespace Isolation

```
production-cluster
  ├── web-service (10 tasks)
  ├── api-service (5 tasks)
  └── worker-service (3 tasks)

staging-cluster
  ├── web-service (2 tasks)   ← Same service name, different cluster
  ├── api-service (1 task)
  └── worker-service (1 task)

Clusters provide logical isolation:
────────────────────────────────────
✓ Services in different clusters cannot see each other (unless networked)
✓ Same service name can exist in multiple clusters
✓ IAM policies can restrict access per cluster
✓ CloudWatch metrics are per cluster
```

### Cluster Limits

| Resource | Default Limit | Can Increase? |
|:---------|:--------------|:--------------|
| Clusters per account per region | 10,000 | Yes |
| Services per cluster | 5,000 | Yes |
| Tasks per service | 10,000 | Yes |
| Container instances per cluster | 5,000 | Yes |
| Tasks per container instance | Depends on instance size | No |

---

## 4. Task Definitions — The Blueprint

A task definition is a JSON document that describes how to run your containers. Think of it as a recipe.

### Task Definition Structure

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/myAppTaskRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0",
      "cpu": 256,
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password-abc123"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ]
}
```

### Key Fields Explained

**family** — Name of the task definition. Each revision creates a new version:
```
my-app:1
my-app:2
my-app:3
```

**networkMode** — How containers connect to networking:
- `awsvpc` — Each task gets its own ENI (Elastic Network Interface) with its own private IP (required for Fargate)
- `bridge` — Docker bridge networking (default for EC2 launch type)
- `host` — Container uses host's network directly (no isolation)
- `none` — No networking

**requiresCompatibilities** — Which launch types can run this task:
- `["EC2"]` — Only EC2
- `["FARGATE"]` — Only Fargate
- `["EC2", "FARGATE"]` — Both

**cpu** — Task-level CPU units (1024 units = 1 vCPU):
```
Fargate requires specific cpu/memory combinations:
256  (.25 vCPU) → memory: 512 MB - 2 GB
512  (.5 vCPU)  → memory: 1 GB - 4 GB
1024 (1 vCPU)   → memory: 2 GB - 8 GB
2048 (2 vCPU)   → memory: 4 GB - 16 GB
4096 (4 vCPU)   → memory: 8 GB - 30 GB
```

**memory** — Task-level memory in MB

**executionRoleArn** — IAM role for ECS agent to:
- Pull images from ECR
- Write logs to CloudWatch
- Retrieve secrets from Secrets Manager/Parameter Store

**taskRoleArn** — IAM role for containers to:
- Access AWS services (S3, DynamoDB, etc.) from inside the container
- This is what your application uses

**containerDefinitions** — Array of containers in the task:

- **name** — Container name (must be unique within the task)
- **image** — Docker image URI
- **essential** — If true and this container stops, all other containers in the task are stopped
- **portMappings** — Which ports to expose
- **environment** — Environment variables (plain text)
- **secrets** — Environment variables from Secrets Manager/Parameter Store (encrypted)
- **logConfiguration** — Where to send logs

### Task Definition Revisions

Every time you update a task definition, ECS creates a new **revision**:

```
Revision 1: my-app:1  (image: my-app:v1.0.0)
Revision 2: my-app:2  (image: my-app:v1.1.0)  ← changed image version
Revision 3: my-app:3  (image: my-app:v1.1.0, cpu: 512)  ← changed CPU

Active services can reference any revision:
────────────────────────────────────────────
Service A → my-app:3  (latest)
Service B → my-app:1  (old version still running for testing)
```

Revisions are immutable — you cannot edit revision 1. You register a new revision and update the service to use it.

### Registering a Task Definition

```bash
# Save task definition JSON to file
cat > task-definition.json <<EOF
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
EOF

# Register the task definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region us-east-1

# Output shows the new revision:
{
  "taskDefinition": {
    "taskDefinitionArn": "arn:aws:ecs:us-east-1:123456789012:task-definition/my-app:1",
    "family": "my-app",
    "revision": 1,
    ...
  }
}
```

### Multiple Containers in One Task

A task can run multiple containers that share:
- The same network namespace (localhost communication)
- The same volumes
- Lifecycle (start/stop together)

```json
{
  "family": "app-with-sidecar",
  "containerDefinitions": [
    {
      "name": "main-app",
      "image": "my-app:v1.0",
      "essential": true,
      "portMappings": [{"containerPort": 80}]
    },
    {
      "name": "log-router",
      "image": "fluentbit:latest",
      "essential": false,
      "dependsOn": [
        {
          "containerName": "main-app",
          "condition": "START"
        }
      ]
    }
  ]
}
```

Use cases for multiple containers:
- **Sidecar pattern:** Logging agent, monitoring agent
- **Ambassador pattern:** Proxy for external services
- **Adapter pattern:** Transform data format before sending to main app

---

## 5. Tasks vs Services — The Critical Difference

### Task

A **task** is a running instance of a task definition. It is ephemeral.

```
Analogy: A task is like running `docker run` manually
         You start it, it runs, it stops, it's gone.

Use cases:
──────────
✓ Batch jobs
✓ One-time tasks
✓ Cron jobs (run task on schedule)
✓ CI/CD build agents
```

**Running a standalone task:**

```bash
aws ecs run-task \
  --cluster production-cluster \
  --task-definition my-app:1 \
  --count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-xyz789],assignPublicIp=ENABLED}" \
  --region us-east-1
```

This starts a single task. When the container exits, the task stops. ECS does NOT restart it.

### Service

A **service** maintains a desired number of tasks running at all times. It is long-running.

```
Analogy: A service is like systemd or a Kubernetes Deployment
         You declare "I want 3 copies running" and ECS maintains that state

ECS automatically:
──────────────────
✓ Starts tasks
✓ Restarts failed tasks
✓ Integrates with load balancer
✓ Handles rolling deployments
✓ Scales up/down

Use cases:
──────────
✓ Web servers
✓ APIs
✓ Long-running workers
✓ Microservices
```

**Creating a service:**

```bash
aws ecs create-service \
  --cluster production-cluster \
  --service-name web-service \
  --task-definition my-app:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123,subnet-def456],securityGroups=[sg-xyz789],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-tg/abc123,containerName=web,containerPort=80" \
  --region us-east-1
```

ECS will:
1. Start 3 tasks
2. Register them with the target group (ALB)
3. Monitor health
4. Restart any that fail
5. Maintain 3 running tasks forever

### Task vs Service Comparison

| Feature | Task | Service |
|:--------|:-----|:--------|
| **Lifecycle** | Ephemeral (run once, then stop) | Long-running (always maintains desired count) |
| **Auto-restart on failure** | No | Yes |
| **Load balancer integration** | No | Yes (automatic registration/deregistration) |
| **Auto scaling** | No | Yes |
| **Rolling deployments** | N/A | Yes |
| **Use Case** | Batch jobs, one-time tasks | Web servers, APIs, microservices |

---

## 6. Launch Types: EC2 vs Fargate

ECS supports two launch types: **EC2** and **Fargate**. The choice fundamentally changes who manages the infrastructure.

### EC2 Launch Type

**You provision EC2 instances → ECS schedules containers onto them**

```
┌─────────────────────────────────────────┐
│            ECS CLUSTER                  │
│                                         │
│  ┌────────────────┐  ┌────────────────┐│
│  │ EC2 Instance   │  │ EC2 Instance   ││
│  │ (t3.medium)    │  │ (t3.medium)    ││
│  │                │  │                ││
│  │  ┌──────────┐  │  │  ┌──────────┐  ││
│  │  │ Task 1   │  │  │  │ Task 3   │  ││
│  │  └──────────┘  │  │  └──────────┘  ││
│  │  ┌──────────┐  │  │                ││
│  │  │ Task 2   │  │  │                ││
│  │  └──────────┘  │  │                ││
│  └────────────────┘  └────────────────┘│
└─────────────────────────────────────────┘

You manage:
───────────
✓ EC2 instance type selection
✓ AMI (Amazon Machine Image)
✓ Auto Scaling Group
✓ Security groups
✓ SSH access (optional)
✓ OS patching

ECS manages:
───────────
✓ Which tasks go on which instances
✓ Starting/stopping containers
✓ Health monitoring
```

**Pros:**
- Full control over instance type and size
- Can use Spot instances (70-90% cheaper)
- Can SSH into instances for debugging
- Better for large, stable workloads (cost-effective at scale)
- Can use reserved instances for cost savings

**Cons:**
- You manage the instances (patching, scaling, monitoring)
- Must plan capacity (what if all instances are full?)
- Undifferentiated heavy lifting

### Fargate Launch Type

**AWS provisions and manages compute → You just define tasks**

```
┌─────────────────────────────────────────┐
│            ECS CLUSTER                  │
│                                         │
│     ┌──────────┐  ┌──────────┐         │
│     │ Task 1   │  │ Task 2   │         │
│     │ (Fargate)│  │ (Fargate)│         │
│     └──────────┘  └──────────┘         │
│                                         │
│  [AWS manages compute infrastructure]  │
└─────────────────────────────────────────┘

You manage:
───────────
✓ Task definition (cpu/memory requirements)
✓ Networking (VPC, subnets, security groups)

AWS manages:
───────────
✓ Provisioning servers
✓ Patching
✓ Scaling
✓ Right-sizing
✓ Everything infrastructure-related
```

**Pros:**
- Serverless — no servers to manage
- Pay per task, per second (no idle capacity costs)
- Automatic scaling
- Simpler operational model

**Cons:**
- More expensive per vCPU/GB compared to EC2
- Less control (no SSH, fixed instance types)
- Cold start time (3-5 seconds to launch new tasks)
- Limited CPU/memory combinations

### Cost Comparison Example

**Scenario:** Run a web app needing 1 vCPU, 2 GB RAM, 24/7

**Fargate:**
```
Price: $0.04048/vCPU/hour + $0.004445/GB/hour

1 vCPU:  $0.04048 × 730 hours = $29.55/month
2 GB:    $0.004445 × 2 GB × 730 hours = $6.49/month
Total:   $36.04/month
```

**EC2 (t3.medium: 2 vCPUs, 4 GB RAM):**
```
On-Demand: $0.0416/hour × 730 hours = $30.37/month
Reserved (1-year): $20/month
Spot (average 70% discount): $9/month

But you have spare capacity: 1 unused vCPU + 2 GB unused RAM
```

**When Fargate is cheaper:**
- Variable workloads (scale to zero during off-hours)
- Small workloads (< 5-10 tasks)
- Development/testing environments

**When EC2 is cheaper:**
- Large, steady workloads (50+ tasks 24/7)
- Can use Reserved Instances or Spot
- Need full instance utilization (multiple tasks per instance)

### Choosing Launch Type

| Use Case | Recommended Launch Type |
|:---------|:----------------------|
| Microservices with variable traffic | Fargate |
| Batch jobs (run, then stop) | Fargate |
| Development/staging environments | Fargate |
| Large-scale production (50+ tasks, 24/7) | EC2 + Spot |
| Need GPU/specialized hardware | EC2 (Fargate doesn't support) |
| Regulatory requirement for dedicated hosts | EC2 Dedicated Hosts |
| Want to avoid infrastructure management | Fargate |
| Cost optimization at scale | EC2 (with Reserved/Spot) |

---

## 7. Container Instances (EC2 Launch Type)

When using EC2 launch type, container instances are the EC2 machines running your containers.

### What is a Container Instance?

An EC2 instance that:
1. Runs the **ECS-optimized AMI** (Amazon Linux with Docker + ECS agent pre-installed)
2. Runs the **ECS agent** (communicates with ECS control plane)
3. Is **registered to an ECS cluster**

### ECS-Optimized AMI

AWS provides pre-built AMIs with:
- Amazon Linux 2023 or Amazon Linux 2
- Docker engine
- ECS agent
- CloudWatch agent
- ecs-init service

```bash
# Find latest ECS-optimized AMI
aws ssm get-parameter \
  --name /aws/service/ecs/optimized-ami/amazon-linux-2023/recommended \
  --region us-east-1 \
  --query 'Parameter.Value' \
  --output text | jq -r '.image_id'

# Output: ami-0c55b159cbfafe1f0
```

### Registering an EC2 Instance to a Cluster

**Method 1: User Data Script (Automatic)**

When launching EC2 instances, use this user data script:

```bash
#!/bin/bash
echo ECS_CLUSTER=production-cluster >> /etc/ecs/ecs.config
echo ECS_ENABLE_TASK_IAM_ROLE=true >> /etc/ecs/ecs.config
echo ECS_ENABLE_TASK_IAM_ROLE_NETWORK_HOST=true >> /etc/ecs/ecs.config
```

The ECS agent reads `/etc/ecs/ecs.config` on startup and registers the instance to the specified cluster.

**Method 2: Auto Scaling Group with Launch Template**

```json
{
  "LaunchTemplateName": "ecs-instance-template",
  "LaunchTemplateData": {
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "t3.medium",
    "IamInstanceProfile": {
      "Arn": "arn:aws:iam::123456789012:instance-profile/ecsInstanceRole"
    },
    "SecurityGroupIds": ["sg-xyz789"],
    "UserData": "IyEvYmluL2Jhc2gKZWNobyBFQ1NfQ0xVU1RFUj1wcm9kdWN0aW9uLWNsdXN0ZXIgPj4gL2V0Yy9lY3MvZWNzLmNvbmZpZw==",
    "TagSpecifications": [
      {
        "ResourceType": "instance",
        "Tags": [
          {"Key": "Name", "Value": "ECS Container Instance"}
        ]
      }
    ]
  }
}
```

### Instance IAM Role (ecsInstanceRole)

Container instances need an IAM role with permissions to:
- Register with ECS cluster
- Pull images from ECR
- Write logs to CloudWatch
- Retrieve secrets (optional)

**Managed policy to attach:**
```
AmazonEC2ContainerServiceforEC2Role
```

### Task Placement on EC2 Instances

ECS scheduler places tasks on instances based on:
- Available CPU/memory
- Task placement strategies (spread, binpack, random)
- Task placement constraints (instance type, AZ, custom attributes)

```
Example:
────────
Instance 1: t3.medium (2 vCPUs, 4 GB RAM)
  Available: 1.5 vCPUs, 3 GB RAM
  Running:  Task A (0.5 vCPU, 1 GB)

Instance 2: t3.medium (2 vCPUs, 4 GB RAM)
  Available: 2 vCPUs, 4 GB RAM
  Running:  Nothing

New task needs: 1 vCPU, 2 GB RAM

ECS places on Instance 2 (more available resources)
```

---

## 8. ECS Agent Deep Dive

The ECS agent is the software running on container instances that communicates with ECS control plane.

### What the Agent Does

```
1. Registers instance to cluster
2. Receives task placement instructions from ECS
3. Starts/stops Docker containers
4. Monitors container health
5. Reports metrics to ECS (CPU, memory, network)
6. Handles task IAM roles
7. Pulls images from ECR
```

### Agent Configuration File

`/etc/ecs/ecs.config` — Configuration options:

```bash
# Cluster assignment
ECS_CLUSTER=production-cluster

# Enable task IAM roles (required for task roles)
ECS_ENABLE_TASK_IAM_ROLE=true
ECS_ENABLE_TASK_IAM_ROLE_NETWORK_HOST=true

# Container instance attributes (for task placement constraints)
ECS_INSTANCE_ATTRIBUTES={"environment": "production", "tier": "backend"}

# Logging
ECS_LOGLEVEL=info
ECS_LOGFILE=/log/ecs-agent.log

# Disable privileged containers (security hardening)
ECS_DISABLE_PRIVILEGED=true

# Image cleanup
ECS_IMAGE_CLEANUP_INTERVAL=30m
ECS_IMAGE_MINIMUM_CLEANUP_AGE=1h
ECS_NUM_IMAGES_DELETE_PER_CYCLE=5
```

### Updating the ECS Agent

```bash
# Check current agent version
curl -s http://localhost:51678/v1/metadata | jq -r '.Version'

# Update agent (for Amazon Linux 2)
sudo yum update -y ecs-init

# Restart agent
sudo systemctl restart ecs
```

### Agent Introspection API

The agent exposes a local API on port 51678:

```bash
# Get instance metadata
curl http://localhost:51678/v1/metadata

# Get running tasks
curl http://localhost:51678/v1/tasks

# Get task metadata (from inside a container)
curl $ECS_CONTAINER_METADATA_URI_V4/task
```

---

## 9. Task Placement Strategies

When using EC2 launch type, ECS uses placement strategies to decide which instance to place a task on.

### Placement Strategy Types

**1. Binpack** — Pack tasks onto instances to minimize number of instances used

```
Goal: Maximize resource utilization, minimize instance count

Strategy:
  Place task on instance with LEAST available CPU or memory

Use case: Cost optimization (leave some instances idle so you can terminate them)
```

```json
{
  "placementStrategy": [
    {
      "type": "binpack",
      "field": "memory"
    }
  ]
}
```

**2. Spread** — Distribute tasks evenly across instances

```
Goal: High availability, even distribution

Strategy:
  Place tasks evenly across instances or AZs

Use case: Fault tolerance (if one instance/AZ fails, impact is minimized)
```

```json
{
  "placementStrategy": [
    {
      "type": "spread",
      "field": "attribute:ecs.availability-zone"
    }
  ]
}
```

**3. Random** — Place tasks randomly

```
Goal: Simple, no specific optimization

Strategy:
  Random placement

Use case: When you don't care about optimization
```

```json
{
  "placementStrategy": [
    {
      "type": "random"
    }
  ]
}
```

### Placement Constraints

Force tasks to run only on instances meeting certain conditions:

**distinctInstance** — Each task runs on a different instance

```json
{
  "placementConstraints": [
    {
      "type": "distinctInstance"
    }
  ]
}
```

**memberOf** — Expression-based constraint

```json
{
  "placementConstraints": [
    {
      "type": "memberOf",
      "expression": "attribute:ecs.instance-type =~ t3.*"
    }
  ]
}
```

Examples:
```
attribute:ecs.availability-zone == us-east-1a
attribute:ecs.instance-type =~ t3.*
attribute:environment == production
attribute:ecs.ami-id == ami-abc123
```

---

## 10. Capacity Providers

Capacity providers are a way to manage compute capacity for ECS clusters. They replace the legacy `launchType` field with a more flexible model.

### Why Capacity Providers?

```
Old way (launch type):
───────────────────────
Service → launch type: EC2 or Fargate

Problem:
  Can't mix EC2 and Fargate in one service
  Can't use Spot and On-Demand in one service

New way (capacity providers):
───────────────────────────────
Service → capacity provider strategy
  → 70% on Fargate Spot
  → 30% on Fargate On-Demand

Benefits:
  ✓ Mix launch types
  ✓ Cost optimization (use Spot for cost, On-Demand for stability)
  ✓ Cluster Auto Scaling (automatic EC2 scaling)
```

### Capacity Provider Types

**1. Fargate Capacity Providers**

```
FARGATE           → Fargate On-Demand
FARGATE_SPOT      → Fargate Spot (70% cheaper, can be interrupted)
```

**2. EC2 Auto Scaling Group Capacity Providers**

```
Link an ECS capacity provider to an Auto Scaling Group
→ ECS automatically scales the ASG based on task demand
```

### Example: Fargate with Spot for Cost Optimization

```bash
# Create service with capacity provider strategy
aws ecs create-service \
  --cluster production-cluster \
  --service-name web-service \
  --task-definition my-app:1 \
  --desired-count 10 \
  --capacity-provider-strategy \
    capacityProvider=FARGATE_SPOT,weight=7,base=0 \
    capacityProvider=FARGATE,weight=3,base=2 \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-xyz789]}" \
  --region us-east-1
```

**What this does:**
```
base=2 on FARGATE → Always run at least 2 tasks on Fargate On-Demand (baseline)

Remaining 8 tasks split by weight:
  70% (weight=7) on FARGATE_SPOT → ~6 tasks
  30% (weight=3) on FARGATE → ~2 tasks

Total:
  2 (base) + 2 (weight) = 4 tasks on Fargate On-Demand
  6 tasks on Fargate Spot

Cost savings: ~50% cheaper while maintaining 40% on stable On-Demand capacity
```

---

## 11. ECS Namespaces and Service Discovery

Service discovery allows services to find each other without hard-coding IP addresses.

### AWS Cloud Map Integration

ECS integrates with **AWS Cloud Map** for DNS-based service discovery.

```
Service A (web) needs to call Service B (api)

Without Service Discovery:
  web → http://10.0.1.45:8080   ← hard-coded IP (breaks when task restarts)

With Service Discovery:
  web → http://api.local:8080   ← DNS name (auto-updated when tasks change)
```

### Creating a Service with Service Discovery

```bash
# 1. Create a private DNS namespace
aws servicediscovery create-private-dns-namespace \
  --name local \
  --vpc vpc-abc123 \
  --region us-east-1

# Output:
{
  "OperationId": "op-abc123"
}

# 2. Get namespace ID
aws servicediscovery get-operation \
  --operation-id op-abc123 \
  --region us-east-1 | jq -r '.Operation.Targets.NAMESPACE'

# Output: ns-xyz789

# 3. Create ECS service with service discovery
aws ecs create-service \
  --cluster production-cluster \
  --service-name api-service \
  --task-definition api:1 \
  --desired-count 2 \
  --service-registries "registryArn=arn:aws:servicediscovery:us-east-1:123456789012:service/srv-abc123" \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-xyz789]}" \
  --region us-east-1
```

**What happens:**

```
1. ECS creates Cloud Map service: api.local
2. When tasks start, ECS registers them in Cloud Map:
   api.local → 10.0.1.10
   api.local → 10.0.1.11

3. DNS queries return all healthy task IPs (round-robin)
4. When tasks stop, ECS deregisters them automatically
```

### Service Discovery Naming

```
[service-name].[namespace]

Examples:
─────────
api.local          → private namespace "local"
web.production.local
auth.staging.local
```

---

## 12. Quick Reference Architecture Patterns

### Pattern 1: Three-Tier Web Application

```
┌──────────────────────────────────────────────────────────────┐
│                    Application Load Balancer                 │
└────────────────────┬─────────────────────────────────────────┘
                     │
      ┌──────────────┴──────────────┐
      │                             │
┌─────▼──────┐              ┌───────▼────┐
│ Web Service│              │ Web Service│   (ECS Service, 3 tasks)
│   Task     │              │   Task     │   (Fargate)
└─────┬──────┘              └───────┬────┘
      │                             │
      └──────────────┬──────────────┘
                     │
             ┌───────▼────────┐
             │  API Service   │   (ECS Service, 2 tasks)
             │                │   (Fargate)
             └───────┬────────┘
                     │
             ┌───────▼────────┐
             │   RDS Database │
             └────────────────┘
```

### Pattern 2: Microservices with Service Discovery

```
┌─────────────────────────────────────────────────────────────┐
│                    ECS Cluster                              │
│                                                             │
│  ┌─────────────┐      ┌─────────────┐      ┌────────────┐ │
│  │   User      │──────▶   Order     │──────▶  Payment   │ │
│  │  Service    │      │   Service   │      │  Service   │ │
│  │             │      │             │      │            │ │
│  │ user.local  │      │ order.local │      │ payment.local│
│  └─────────────┘      └─────────────┘      └────────────┘ │
│        │                                                    │
│        └────────────────┐                                   │
│                         ▼                                   │
│                  ┌─────────────┐                            │
│                  │  AWS Cloud  │   (Service Discovery)      │
│                  │     Map     │                            │
│                  └─────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 3: Batch Processing

```
CloudWatch Events (Cron)
        │
        ▼
┌───────────────┐
│  ECS Task     │  (Run task on schedule)
│  (Batch Job)  │  (Fargate)
│               │
│ Process data  │
│ from S3, write│
│ results to DB │
└───────────────┘
```

---

**End of Part 3 — ECS Fundamentals**
