# Part 4: Building ECS with EC2 Launch Type - Hands-On

---

## Table of Contents

1. [Overview - What We're Building](#1-overview--what-were-building)
2. [Prerequisites and Setup](#2-prerequisites-and-setup)
3. [Step 1: Create VPC and Networking](#3-step-1-create-vpc-and-networking)
4. [Step 2: Create IAM Roles](#4-step-2-create-iam-roles)
5. [Step 3: Create ECS Cluster](#5-step-3-create-ecs-cluster)
6. [Step 4: Create Launch Template](#6-step-4-create-launch-template)
7. [Step 5: Create Auto Scaling Group](#7-step-5-create-auto-scaling-group)
8. [Step 6: Register Task Definition](#8-step-6-register-task-definition)
9. [Step 7: Create Application Load Balancer](#9-step-7-create-application-load-balancer)
10. [Step 8: Create ECS Service](#10-step-8-create-ecs-service)
11. [Step 9: Test and Verify](#11-step-9-test-and-verify)
12. [Step 10: Enable Auto Scaling](#12-step-10-enable-auto-scaling)
13. [Cleanup - Delete All Resources](#13-cleanup--delete-all-resources)
14. [Troubleshooting Common Issues](#14-troubleshooting-common-issues)

---

## 1. Overview - What We're Building

We will build a complete production-ready ECS cluster with EC2 launch type running a simple web application behind an Application Load Balancer.

### Architecture Diagram

```
                        Internet
                           ↓
                    ┌──────────────┐
                    │     ALB      │ (Application Load Balancer)
                    └──────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           ↓               ↓               ↓
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │   AZ-1a  │    │   AZ-1b  │    │   AZ-1c  │
    │          │    │          │    │          │
    │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │
    │ │ EC2  │ │    │ │ EC2  │ │    │ │ EC2  │ │
    │ │ ECS  │ │    │ │ ECS  │ │    │ │ ECS  │ │
    │ │ Task │ │    │ │ Task │ │    │ │ Task │ │
    │ └──────┘ │    │ └──────┘ │    │ └──────┘ │
    └──────────┘    └──────────┘    └──────────┘
         │                │                │
         └────────────────┼────────────────┘
                          ↓
                    Auto Scaling Group
                    (Min: 2, Max: 6)
```

### What You'll Learn

- Create ECS-optimized EC2 instances
- Configure Auto Scaling Groups for ECS
- Register task definitions with proper IAM roles
- Integrate with Application Load Balancer
- Enable service auto scaling
- Implement health checks and monitoring

### Components We'll Create

| Component | Purpose |
|:----------|:--------|
| VPC with subnets | Network isolation |
| ECS Cluster | Logical grouping |
| Launch Template | EC2 instance configuration |
| Auto Scaling Group | Dynamic capacity management |
| Task Definition | Container blueprint |
| Application Load Balancer | Traffic distribution |
| ECS Service | Maintain desired task count |
| CloudWatch Logs | Application logging |

---

## 2. Prerequisites and Setup

### Required Tools

```bash
# Verify AWS CLI is installed and configured
aws --version
# Output: aws-cli/2.x.x or higher

# Verify credentials are configured
aws sts get-caller-identity
# Should return your account ID and user/role

# Install jq for JSON parsing (optional but helpful)
# macOS: brew install jq
# Ubuntu: sudo apt-get install jq
# Windows: download from https://stedolan.github.io/jq/
```

### Environment Variables

Set these variables for reuse throughout the tutorial:

```bash
# Region and naming
export AWS_REGION="us-east-1"
export CLUSTER_NAME="production-ecs-cluster"
export SERVICE_NAME="web-service"
export TASK_FAMILY="web-app"

# Networking
export VPC_CIDR="10.0.0.0/16"
export PUBLIC_SUBNET_1_CIDR="10.0.1.0/24"
export PUBLIC_SUBNET_2_CIDR="10.0.2.0/24"
export PRIVATE_SUBNET_1_CIDR="10.0.10.0/24"
export PRIVATE_SUBNET_2_CIDR="10.0.11.0/24"

# EC2 Configuration
export INSTANCE_TYPE="t3.medium"
export KEY_PAIR_NAME="ecs-keypair"  # Change to your key pair name
export MIN_INSTANCES=2
export MAX_INSTANCES=6
export DESIRED_INSTANCES=2
```

---

## 3. Step 1: Create VPC and Networking

### Create VPC

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block $VPC_CIDR \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=ecs-vpc}]" \
  --query 'Vpc.VpcId' \
  --output text \
  --region $AWS_REGION)

echo "VPC ID: $VPC_ID"

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames \
  --region $AWS_REGION
```

### Create Subnets

```bash
# Get availability zones
AZ_1=$(aws ec2 describe-availability-zones \
  --region $AWS_REGION \
  --query 'AvailabilityZones[0].ZoneName' \
  --output text)

AZ_2=$(aws ec2 describe-availability-zones \
  --region $AWS_REGION \
  --query 'AvailabilityZones[1].ZoneName' \
  --output text)

echo "AZ 1: $AZ_1"
echo "AZ 2: $AZ_2"

# Create Public Subnet 1
PUBLIC_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block $PUBLIC_SUBNET_1_CIDR \
  --availability-zone $AZ_1 \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $AWS_REGION)

# Create Public Subnet 2
PUBLIC_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block $PUBLIC_SUBNET_2_CIDR \
  --availability-zone $AZ_2 \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-2}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $AWS_REGION)

# Create Private Subnet 1
PRIVATE_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block $PRIVATE_SUBNET_1_CIDR \
  --availability-zone $AZ_1 \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $AWS_REGION)

# Create Private Subnet 2
PRIVATE_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block $PRIVATE_SUBNET_2_CIDR \
  --availability-zone $AZ_2 \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-2}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $AWS_REGION)

echo "Public Subnets: $PUBLIC_SUBNET_1, $PUBLIC_SUBNET_2"
echo "Private Subnets: $PRIVATE_SUBNET_1, $PRIVATE_SUBNET_2"
```

### Create Internet Gateway

```bash
# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=ecs-igw}]" \
  --query 'InternetGateway.InternetGatewayId' \
  --output text \
  --region $AWS_REGION)

# Attach to VPC
aws ec2 attach-internet-gateway \
  --vpc-id $VPC_ID \
  --internet-gateway-id $IGW_ID \
  --region $AWS_REGION

echo "Internet Gateway ID: $IGW_ID"
```

### Create NAT Gateway

```bash
# Allocate Elastic IP for NAT Gateway
NAT_EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text \
  --region $AWS_REGION)

# Create NAT Gateway in Public Subnet 1
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_1 \
  --allocation-id $NAT_EIP_ALLOC_ID \
  --tag-specifications "ResourceType=nat-gateway,Tags=[{Key=Name,Value=ecs-nat-gw}]" \
  --query 'NatGateway.NatGatewayId' \
  --output text \
  --region $AWS_REGION)

echo "NAT Gateway ID: $NAT_GW_ID"

# Wait for NAT Gateway to become available
echo "Waiting for NAT Gateway to be available..."
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids $NAT_GW_ID \
  --region $AWS_REGION
echo "NAT Gateway is now available"
```

### Create Route Tables

```bash
# Create Public Route Table
PUBLIC_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]" \
  --query 'RouteTable.RouteTableId' \
  --output text \
  --region $AWS_REGION)

# Add route to Internet Gateway
aws ec2 create-route \
  --route-table-id $PUBLIC_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID \
  --region $AWS_REGION

# Associate public subnets
aws ec2 associate-route-table \
  --subnet-id $PUBLIC_SUBNET_1 \
  --route-table-id $PUBLIC_RT_ID \
  --region $AWS_REGION

aws ec2 associate-route-table \
  --subnet-id $PUBLIC_SUBNET_2 \
  --route-table-id $PUBLIC_RT_ID \
  --region $AWS_REGION

# Create Private Route Table
PRIVATE_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]" \
  --query 'RouteTable.RouteTableId' \
  --output text \
  --region $AWS_REGION)

# Add route to NAT Gateway
aws ec2 create-route \
  --route-table-id $PRIVATE_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID \
  --region $AWS_REGION

# Associate private subnets
aws ec2 associate-route-table \
  --subnet-id $PRIVATE_SUBNET_1 \
  --route-table-id $PRIVATE_RT_ID \
  --region $AWS_REGION

aws ec2 associate-route-table \
  --subnet-id $PRIVATE_SUBNET_2 \
  --route-table-id $PRIVATE_RT_ID \
  --region $AWS_REGION

echo "Route Tables created and associated"
```

---

## 4. Step 2: Create IAM Roles

### ECS Instance Role (for EC2 instances)

This role allows EC2 instances to:
- Register with ECS cluster
- Pull images from ECR
- Write logs to CloudWatch

```bash
# Create trust policy for EC2
cat > ecs-instance-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name ecsInstanceRole \
  --assume-role-policy-document file://ecs-instance-trust-policy.json \
  --region $AWS_REGION

# Attach AWS managed policy
aws iam attach-role-policy \
  --role-name ecsInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role \
  --region $AWS_REGION

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name ecsInstanceProfile \
  --region $AWS_REGION

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ecsInstanceProfile \
  --role-name ecsInstanceRole \
  --region $AWS_REGION

echo "ECS Instance Role created"
```

### ECS Task Execution Role

This role allows ECS to:
- Pull container images from ECR
- Write logs to CloudWatch
- Retrieve secrets from Secrets Manager

```bash
# Create trust policy for ECS Tasks
cat > ecs-task-execution-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-task-execution-trust-policy.json \
  --region $AWS_REGION

# Attach AWS managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy \
  --region $AWS_REGION

echo "ECS Task Execution Role created"
```

### ECS Task Role (for application access to AWS services)

This role allows your application containers to access AWS services:

```bash
# Create trust policy
cat > ecs-task-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name ecsTaskRole \
  --assume-role-policy-document file://ecs-task-trust-policy.json \
  --region $AWS_REGION

# Create policy for S3 read access (example)
cat > ecs-task-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Attach custom policy
aws iam put-role-policy \
  --role-name ecsTaskRole \
  --policy-name S3ReadPolicy \
  --policy-document file://ecs-task-policy.json \
  --region $AWS_REGION

echo "ECS Task Role created"
```

---

## 5. Step 3: Create ECS Cluster

```bash
# Create ECS cluster
aws ecs create-cluster \
  --cluster-name $CLUSTER_NAME \
  --settings name=containerInsights,value=enabled \
  --region $AWS_REGION

# Verify cluster creation
aws ecs describe-clusters \
  --clusters $CLUSTER_NAME \
  --region $AWS_REGION

echo "ECS Cluster '$CLUSTER_NAME' created"
```

---

## 6. Step 4: Create Launch Template

### Create Security Group for ECS Instances

```bash
# Create security group
ECS_SG_ID=$(aws ec2 create-security-group \
  --group-name ecs-instance-sg \
  --description "Security group for ECS container instances" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text \
  --region $AWS_REGION)

# Allow ALB to communicate with containers (port 32768-65535 for dynamic port mapping)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 32768-65535 \
  --source-group $ECS_SG_ID \
  --region $AWS_REGION

# Allow SSH (optional, for debugging)
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION

echo "Security Group ID: $ECS_SG_ID"
```

### Get Latest ECS-Optimized AMI

```bash
# Get latest ECS-optimized AMI ID
ECS_AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ecs/optimized-ami/amazon-linux-2/recommended \
  --region $AWS_REGION \
  --query 'Parameters[0].Value' \
  --output text | jq -r '.image_id')

echo "ECS AMI ID: $ECS_AMI_ID"
```

### Create Launch Template

```bash
# Create user data script
cat > user-data.txt <<EOF
#!/bin/bash
echo ECS_CLUSTER=$CLUSTER_NAME >> /etc/ecs/ecs.config
echo ECS_ENABLE_TASK_IAM_ROLE=true >> /etc/ecs/ecs.config
echo ECS_ENABLE_TASK_IAM_ROLE_NETWORK_HOST=true >> /etc/ecs/ecs.config
echo ECS_LOGLEVEL=info >> /etc/ecs/ecs.config
echo ECS_AVAILABLE_LOGGING_DRIVERS='["json-file","awslogs"]' >> /etc/ecs/ecs.config
EOF

# Base64 encode user data
USER_DATA=$(base64 -w 0 user-data.txt)

# Get instance profile ARN
INSTANCE_PROFILE_ARN=$(aws iam get-instance-profile \
  --instance-profile-name ecsInstanceProfile \
  --query 'InstanceProfile.Arn' \
  --output text)

# Create launch template
cat > launch-template.json <<EOF
{
  "LaunchTemplateName": "ecs-launch-template",
  "VersionDescription": "Initial version",
  "LaunchTemplateData": {
    "ImageId": "$ECS_AMI_ID",
    "InstanceType": "$INSTANCE_TYPE",
    "KeyName": "$KEY_PAIR_NAME",
    "IamInstanceProfile": {
      "Arn": "$INSTANCE_PROFILE_ARN"
    },
    "SecurityGroupIds": ["$ECS_SG_ID"],
    "UserData": "$USER_DATA",
    "TagSpecifications": [
      {
        "ResourceType": "instance",
        "Tags": [
          {
            "Key": "Name",
            "Value": "ECS-Container-Instance"
          },
          {
            "Key": "Cluster",
            "Value": "$CLUSTER_NAME"
          }
        ]
      }
    ],
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpPutResponseHopLimit": 1
    }
  }
}
EOF

# Create the launch template
aws ec2 create-launch-template \
  --cli-input-json file://launch-template.json \
  --region $AWS_REGION

echo "Launch Template created"
```

---

## 7. Step 5: Create Auto Scaling Group

```bash
# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name ecs-asg \
  --launch-template LaunchTemplateName=ecs-launch-template \
  --min-size $MIN_INSTANCES \
  --max-size $MAX_INSTANCES \
  --desired-capacity $DESIRED_INSTANCES \
  --vpc-zone-identifier "$PRIVATE_SUBNET_1,$PRIVATE_SUBNET_2" \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --tags "Key=Name,Value=ECS-ASG-Instance,PropagateAtLaunch=true" \
  --region $AWS_REGION

echo "Auto Scaling Group 'ecs-asg' created"

# Wait a few minutes for instances to launch and register
echo "Waiting for instances to launch and register with cluster..."
sleep 120

# Verify container instances registered
aws ecs list-container-instances \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION
```

---

## 8. Step 6: Register Task Definition

### Create CloudWatch Log Group

```bash
# Create log group
aws logs create-log-group \
  --log-group-name /ecs/$TASK_FAMILY \
  --region $AWS_REGION

echo "CloudWatch Log Group created"
```

### Create Task Definition

```bash
# Get Task Execution Role ARN
TASK_EXECUTION_ROLE_ARN=$(aws iam get-role \
  --role-name ecsTaskExecutionRole \
  --query 'Role.Arn' \
  --output text)

# Get Task Role ARN
TASK_ROLE_ARN=$(aws iam get-role \
  --role-name ecsTaskRole \
  --query 'Role.Arn' \
  --output text)

# Create task definition JSON
cat > task-definition.json <<EOF
{
  "family": "$TASK_FAMILY",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "$TASK_EXECUTION_ROLE_ARN",
  "taskRoleArn": "$TASK_ROLE_ARN",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:latest",
      "cpu": 256,
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 0,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/$TASK_FAMILY",
          "awslogs-region": "$AWS_REGION",
          "awslogs-stream-prefix": "nginx"
        }
      }
    }
  ]
}
EOF

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region $AWS_REGION

echo "Task Definition '$TASK_FAMILY' registered"
```

**Key points about the task definition:**

- **networkMode: bridge** — Default for EC2, uses Docker bridge networking
- **hostPort: 0** — Dynamic port mapping (ECS assigns random port from 32768-65535)
- **requiresCompatibilities: ["EC2"]** — This task runs only on EC2
- **logConfiguration** — Sends logs to CloudWatch

---

## 9. Step 7: Create Application Load Balancer

### Create ALB Security Group

```bash
# Create ALB security group
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name alb-sg \
  --description "Security group for Application Load Balancer" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text \
  --region $AWS_REGION)

# Allow HTTP from internet
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION

# Allow HTTPS from internet (optional)
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --region $AWS_REGION

echo "ALB Security Group ID: $ALB_SG_ID"

# Update ECS security group to allow traffic from ALB
aws ec2 authorize-security-group-ingress \
  --group-id $ECS_SG_ID \
  --protocol tcp \
  --port 32768-65535 \
  --source-group $ALB_SG_ID \
  --region $AWS_REGION
```

### Create ALB

```bash
# Create Application Load Balancer
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name ecs-alb \
  --subnets $PUBLIC_SUBNET_1 $PUBLIC_SUBNET_2 \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text \
  --region $AWS_REGION)

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text \
  --region $AWS_REGION)

echo "ALB ARN: $ALB_ARN"
echo "ALB DNS: $ALB_DNS"
```

### Create Target Group

```bash
# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name ecs-target-group \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-enabled \
  --health-check-protocol HTTP \
  --health-check-path / \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text \
  --region $AWS_REGION)

echo "Target Group ARN: $TG_ARN"
```

### Create Listener

```bash
# Create listener for ALB
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --region $AWS_REGION

echo "ALB Listener created"
```

---

## 10. Step 8: Create ECS Service

```bash
# Create ECS service
aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name $SERVICE_NAME \
  --task-definition $TASK_FAMILY \
  --desired-count 3 \
  --launch-type EC2 \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=nginx,containerPort=80" \
  --health-check-grace-period-seconds 60 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=50" \
  --placement-strategy "type=spread,field=attribute:ecs.availability-zone" \
  --region $AWS_REGION

echo "ECS Service '$SERVICE_NAME' created with 3 tasks"

# Wait for service to stabilize
echo "Waiting for service to become stable (this may take 5-10 minutes)..."
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --region $AWS_REGION

echo "Service is now stable!"
```

**Service Configuration Explained:**

- **desired-count: 3** — ECS maintains 3 running tasks
- **maximumPercent: 200** — During deployment, can have up to 200% (6 tasks)
- **minimumHealthyPercent: 50** — Must keep at least 50% (2 tasks) healthy
- **placement-strategy: spread** — Distributes tasks evenly across AZs

---

## 11. Step 9: Test and Verify

### Verify Tasks are Running

```bash
# List running tasks
aws ecs list-tasks \
  --cluster $CLUSTER_NAME \
  --service-name $SERVICE_NAME \
  --desired-status RUNNING \
  --region $AWS_REGION

# Describe service
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --region $AWS_REGION
```

### Test the Application

```bash
# Access the application via ALB
echo "Application URL: http://$ALB_DNS"
echo ""
echo "Testing application..."
curl -I http://$ALB_DNS

# Expected output: HTTP/1.1 200 OK (from nginx)
```

Open in browser:
```
http://<ALB_DNS>
```

You should see the default nginx welcome page.

### Verify Targets are Healthy

```bash
# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --region $AWS_REGION
```

All targets should show `State: healthy`.

### View Logs

```bash
# View CloudWatch logs
aws logs tail /ecs/$TASK_FAMILY --follow --region $AWS_REGION
```

---

## 12. Step 10: Enable Auto Scaling

### Enable Service Auto Scaling

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/$CLUSTER_NAME/$SERVICE_NAME \
  --min-capacity 2 \
  --max-capacity 10 \
  --region $AWS_REGION

# Create scaling policy (target tracking - CPU)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/$CLUSTER_NAME/$SERVICE_NAME \
  --policy-name cpu-target-tracking-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }' \
  --region $AWS_REGION

echo "Service Auto Scaling enabled"
```

**What this does:**

- Maintains average CPU utilization at 70%
- Scales out when CPU > 70% for 3 minutes
- Scales in when CPU < 70% for 5 minutes
- Min: 2 tasks, Max: 10 tasks

### Enable Cluster Auto Scaling (Capacity Provider)

```bash
# Create capacity provider
aws ecs create-capacity-provider \
  --name ecs-capacity-provider \
  --auto-scaling-group-provider "autoScalingGroupArn=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ecs-asg --query 'AutoScalingGroups[0].AutoScalingGroupARN' --output text),managedScaling={status=ENABLED,targetCapacity=80,minimumScalingStepSize=1,maximumScalingStepSize=4},managedTerminationProtection=DISABLED" \
  --region $AWS_REGION

# Associate capacity provider with cluster
aws ecs put-cluster-capacity-providers \
  --cluster $CLUSTER_NAME \
  --capacity-providers ecs-capacity-provider \
  --default-capacity-provider-strategy capacityProvider=ecs-capacity-provider,weight=1,base=0 \
  --region $AWS_REGION

echo "Cluster Auto Scaling enabled via Capacity Provider"
```

**Capacity Provider benefits:**

- Automatically scales EC2 instances based on task demand
- Maintains 80% target capacity utilization
- Adds instances when tasks cannot be placed
- Removes instances when underutilized

---

## 13. Cleanup - Delete All Resources

**Important:** Run these commands in order to avoid dependency errors.

```bash
# Delete ECS Service
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --desired-count 0 \
  --region $AWS_REGION

aws ecs delete-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --force \
  --region $AWS_REGION

# Delete ECS Cluster
aws ecs delete-cluster \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION

# Delete Auto Scaling Group
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name ecs-asg \
  --force-delete \
  --region $AWS_REGION

# Delete Launch Template
aws ec2 delete-launch-template \
  --launch-template-name ecs-launch-template \
  --region $AWS_REGION

# Delete ALB
aws elbv2 delete-load-balancer \
  --load-balancer-arn $ALB_ARN \
  --region $AWS_REGION

# Wait for ALB to delete
sleep 60

# Delete Target Group
aws elbv2 delete-target-group \
  --target-group-arn $TG_ARN \
  --region $AWS_REGION

# Delete NAT Gateway
aws ec2 delete-nat-gateway \
  --nat-gateway-id $NAT_GW_ID \
  --region $AWS_REGION

# Wait for NAT Gateway deletion
sleep 60

# Release Elastic IP
aws ec2 release-address \
  --allocation-id $NAT_EIP_ALLOC_ID \
  --region $AWS_REGION

# Delete Internet Gateway
aws ec2 detach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID \
  --region $AWS_REGION

aws ec2 delete-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --region $AWS_REGION

# Delete Subnets
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_1 --region $AWS_REGION
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_2 --region $AWS_REGION
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_1 --region $AWS_REGION
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_2 --region $AWS_REGION

# Delete Security Groups
aws ec2 delete-security-group --group-id $ALB_SG_ID --region $AWS_REGION
aws ec2 delete-security-group --group-id $ECS_SG_ID --region $AWS_REGION

# Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID --region $AWS_REGION

# Delete CloudWatch Log Group
aws logs delete-log-group --log-group-name /ecs/$TASK_FAMILY --region $AWS_REGION

# Delete IAM Roles (optional - only if not used elsewhere)
aws iam remove-role-from-instance-profile --instance-profile-name ecsInstanceProfile --role-name ecsInstanceRole
aws iam delete-instance-profile --instance-profile-name ecsInstanceProfile
aws iam detach-role-policy --role-name ecsInstanceRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role
aws iam delete-role --role-name ecsInstanceRole

aws iam detach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ecsTaskExecutionRole

aws iam delete-role-policy --role-name ecsTaskRole --policy-name S3ReadPolicy
aws iam delete-role --role-name ecsTaskRole

echo "All resources deleted"
```

---

## 14. Troubleshooting Common Issues

### Issue 1: Instances Not Registering to Cluster

**Symptoms:**
```bash
aws ecs list-container-instances --cluster $CLUSTER_NAME
# Returns empty array
```

**Causes & Solutions:**

1. **User data script not executed properly**
   ```bash
   # SSH into instance and check
   cat /etc/ecs/ecs.config
   # Should contain: ECS_CLUSTER=production-ecs-cluster
   ```

2. **IAM role missing permissions**
   ```bash
   # Verify instance has correct IAM role
   aws ec2 describe-instances --instance-ids <instance-id> \
     --query 'Reservations[0].Instances[0].IamInstanceProfile'
   ```

3. **ECS agent not running**
   ```bash
   # SSH into instance
   sudo systemctl status ecs
   sudo systemctl restart ecs
   ```

### Issue 2: Tasks Stuck in PENDING

**Check reasons:**
```bash
aws ecs describe-tasks \
  --cluster $CLUSTER_NAME \
  --tasks <task-arn> \
  --query 'tasks[0].stoppedReason'
```

**Common causes:**

- Insufficient memory/CPU on instances
- Cannot pull image from ECR (IAM permissions)
- Port conflicts

### Issue 3: ALB Health Checks Failing

**Check target health:**
```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

**Common causes:**

- Security group not allowing ALB → ECS traffic
- Application not listening on correct port
- Health check path incorrect

### Issue 4: Service Won't Scale

**Verify auto scaling configuration:**
```bash
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs \
  --resource-ids service/$CLUSTER_NAME/$SERVICE_NAME
```

**Check CloudWatch metrics:**
- `ECSServiceAverageCPUUtilization`
- `ECSServiceAverageMemoryUtilization`

---

**End of Part 4 — Building ECS with EC2 Launch Type**
