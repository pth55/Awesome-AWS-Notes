# Part 4: CloudFormation — Tasks and Solutions

---

## Table of Contents

1. [Task 1 — VPC + EC2 + S3 Foundation Stack](#task-1--vpc--ec2--s3-foundation-stack)
2. [Task 2 — Lambda + SQS + S3 Event Pipeline](#task-2--lambda--sqs--s3-event-pipeline)
3. [Task 3 — Full Production Stack (VPC + ALB + EC2 ASG + RDS)](#task-3--full-production-stack-vpc--alb--ec2-asg--rds)
4. [Task 4 — Basic Infrastructure (VPC + EC2 + IAM + httpd)](#task-4--basic-infrastructure-vpc--ec2--iam--httpd)
5. [Task Solutions](#task-solutions)
6. [Key Points to Remember](#key-points-to-remember)
7. [Resources](#resources)

---

## Task 1 — VPC + EC2 + S3 Foundation Stack

### Objective

Build a foundation stack that provisions:
- A VPC with one public and one private subnet
- Internet Gateway and NAT Gateway
- An EC2 instance in the public subnet (web server)
- An S3 bucket for application logs with versioning enabled
- Proper security groups

This is the base infrastructure that almost every application needs.

### Architecture

```
                    ┌─────────────────────────────────────────────────┐
                    │              my-foundation-stack                 │
                    │                                                 │
                    │  ┌────────────────────────────────────────┐    │
                    │  │ VPC: 10.0.0.0/16                       │    │
                    │  │                                        │    │
                    │  │ ┌──────────────────┐  ┌────────────┐  │    │
                    │  │ │ Public Subnet    │  │ Private    │  │    │
                    │  │ │ 10.0.1.0/24      │  │ Subnet     │  │    │
                    │  │ │                  │  │ 10.0.2.0/24│  │    │
                    │  │ │  [EC2 Web]       │  │            │  │    │
                    │  │ │  [NAT GW]        │  │            │  │    │
                    │  │ └──────────────────┘  └────────────┘  │    │
                    │  │          │IGW                          │    │
                    │  └──────────┼─────────────────────────────┘    │
                    │             │                    [S3 Bucket]   │
                    └─────────────┼──────────────────────────────────┘
                                  │
                               Internet
```

### Requirements

Create `task1-foundation.yaml` with the following:

1. **Parameters:** `Environment` (dev/staging/production), `InstanceType` (t3.micro default), `KeyName`
2. **VPC:** CIDR `10.0.0.0/16`, DNS enabled
3. **Subnets:** Public (`10.0.1.0/24`), Private (`10.0.2.0/24`) — each in a different AZ using `!Select` + `!GetAZs`
4. **IGW + attachment + public route table** with a default route to IGW
5. **NAT Gateway + EIP + private route table** with default route to NAT GW
6. **Security group:** SSH (22) and HTTP (80) from anywhere
7. **EC2 instance:** in public subnet, uses `Mappings` for AMI ID per region, Nginx bootstrapped via `UserData`
8. **S3 bucket:** versioning enabled, server-side encryption (SSE-S3), public access blocked, bucket name uses `!Sub` with account ID for uniqueness
9. **Outputs:** VPC ID, public subnet ID, private subnet ID, EC2 public IP, S3 bucket name

---

## Task 2 — Lambda + SQS + S3 Event Pipeline

### Objective

Build a serverless event pipeline:
- An S3 bucket that triggers events when objects are uploaded
- An SQS queue that receives S3 event notifications
- A Lambda function that polls the SQS queue and processes messages
- Proper IAM roles and permissions

### Architecture

```
   User uploads file
          │
          ▼
   ┌──────────────┐
   │  S3 Bucket   │  ──event notification──►  ┌─────────────┐
   │  (uploads)   │                            │  SQS Queue  │
   └──────────────┘                            └──────┬──────┘
                                                      │ trigger (event source mapping)
                                                      ▼
                                               ┌────────────────┐
                                               │ Lambda Function│
                                               │  (processor)   │
                                               └────────────────┘
                                                      │
                                                      ▼
                                               ┌──────────────┐
                                               │  S3 Bucket   │
                                               │  (processed) │
                                               └──────────────┘
```

### Requirements

Create `task2-event-pipeline.yaml` with:

1. **Parameters:** `Environment`, `LambdaMemory` (128/256/512), `MessageRetentionSeconds`
2. **SQS Queue:**
   - Visibility timeout: 60 seconds
   - Message retention: use the `MessageRetentionSeconds` parameter
   - A Dead Letter Queue (DLQ) for failed messages (max receives: 3)
3. **S3 Upload Bucket:**
   - Event notification to SQS on `s3:ObjectCreated:*`
   - Versioning enabled
   - Bucket policy allowing S3 to send notifications to SQS
4. **S3 Processed Bucket:** destination for Lambda output
5. **IAM Role for Lambda:**
   - Trust policy: `lambda.amazonaws.com`
   - Permissions: SQS `ReceiveMessage`/`DeleteMessage`/`GetQueueAttributes`, S3 `GetObject` on upload bucket, S3 `PutObject` on processed bucket, CloudWatch Logs
6. **Lambda Function:**
   - Runtime: `python3.12`
   - Handler: `index.handler`
   - Memory: use `LambdaMemory` parameter
   - Timeout: 30 seconds
   - Inline code that parses the SQS event, logs the S3 key, and writes a confirmation file to the processed bucket
7. **SQS Event Source Mapping:** connects SQS to Lambda (batch size 5)
8. **Outputs:** SQS URL, DLQ URL, Lambda ARN, both bucket names

---

## Task 3 — Full Production Stack (VPC + ALB + EC2 ASG + RDS)

### Objective

Build a production-ready 3-tier web application:
- **Tier 1 (Public):** Application Load Balancer
- **Tier 2 (Private App):** Auto Scaling Group of EC2 instances
- **Tier 3 (Private Data):** RDS MySQL Multi-AZ

### Architecture

```
Internet
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                          │
│                                                           │
│  Public Subnets (AZ-a, AZ-b):                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │              Application Load Balancer              │   │
│  │                   (internet-facing)                 │   │
│  └───────────────────────┬────────────────────────────┘   │
│                          │ HTTP:80                         │
│  Private App Subnets (AZ-a, AZ-b):                        │
│  ┌────────────────────────────────────────────────────┐   │
│  │         Auto Scaling Group (min:2 max:4)            │   │
│  │     [EC2 App1 AZ-a]    [EC2 App2 AZ-b]             │   │
│  └───────────────────────┬────────────────────────────┘   │
│                          │ MySQL:3306                      │
│  Private DB Subnets (AZ-a, AZ-b):                         │
│  ┌────────────────────────────────────────────────────┐   │
│  │          RDS MySQL Multi-AZ                         │   │
│  │     [Primary AZ-a] ── [Standby AZ-b]               │   │
│  └────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

### Requirements

Create `task3-production.yaml` with:

1. **Parameters:** `Environment`, `EC2InstanceType`, `DBPassword`, `DBInstanceClass`, `MinInstances` (default 2), `MaxInstances` (default 4)
2. **VPC** with 6 subnets: 2 public, 2 private-app, 2 private-db (each pair in different AZs)
3. **IGW + NAT Gateway** (one per AZ for HA)
4. **Security Groups:**
   - ALB SG: inbound 80 from 0.0.0.0/0
   - App SG: inbound 80 only from ALB SG
   - DB SG: inbound 3306 only from App SG
5. **Launch Template:** AL2 AMI via Mappings, Nginx + app installed via UserData, instance profile with SSM access
6. **Auto Scaling Group:** 2 AZs, min/max from parameters, target tracking policy (CPU 60%)
7. **ALB + Target Group + Listener:** internet-facing, health check on `/`, port 80
8. **RDS MySQL:** Multi-AZ enabled in production (use `!If [IsProduction, ...]`), subnet group across 2 private-db subnets, DeletionPolicy Snapshot
9. **Conditions:** `IsProduction` — controls Multi-AZ for RDS and number of NAT Gateways
10. **Outputs:** ALB DNS name, RDS endpoint, VPC ID

---

## Task 4 — Basic Infrastructure (VPC + EC2 + IAM + httpd)

### Objective

This is a real-world lab task. Create a CloudFormation stack named `cmtr-58ir3aht-basic-infra` that deploys a VPC with two public subnets across two AZs, two EC2 instances each running an `httpd` server, and an IAM role with SSM access attached to the instances. Every resource must carry a `Maintainer` tag sourced from a template parameter.

### Architecture

```
                           Internet
                               │
                        ┌──────┴───────┐
                        │     IGW      │
                        │ cmtr-...-igw │
                        └──────┬───────┘
                               │
          ┌────────────────────┴─────────────────────┐
          │            VPC: cmtr-58ir3aht-vpc         │
          │             CIDR: 10.0.0.0/16             │
          │                                           │
          │  ┌──────────────────┐  ┌───────────────┐  │
          │  │ cmtr-...-subnet1 │  │ cmtr-...-sub2 │  │
          │  │   eu-west-1a     │  │  eu-west-1b   │  │
          │  │  10.0.1.0/24     │  │  10.0.2.0/24  │  │
          │  │                  │  │               │  │
          │  │  [instance1]     │  │  [instance2]  │  │
          │  │  httpd:          │  │  httpd:       │  │
          │  │  Hello eu-west-1a│  │  Hello eu-1b  │  │
          │  └──────────────────┘  └───────────────┘  │
          │  cmtr-...-public-rt1    cmtr-...-public-rt2│
          │  (routes → IGW)         (routes → IGW)     │
          └───────────────────────────────────────────┘
```

### Resources to create (exact names required)

| Resource Type | Name / ID | Notes |
|:-------------|:----------|:------|
| VPC | `cmtr-58ir3aht-vpc` | CIDR: `10.0.0.0/16` |
| Subnet 1 | `cmtr-58ir3aht-subnet1` | AZ: `eu-west-1a`, public |
| Subnet 2 | `cmtr-58ir3aht-subnet2` | AZ: `eu-west-1b`, public |
| Internet Gateway | `cmtr-58ir3aht-igw` | Attached to VPC |
| Route Table 1 | `cmtr-58ir3aht-public-rt1` | Routes subnet1 → IGW |
| Route Table 2 | `cmtr-58ir3aht-public-rt2` | Routes subnet2 → IGW |
| Security Group | `cmtr-58ir3aht-sg` | Ports 22 (SSH) and 80 (HTTP) open |
| EC2 Instance 1 | `cmtr-58ir3aht-instance1` | In subnet1, httpd → "Hello from Region eu-west-1a" |
| EC2 Instance 2 | `cmtr-58ir3aht-instance2` | In subnet2, httpd → "Hello from Region eu-west-1b" |
| IAM Role | `cmtr-58ir3aht-role` | `AmazonSSMManagedInstanceCore` policy |
| IAM Instance Profile | `cmtr-58ir3aht-role` | Profile wrapping the IAM role |

### Requirements

Create `cmtr-58ir3aht-basic-infra.yaml` with the following:

1. **Parameter:** `Maintainer` — default value `cmtr-58ir3aht-maintainer`. Every resource must include a `Maintainer` tag using `!Ref Maintainer`.
2. **VPC** with `EnableDnsHostnames: true` and `EnableDnsSupport: true`
3. **Two public subnets** with hardcoded AZs (`eu-west-1a`, `eu-west-1b`) and `MapPublicIpOnLaunch: true`
4. **Two separate route tables** (one per subnet) — each with its own default route to the IGW and subnet association
5. **Security group** allowing port 22 and port 80 from `0.0.0.0/0`
6. **IAM Role** with EC2 trust policy and `AmazonSSMManagedInstanceCore` managed policy
7. **IAM Instance Profile** wrapping the role (same name as the role)
8. **Two EC2 instances** — each uses `UserData` to install `httpd`, start and enable it, and write the appropriate region message to `/var/www/html/index.html`

### Deployment Steps

```
Step 1: Create a CloudFormation deployment IAM role
──────────────────────────────────────────────────────────────────────────
IAM Console → Roles → Create Role
  Trusted entity: AWS Service → CloudFormation
  Permissions: AdministratorAccess
  Role name: cfn-deployment-role   (or any name you prefer)

Step 2: Write the template locally
  Save as:  cmtr-58ir3aht-basic-infra.yaml

Step 3: Deploy from Console
  CloudFormation → Create Stack → Upload file
  Stack name:       cmtr-58ir3aht-basic-infra
  Parameters:       Maintainer = cmtr-58ir3aht-maintainer
  Permissions:      IAM Role → select cfn-deployment-role
  Capabilities:     ✓ I acknowledge that AWS CloudFormation might create IAM resources

Step 4: Deploy from CLI (alternative)
  aws cloudformation deploy \
    --stack-name cmtr-58ir3aht-basic-infra \
    --template-file cmtr-58ir3aht-basic-infra.yaml \
    --parameter-overrides Maintainer=cmtr-58ir3aht-maintainer \
    --capabilities CAPABILITY_NAMED_IAM \
    --role-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/cfn-deployment-role \
    --region eu-west-1
```

### Verification Checklist

```
✓ Stack status: CREATE_COMPLETE (no events showing FAILED)
✓ VPC "cmtr-58ir3aht-vpc" exists in eu-west-1
✓ Both subnets have MapPublicIpOnLaunch enabled
✓ Both route tables have a 0.0.0.0/0 → IGW route
✓ Each subnet is associated with its own route table
✓ Security group allows port 22 and 80
✓ Both EC2 instances are running and have public IPs
✓ IAM role "cmtr-58ir3aht-role" has AmazonSSMManagedInstanceCore
✓ All resources have the Maintainer tag

Connect via Session Manager and verify:
  Instance 1:  curl http://localhost   → "Hello from Region eu-west-1a"
  Instance 2:  curl http://localhost   → "Hello from Region eu-west-1b"
```

---

## Task Solutions

> Click the arrow on any solution to expand it. Try the task yourself first.

### Task 1 Solution

<details>
<summary>▶ Show Solution — task1-foundation.yaml</summary>

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Foundation stack - VPC, EC2, S3

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, production]
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName

Mappings:
  RegionAMI:
    us-east-1:
      AL2: ami-0c02fb55956c7d316
    us-west-2:
      AL2: ami-0ca285d4c2cda3300
    eu-west-1:
      AL2: ami-0d71ea30463e0ff49

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-vpc"

  # Internet Gateway
  IGW:
    Type: AWS::EC2::InternetGateway
  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  # Subnets
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.1.0/24"
      AvailabilityZone: !Select [0, !GetAZs !Ref "AWS::Region"]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public"

  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.2.0/24"
      AvailabilityZone: !Select [1, !GetAZs !Ref "AWS::Region"]
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-private"

  # Public Route Table
  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: IGWAttachment
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref IGW
  PublicSubnetRTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRT

  # NAT Gateway
  NatEIP:
    Type: AWS::EC2::EIP
    DependsOn: IGWAttachment
    Properties:
      Domain: vpc
  NatGW:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEIP.AllocationId
      SubnetId: !Ref PublicSubnet

  # Private Route Table
  PrivateRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRT
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGW
  PrivateSubnetRTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet
      RouteTableId: !Ref PrivateRT

  # Security Group
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server SG
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: "0.0.0.0/0"
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: "0.0.0.0/0"

  # EC2 Instance
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionAMI, !Ref "AWS::Region", AL2]
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebSG
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          amazon-linux-extras install -y nginx1
          systemctl start nginx
          systemctl enable nginx
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web"
        - Key: Environment
          Value: !Ref Environment

  # S3 Bucket
  LogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::StackName}-logs-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

Outputs:
  VpcId:
    Value: !Ref VPC
  PublicSubnetId:
    Value: !Ref PublicSubnet
  PrivateSubnetId:
    Value: !Ref PrivateSubnet
  EC2PublicIP:
    Value: !GetAtt WebServer.PublicIp
  S3BucketName:
    Value: !Ref LogsBucket
```

</details>

### Task 2 Solution

<details>
<summary>▶ Show Solution — task2-event-pipeline.yaml</summary>

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lambda + SQS + S3 event pipeline

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, production]
  LambdaMemory:
    Type: Number
    Default: 128
    AllowedValues: [128, 256, 512]
  MessageRetentionSeconds:
    Type: Number
    Default: 86400

Resources:
  # Dead Letter Queue
  ProcessingDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "${AWS::StackName}-dlq"
      MessageRetentionPeriod: 1209600  # 14 days

  # Main Processing Queue
  ProcessingQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "${AWS::StackName}-queue"
      VisibilityTimeout: 60
      MessageRetentionPeriod: !Ref MessageRetentionSeconds
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt ProcessingDLQ.Arn
        maxReceiveCount: 3

  # Allow S3 to send messages to SQS
  QueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref ProcessingQueue
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: s3.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt ProcessingQueue.Arn
            Condition:
              ArnLike:
                aws:SourceArn: !Sub "arn:aws:s3:::${AWS::StackName}-uploads-${AWS::AccountId}"

  # S3 Upload Bucket
  UploadBucket:
    Type: AWS::S3::Bucket
    DependsOn: QueuePolicy
    Properties:
      BucketName: !Sub "${AWS::StackName}-uploads-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled
      NotificationConfiguration:
        QueueConfigurations:
          - Event: "s3:ObjectCreated:*"
            Queue: !GetAtt ProcessingQueue.Arn

  # S3 Processed Bucket
  ProcessedBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::StackName}-processed-${AWS::AccountId}"

  # IAM Role for Lambda
  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "${AWS::StackName}-lambda-role"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: SQSAndS3Access
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - sqs:ReceiveMessage
                  - sqs:DeleteMessage
                  - sqs:GetQueueAttributes
                Resource: !GetAtt ProcessingQueue.Arn
              - Effect: Allow
                Action: s3:GetObject
                Resource: !Sub "arn:aws:s3:::${UploadBucket}/*"
              - Effect: Allow
                Action: s3:PutObject
                Resource: !Sub "arn:aws:s3:::${ProcessedBucket}/*"

  # Lambda Function
  ProcessorFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub "${AWS::StackName}-processor"
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt LambdaRole.Arn
      MemorySize: !Ref LambdaMemory
      Timeout: 30
      Environment:
        Variables:
          PROCESSED_BUCKET: !Ref ProcessedBucket
      Code:
        ZipFile: |
          import json
          import boto3
          import os

          s3 = boto3.client('s3')
          processed_bucket = os.environ['PROCESSED_BUCKET']

          def handler(event, context):
              for record in event['Records']:
                  body = json.loads(record['body'])
                  for s3_record in body.get('Records', []):
                      key = s3_record['s3']['object']['key']
                      source_bucket = s3_record['s3']['bucket']['name']
                      print(f"Processing: s3://{source_bucket}/{key}")
                      s3.put_object(
                          Bucket=processed_bucket,
                          Key=f"done/{key}",
                          Body=f"Processed: {key}"
                      )
              return {'statusCode': 200}

  # Event Source Mapping: SQS → Lambda
  SQSTrigger:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt ProcessingQueue.Arn
      FunctionName: !GetAtt ProcessorFunction.Arn
      BatchSize: 5
      Enabled: true

Outputs:
  QueueUrl:
    Value: !Ref ProcessingQueue
  DLQUrl:
    Value: !Ref ProcessingDLQ
  LambdaArn:
    Value: !GetAtt ProcessorFunction.Arn
  UploadBucketName:
    Value: !Ref UploadBucket
  ProcessedBucketName:
    Value: !Ref ProcessedBucket
```

</details>

### Task 3 Solution

<details>
<summary>▶ Show Solution — task3-production.yaml</summary>

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Production 3-tier stack - VPC + ALB + ASG + RDS MySQL

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, production]
  EC2InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium, m5.large]
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
  DBInstanceClass:
    Type: String
    Default: db.t3.micro
    AllowedValues: [db.t3.micro, db.t3.small, db.m5.large]
  MinInstances:
    Type: Number
    Default: 2
  MaxInstances:
    Type: Number
    Default: 4

Conditions:
  IsProduction: !Equals [!Ref Environment, production]

Mappings:
  RegionAMI:
    us-east-1:
      AL2: ami-0c02fb55956c7d316
    us-west-2:
      AL2: ami-0ca285d4c2cda3300
    eu-west-1:
      AL2: ami-0d71ea30463e0ff49

Resources:
  # ─── VPC ─────────────────────────────────────────────────────────────
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-vpc"

  IGW:
    Type: AWS::EC2::InternetGateway
  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  # ─── Public Subnets ───────────────────────────────────────────────────
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.1.0/24"
      AvailabilityZone: !Select [0, !GetAZs !Ref "AWS::Region"]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-1"

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.2.0/24"
      AvailabilityZone: !Select [1, !GetAZs !Ref "AWS::Region"]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-2"

  # ─── Private App Subnets ─────────────────────────────────────────────
  AppSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.3.0/24"
      AvailabilityZone: !Select [0, !GetAZs !Ref "AWS::Region"]
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-app-1"

  AppSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.4.0/24"
      AvailabilityZone: !Select [1, !GetAZs !Ref "AWS::Region"]
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-app-2"

  # ─── Private DB Subnets ──────────────────────────────────────────────
  DBSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.5.0/24"
      AvailabilityZone: !Select [0, !GetAZs !Ref "AWS::Region"]
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-db-1"

  DBSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.6.0/24"
      AvailabilityZone: !Select [1, !GetAZs !Ref "AWS::Region"]
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-db-2"

  # ─── NAT Gateways (one per AZ for HA) ────────────────────────────────
  NatEIP1:
    Type: AWS::EC2::EIP
    DependsOn: IGWAttachment
    Properties:
      Domain: vpc

  NatGW1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEIP1.AllocationId
      SubnetId: !Ref PublicSubnet1

  NatEIP2:
    Type: AWS::EC2::EIP
    DependsOn: IGWAttachment
    Properties:
      Domain: vpc

  NatGW2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEIP2.AllocationId
      SubnetId: !Ref PublicSubnet2

  # ─── Public Route Table ──────────────────────────────────────────────
  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: IGWAttachment
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref IGW

  PublicSubnet1RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRT

  PublicSubnet2RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRT

  # ─── Private App Route Tables (one per NAT GW for HA) ─────────────────
  AppRT1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  AppRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref AppRT1
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGW1

  AppSubnet1RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref AppSubnet1
      RouteTableId: !Ref AppRT1

  AppRT2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  AppRoute2:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref AppRT2
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGW2

  AppSubnet2RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref AppSubnet2
      RouteTableId: !Ref AppRT2

  # ─── Security Groups ─────────────────────────────────────────────────
  ALBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB security group — HTTP from internet
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: "0.0.0.0/0"
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-alb-sg"

  AppSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App server — HTTP only from ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSG
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-app-sg"

  DBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: RDS — MySQL only from App tier
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref AppSG
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-db-sg"

  # ─── IAM for EC2 (SSM access) ─────────────────────────────────────────
  AppInstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

  AppInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref AppInstanceRole

  # ─── Launch Template ─────────────────────────────────────────────────
  AppLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub "${AWS::StackName}-lt"
      LaunchTemplateData:
        ImageId: !FindInMap [RegionAMI, !Ref "AWS::Region", AL2]
        InstanceType: !Ref EC2InstanceType
        IamInstanceProfile:
          Arn: !GetAtt AppInstanceProfile.Arn
        SecurityGroupIds:
          - !Ref AppSG
        UserData:
          Fn::Base64: |
            #!/bin/bash
            yum update -y
            amazon-linux-extras install -y nginx1
            systemctl start nginx
            systemctl enable nginx
            echo "<h1>OK</h1>" > /usr/share/nginx/html/index.html

  # ─── Auto Scaling Group ──────────────────────────────────────────────
  AppASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: !Sub "${AWS::StackName}-asg"
      MinSize: !Ref MinInstances
      MaxSize: !Ref MaxInstances
      DesiredCapacity: !Ref MinInstances
      VPCZoneIdentifier:
        - !Ref AppSubnet1
        - !Ref AppSubnet2
      LaunchTemplate:
        LaunchTemplateId: !Ref AppLaunchTemplate
        Version: !GetAtt AppLaunchTemplate.LatestVersionNumber
      TargetGroupARNs:
        - !Ref AppTG
      HealthCheckType: ELB
      HealthCheckGracePeriod: 60
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-app"
          PropagateAtLaunch: true

  CPUScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AppASG
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 60.0

  # ─── ALB ─────────────────────────────────────────────────────────────
  AppALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub "${AWS::StackName}-alb"
      Scheme: internet-facing
      Type: application
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSG

  AppTG:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub "${AWS::StackName}-tg"
      VpcId: !Ref VPC
      Protocol: HTTP
      Port: 80
      TargetType: instance
      HealthCheckPath: /
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref AppALB
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref AppTG

  # ─── RDS MySQL ───────────────────────────────────────────────────────
  RDSSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: RDS subnet group
      SubnetIds:
        - !Ref DBSubnet1
        - !Ref DBSubnet2

  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      DBInstanceIdentifier: !Sub "${AWS::StackName}-mysql"
      DBName: appdb
      DBInstanceClass: !Ref DBInstanceClass
      Engine: MySQL
      EngineVersion: "8.0"
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      AllocatedStorage: "20"
      StorageType: gp3
      MultiAZ: !If [IsProduction, true, false]
      DBSubnetGroupName: !Ref RDSSubnetGroup
      VPCSecurityGroups:
        - !Ref DBSG
      BackupRetentionPeriod: 7
      StorageEncrypted: true

Outputs:
  ALBDNSName:
    Description: Application Load Balancer DNS — paste this in your browser
    Value: !GetAtt AppALB.DNSName
  DBEndpoint:
    Description: RDS MySQL endpoint
    Value: !GetAtt DBInstance.Endpoint.Address
  VpcId:
    Value: !Ref VPC
```

</details>

### Task 4 Solution

<details>
<summary>▶ Show Solution — cmtr-58ir3aht-basic-infra.yaml</summary>

**Deployment note:** Before running the template, create a CloudFormation IAM role with `AdministratorAccess`, then pass it via `--role-arn` (CLI) or the **Permissions** step in the Console. Use `--capabilities CAPABILITY_NAMED_IAM` because the template creates a named IAM role.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Basic infrastructure - VPC, public subnets, EC2 with httpd, IAM role with SSM

Parameters:
  Maintainer:
    Type: String
    Default: cmtr-58ir3aht-maintainer
    Description: Applied as the Maintainer tag on every resource

Resources:
  # ─── VPC ─────────────────────────────────────────────────────────────
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-vpc
        - Key: Maintainer
          Value: !Ref Maintainer

  # ─── Internet Gateway ─────────────────────────────────────────────────
  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-igw
        - Key: Maintainer
          Value: !Ref Maintainer

  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  # ─── Public Subnets ───────────────────────────────────────────────────
  Subnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.1.0/24"
      AvailabilityZone: eu-west-1a
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-subnet1
        - Key: Maintainer
          Value: !Ref Maintainer

  Subnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: "10.0.2.0/24"
      AvailabilityZone: eu-west-1b
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-subnet2
        - Key: Maintainer
          Value: !Ref Maintainer

  # ─── Route Table 1 → Subnet1 ──────────────────────────────────────────
  PublicRT1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-public-rt1
        - Key: Maintainer
          Value: !Ref Maintainer

  PublicRoute1:
    Type: AWS::EC2::Route
    DependsOn: IGWAttachment
    Properties:
      RouteTableId: !Ref PublicRT1
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref IGW

  Subnet1RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Subnet1
      RouteTableId: !Ref PublicRT1

  # ─── Route Table 2 → Subnet2 ──────────────────────────────────────────
  PublicRT2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-public-rt2
        - Key: Maintainer
          Value: !Ref Maintainer

  PublicRoute2:
    Type: AWS::EC2::Route
    DependsOn: IGWAttachment
    Properties:
      RouteTableId: !Ref PublicRT2
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref IGW

  Subnet2RTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Subnet2
      RouteTableId: !Ref PublicRT2

  # ─── Security Group ───────────────────────────────────────────────────
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: cmtr-58ir3aht-sg
      GroupDescription: Allow SSH (22) and HTTP (80) from anywhere
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: "0.0.0.0/0"
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: "0.0.0.0/0"
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-sg
        - Key: Maintainer
          Value: !Ref Maintainer

  # ─── IAM Role with SSM ────────────────────────────────────────────────
  SSMRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: cmtr-58ir3aht-role
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      Tags:
        - Key: Maintainer
          Value: !Ref Maintainer

  SSMInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: cmtr-58ir3aht-role
      Roles:
        - !Ref SSMRole

  # ─── EC2 Instances ────────────────────────────────────────────────────
  Instance1:
    Type: AWS::EC2::Instance
    Properties:
      # Amazon Linux 2 AMI for eu-west-1 — verify latest ID in console if needed
      ImageId: ami-0d71ea30463e0ff49
      InstanceType: t3.micro
      SubnetId: !Ref Subnet1
      SecurityGroupIds:
        - !Ref WebSG
      IamInstanceProfile: !Ref SSMInstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "Hello from Region eu-west-1a" > /var/www/html/index.html
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-instance1
        - Key: Maintainer
          Value: !Ref Maintainer

  Instance2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d71ea30463e0ff49
      InstanceType: t3.micro
      SubnetId: !Ref Subnet2
      SecurityGroupIds:
        - !Ref WebSG
      IamInstanceProfile: !Ref SSMInstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "Hello from Region eu-west-1b" > /var/www/html/index.html
      Tags:
        - Key: Name
          Value: cmtr-58ir3aht-instance2
        - Key: Maintainer
          Value: !Ref Maintainer

Outputs:
  Instance1PublicIP:
    Description: Public IP of Instance 1 (eu-west-1a)
    Value: !GetAtt Instance1.PublicIp
  Instance2PublicIP:
    Description: Public IP of Instance 2 (eu-west-1b)
    Value: !GetAtt Instance2.PublicIp
  VpcId:
    Value: !Ref VPC
```

</details>

---

## Key Points to Remember

```
CloudFormation fundamentals:
  ✓ Resources is the only mandatory section
  ✓ CloudFormation figures out dependency order from !Ref and !GetAtt
  ✓ Delete a stack → deletes all resources it owns
  ✓ DeletionPolicy: Snapshot protects RDS data on stack delete
  ✓ Always use Change Sets in production before applying updates

Intrinsic functions:
  ✓ !Ref parameter → returns parameter value
  ✓ !Ref resource → returns resource ID (VPC ID, instance ID, etc.)
  ✓ !GetAtt → returns a specific attribute (DNS, ARN, IP, etc.)
  ✓ !Sub → string substitution, best for building names and ARNs
  ✓ !FindInMap → lookup from a Mappings table
  ✓ !Select + !GetAZs → pick an AZ automatically

Common gotchas:
  ✓ NAT Gateway needs an EIP, and both need a public subnet
  ✓ RDS needs a DBSubnetGroup with subnets in at least 2 AZs
  ✓ ALB needs subnets in at least 2 AZs
  ✓ Changing a VPC CIDR requires resource replacement (downtime)
  ✓ S3 bucket names must be globally unique — use !Sub with AccountId
  ✓ IAM resources require --capabilities CAPABILITY_IAM in CLI
  ✓ VPCGatewayAttachment must exist before creating routes to IGW

Tools:
  ✓ cfn-lint: catches schema errors before you deploy
  ✓ AWS Toolkit in VS Code: inline docs and auto-complete
  ✓ Infrastructure Composer: visualize any template visually
  ✓ Change Sets: see exactly what will change before it changes
  ✓ Drift Detection: find resources changed outside CloudFormation
```

---

## Resources

- [CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/)
- [CloudFormation Resource Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html) — all resource types and their properties
- [Intrinsic Functions Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)
- [Infrastructure Composer](https://console.aws.amazon.com/composer/home)
- [cfn-lint GitHub](https://github.com/aws-cloudformation/cfn-lint)
- [AWS Toolkit for VS Code](https://marketplace.visualstudio.com/items?itemName=AmazonWebServices.aws-toolkit-vscode)
- [AWS CloudFormation Templates (GitHub — official samples)](https://github.com/awslabs/aws-cloudformation-templates)
- [Telugu YouTube Series — CloudFormation Playlist](https://www.youtube.com/watch?v=XlOxrLt3QL8&list=PLneBjIzDLECkYQ7dYpvWFSgEwTMeZkykF&index=54)
