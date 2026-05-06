# Part 2: AWS CloudFormation — Complete Guide

---

## Table of Contents

1. [What is CloudFormation?](#1-what-is-cloudformation)
2. [Core Concepts — Stacks, Templates, Change Sets](#2-core-concepts--stacks-templates-change-sets)
3. [Template Anatomy — All 9 Sections](#3-template-anatomy--all-9-sections)
4. [Parameters — Making Templates Reusable](#4-parameters--making-templates-reusable)
5. [Mappings — Region and Environment Lookups](#5-mappings--region-and-environment-lookups)
6. [Conditions — Deploy Logic in Templates](#6-conditions--deploy-logic-in-templates)
7. [Resources — The Core Section](#7-resources--the-core-section)
8. [Outputs — Sharing Values Between Stacks](#8-outputs--sharing-values-between-stacks)
9. [Intrinsic Functions — The Power Tools](#9-intrinsic-functions--the-power-tools)
10. [Pseudo Parameters](#10-pseudo-parameters)
11. [Creating and Uploading Templates](#11-creating-and-uploading-templates)
12. [VS Code AWS Toolkit — Visualize While You Write](#12-vs-code-aws-toolkit--visualize-while-you-write)
13. [Infrastructure Composer — Visual Template Builder](#13-infrastructure-composer--visual-template-builder)
14. [Deploying and Updating Stacks](#14-deploying-and-updating-stacks)
15. [Change Sets — Preview Before You Apply](#15-change-sets--preview-before-you-apply)
16. [Stack Drift Detection](#16-stack-drift-detection)
17. [Nested Stacks](#17-nested-stacks)
18. [Complete Examples → See Part 3](#complete-examples--see-part-3)
19. [Tasks and Solutions → See Part 4](#tasks-and-solutions--see-part-4)

---

## 1. What is CloudFormation?

AWS CloudFormation is **Infrastructure as Code (IaC)** — you write a template file that describes the infrastructure you want, and CloudFormation creates, updates, and deletes those resources automatically. Instead of clicking through the AWS Console to create a VPC, subnets, EC2 instances, and databases one by one, you write a YAML file and CloudFormation does all of it in the right order.

```
Without CloudFormation:                With CloudFormation:
───────────────────────────────        ──────────────────────────────────────
You → Console → Create VPC             Write template.yaml
You → Console → Create Subnet          cfn deploy --template template.yaml
You → Console → Create IGW             CloudFormation:
You → Console → Create EC2               ✓ Creates VPC
You → Console → Create RDS               ✓ Creates Subnet (after VPC)
You → Console → Create SG                ✓ Creates IGW
...30+ manual steps, easy              ✓ Creates EC2 (after subnet)
to mess up, hard to repeat               ✓ Creates RDS (after SG, subnet)
                                         ✓ Handles dependencies automatically
```

### Why use CloudFormation?

| Problem without IaC | How CloudFormation solves it |
|:--------------------|:---------------------------|
| Manual, error-prone setup | Automated, repeatable deployments |
| Can't recreate exact environment | Template is the source of truth |
| Drift between dev/staging/prod | Same template → identical environments |
| No audit trail | Git history of template = change log |
| Hard to clean up | Delete one stack → deletes everything |
| Multi-region deployment is painful | Deploy same template to any region |

### CloudFormation is free

You pay only for the AWS resources that CloudFormation creates. CloudFormation itself has no additional charge.

---

## 2. Core Concepts — Stacks, Templates, Change Sets

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CloudFormation Concepts                          │
│                                                                         │
│   Template (.yaml / .json)                                              │
│   ┌──────────────────────────┐                                          │
│   │  Description of          │  ←── You write this                      │
│   │  what should exist       │                                          │
│   │  (VPC, EC2, RDS, etc.)   │                                          │
│   └────────────┬─────────────┘                                          │
│                │  CloudFormation reads template                          │
│                ▼                                                         │
│   Stack (logical grouping)                                              │
│   ┌──────────────────────────────────────────────────────────┐          │
│   │  Stack: my-app-stack                                     │          │
│   │                                                           │          │
│   │  Resources created and owned by this stack:              │          │
│   │    VPC (vpc-abc123)                                      │          │
│   │    Subnet (subnet-def456)                                │          │
│   │    EC2 (i-ghi789)                                        │          │
│   │    RDS (mydb.xxxxx.us-east-1.rds.amazonaws.com)          │          │
│   │                                                           │          │
│   │  Delete this stack → all these resources are deleted      │          │
│   └──────────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key terms

| Term | Meaning |
|:-----|:--------|
| **Template** | The YAML/JSON file describing what infrastructure to create |
| **Stack** | A live instance of a deployed template — the actual running resources |
| **Stack Name** | Unique name you give to a stack in a region (`my-app-prod`) |
| **Change Set** | A preview of what will change before you apply an update |
| **Drift** | When someone manually changes a resource outside CloudFormation |
| **Logical ID** | The name you give a resource inside the template (`WebServer`) |
| **Physical ID** | The actual AWS resource ID created (`i-0abc123def456`) |
| **Rollback** | If a stack create/update fails, CloudFormation undoes all changes |

### Stack lifecycle

```
Template ready
      │
      ▼
  CREATE_IN_PROGRESS
      │
      ├── Success → CREATE_COMPLETE
      │
      └── Failure → ROLLBACK_IN_PROGRESS → ROLLBACK_COMPLETE

Update template
      │
      ▼
  UPDATE_IN_PROGRESS
      │
      ├── Success → UPDATE_COMPLETE
      │
      └── Failure → UPDATE_ROLLBACK_IN_PROGRESS → UPDATE_ROLLBACK_COMPLETE

Delete stack
      │
      ▼
  DELETE_IN_PROGRESS → DELETE_COMPLETE
```

---

## 3. Template Anatomy — All 9 Sections

A CloudFormation template can have up to 9 top-level sections. Only `Resources` is mandatory.

```yaml
---
AWSTemplateFormatVersion: "2010-09-09"   # Section 1: always this value
Description: What this template does     # Section 2: human-readable description
Metadata: {}                             # Section 3: extra info for tools/UI
Parameters: {}                           # Section 4: user inputs at deploy time
Rules: {}                                # Section 5: validation rules for params
Mappings: {}                             # Section 6: lookup tables
Conditions: {}                           # Section 7: conditional logic
Transform: []                            # Section 8: macros (SAM uses this)
Resources: {}                            # Section 9: MANDATORY - actual resources
Outputs: {}                              # Section 10: values to export
```

### Section order matters

CloudFormation processes sections in a specific order. You must define a parameter before you can reference it. Resources can reference each other — CloudFormation figures out the dependency graph automatically.

```
AWSTemplateFormatVersion → Description → Metadata
       → Parameters → Mappings → Conditions
              → Resources → Outputs
```

### Minimal valid template

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Creates an S3 bucket
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
```

That's it. Three lines (plus the version). One resource. This is a valid, deployable template.

---

## 4. Parameters — Making Templates Reusable

Parameters let users provide input values at deploy time instead of hardcoding them. A template with parameters can create a `t3.micro` dev environment or a `m5.4xlarge` production environment from the same file.

### Basic parameter definition

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    Description: EC2 instance type for the web server
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
      - m5.large
    ConstraintDescription: Must be a valid EC2 instance type from the allowed list.

  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production
    Description: Deployment environment

  DBPassword:
    Type: String
    NoEcho: true          # hides the value in console and CLI output
    MinLength: 8
    MaxLength: 41
    Description: RDS database password (8-41 characters)
    ConstraintDescription: Password must be 8-41 characters.

  VpcCIDR:
    Type: String
    Default: "10.0.0.0/16"
    Description: CIDR block for the VPC
```

### Parameter types

| Type | Example | Notes |
|:-----|:--------|:------|
| `String` | `t3.micro`, `my-bucket` | Most common — any text value |
| `Number` | `8080`, `3` | Numeric, supports Min/MaxValue |
| `List<Number>` | `80,443` | Comma-separated numbers |
| `CommaDelimitedList` | `us-east-1a,us-east-1b` | Comma-separated strings |
| `AWS::EC2::KeyPair::KeyName` | `my-keypair` | Validated against actual key pairs |
| `AWS::EC2::Subnet::Id` | `subnet-abc123` | Validated against actual subnets |
| `AWS::EC2::VPC::Id` | `vpc-abc123` | Validated against actual VPCs |
| `AWS::SSM::Parameter::Value<String>` | `/my/param` | Reads from SSM Parameter Store |

### Referencing parameters in resources

Use `!Ref ParameterName` to use a parameter's value:

```yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType    # uses the InstanceType parameter
      ImageId: ami-0c02fb55956c7d316
      Tags:
        - Key: Environment
          Value: !Ref Environment        # uses the Environment parameter
```

---

## 5. Mappings — Region and Environment Lookups

Mappings are static lookup tables in your template. The classic use case is finding the right AMI ID for each region, or setting different sizes for dev vs production.

```yaml
Mappings:
  RegionAMI:
    us-east-1:
      AmazonLinux2: ami-0c02fb55956c7d316
      Ubuntu2204:   ami-0aa2b7722dc1b5612
    us-west-2:
      AmazonLinux2: ami-0ca285d4c2cda3300
      Ubuntu2204:   ami-017fecd1353bcc96e
    eu-west-1:
      AmazonLinux2: ami-0d71ea30463e0ff49
      Ubuntu2204:   ami-0694d931cee176e7d

  EnvironmentSize:
    dev:
      InstanceType: t3.micro
      DBClass: db.t3.micro
      MultiAZ: false
    staging:
      InstanceType: t3.small
      DBClass: db.t3.small
      MultiAZ: false
    production:
      InstanceType: m5.large
      DBClass: db.m5.large
      MultiAZ: true
```

### Looking up a mapping value

Use `!FindInMap [MapName, TopLevelKey, SecondLevelKey]`:

```yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      # Automatically pick the right AMI for whatever region we deploy to
      ImageId: !FindInMap [RegionAMI, !Ref "AWS::Region", AmazonLinux2]

      # Pick instance size based on the Environment parameter
      InstanceType: !FindInMap [EnvironmentSize, !Ref Environment, InstanceType]
```

`!Ref "AWS::Region"` is a pseudo-parameter — covered in Section 10.

---

## 6. Conditions — Deploy Logic in Templates

Conditions let you create or configure resources only when certain criteria are met. The most common use case: create a Multi-AZ RDS in production but single-AZ in dev.

### Defining conditions

```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, production]

Conditions:
  IsProduction: !Equals [!Ref Environment, production]
  IsDev:        !Equals [!Ref Environment, dev]
  IsMultiAZ:    !Equals [!Ref Environment, production]
```

### Condition functions

| Function | Syntax | Meaning |
|:---------|:-------|:--------|
| `!Equals` | `!Equals [val1, val2]` | True if val1 == val2 |
| `!Not` | `!Not [condition]` | Negates a condition |
| `!And` | `!And [cond1, cond2]` | True if both are true |
| `!Or` | `!Or [cond1, cond2]` | True if either is true |

### Using conditions on resources

```yaml
Resources:
  # Only create a NAT Gateway in production
  NatGateway:
    Type: AWS::EC2::NatGateway
    Condition: IsProduction       # entire resource skipped if false
    Properties:
      AllocationId: !GetAtt ElasticIP.AllocationId
      SubnetId: !Ref PublicSubnet

  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: db.t3.micro
      # Use condition inline to pick a value
      MultiAZ: !If [IsProduction, true, false]
      AllocatedStorage: !If [IsProduction, "100", "20"]
```

`!If [ConditionName, ValueIfTrue, ValueIfFalse]` — pick one of two values based on a condition.

---

## 7. Resources — The Core Section

`Resources` is the only mandatory section. Every resource you want CloudFormation to create goes here.

### Resource structure

```yaml
Resources:
  LogicalID:           # You choose this name — used to reference this resource
    Type: AWS::Service::ResourceType   # from the CloudFormation resource registry
    DependsOn: OtherLogicalID          # optional: explicit dependency
    DeletionPolicy: Retain             # optional: what to do on stack delete
    UpdateReplacePolicy: Retain        # optional: what to do on replacement
    Properties:
      PropertyOne: value
      PropertyTwo: value
```

### LogicalID rules

- Must be unique within the template
- Letters and numbers only — no hyphens or underscores
- Convention: PascalCase (e.g., `WebServer`, `PublicSubnet1`, `AppLoadBalancer`)

### Type format: `AWS::Service::Resource`

```
AWS::EC2::Instance          → EC2 instance
AWS::EC2::VPC               → VPC
AWS::EC2::Subnet            → Subnet
AWS::EC2::InternetGateway   → Internet gateway
AWS::EC2::RouteTable        → Route table
AWS::EC2::NatGateway        → NAT gateway
AWS::EC2::SecurityGroup     → Security group
AWS::EC2::NetworkAcl        → Network ACL
AWS::RDS::DBInstance        → RDS database instance
AWS::RDS::DBSubnetGroup     → RDS subnet group
AWS::ElasticLoadBalancingV2::LoadBalancer   → ALB or NLB
AWS::ElasticLoadBalancingV2::TargetGroup    → Target group
AWS::ElasticLoadBalancingV2::Listener       → LB listener
AWS::Lambda::Function       → Lambda function
AWS::SQS::Queue             → SQS queue
AWS::SNS::Topic             → SNS topic
AWS::S3::Bucket             → S3 bucket
AWS::IAM::Role              → IAM role
AWS::IAM::Policy            → IAM policy
AWS::AutoScaling::AutoScalingGroup → Auto Scaling group
AWS::AutoScaling::LaunchTemplate   → Launch template
```

### DeletionPolicy — protecting resources

```yaml
MyDatabase:
  Type: AWS::RDS::DBInstance
  DeletionPolicy: Snapshot    # take a final snapshot before deleting
  Properties: ...

MyBucket:
  Type: AWS::S3::Bucket
  DeletionPolicy: Retain      # keep the bucket even after stack delete
  Properties: ...
```

| DeletionPolicy | Behavior on stack delete |
|:---------------|:------------------------|
| `Delete` | Resource is deleted (default for most resources) |
| `Retain` | Resource stays, stack just stops managing it |
| `Snapshot` | RDS/EBS: take a snapshot then delete (data preserved) |

---

## 8. Outputs — Sharing Values Between Stacks

Outputs let you export values from a stack so other stacks (or you) can use them.

```yaml
Outputs:
  VpcId:
    Description: The ID of the VPC
    Value: !Ref MyVPC
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"   # exported as "my-stack-VpcId"

  PublicSubnet1Id:
    Description: Public Subnet 1 ID
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnet1"

  LoadBalancerDNS:
    Description: DNS name of the ALB
    Value: !GetAtt AppLoadBalancer.DNSName
    # No Export — just visible in console, not importable by other stacks
```

### Using an exported value in another stack

```yaml
# In stack B, import a value exported by stack A
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !ImportValue "my-network-stack-PublicSubnet1"
```

**Constraint:** You cannot delete a stack if another stack imports its exports. Remove the import first.

---

## 9. Intrinsic Functions — The Power Tools

Intrinsic functions let you compute values dynamically in your template instead of hardcoding them. You can use them in any value position.

### All major intrinsic functions

```yaml
# !Ref — reference a parameter or resource
VpcId: !Ref MyVPC                  # for a resource: returns the resource's ID/name
InstanceType: !Ref InstanceTypeParam  # for a parameter: returns the param value

# !GetAtt — get an attribute of a resource
DNSName: !GetAtt AppLoadBalancer.DNSName
AllocationId: !GetAtt ElasticIP.AllocationId
ARN: !GetAtt MyLambda.Arn

# !Sub — string substitution (most useful for building ARNs and names)
BucketName: !Sub "my-app-${Environment}-${AWS::AccountId}"
RoleArn: !Sub "arn:aws:iam::${AWS::AccountId}:role/my-role"

# !Join — join a list of values with a delimiter
CIDR: !Join [".", [10, 0, 0, 0]]           # → "10.0.0.0"
MultiValue: !Join [",", [a, b, c]]          # → "a,b,c"

# !Select — pick one item from a list by index
AvailabilityZone: !Select [0, !GetAZs ""]  # first AZ in current region

# !GetAZs — returns a list of AZs in a region
FirstAZ: !Select [0, !GetAZs !Ref "AWS::Region"]
SecondAZ: !Select [1, !GetAZs !Ref "AWS::Region"]

# !If — conditional value
MultiAZ: !If [IsProduction, true, false]
DBClass: !If [IsProduction, db.m5.large, db.t3.micro]

# !Equals — used in Conditions section
IsProduction: !Equals [!Ref Environment, production]

# !Not — negation, used in Conditions
IsNotProduction: !Not [!Equals [!Ref Environment, production]]

# !And / !Or — combine conditions
IsProdMultiRegion: !And [!Condition IsProduction, !Condition IsMultiRegion]

# !ImportValue — import an output from another stack
SubnetId: !ImportValue "network-stack-PublicSubnet1"

# !Base64 — encode string to Base64 (used for EC2 UserData)
UserData:
  Fn::Base64: |
    #!/bin/bash
    yum update -y
```

### Short form vs long form

```yaml
# Short form (YAML tag syntax) — preferred for readability
Value: !Ref MyParam
Name: !Sub "prefix-${MyParam}"

# Long form (mapping syntax) — required when nesting functions
Value:
  Fn::Select:
    - 0
    - Fn::GetAZs: !Ref "AWS::Region"
```

Use short form wherever possible. Use long form when you need to nest one function inside another.

### Nesting functions — common patterns

```yaml
# Sub + Join to build a comma-separated list of ARNs
PolicyDocument:
  Statement:
    - Action: "s3:GetObject"
      Resource:
        - !Sub "arn:aws:s3:::${BucketName}/*"

# Select from GetAZs to place subnet in first AZ automatically
Subnet1:
  Type: AWS::EC2::Subnet
  Properties:
    AvailabilityZone: !Select [0, !GetAZs !Ref "AWS::Region"]
    CidrBlock: !Select [0, !Cidr [!GetAtt VPC.CidrBlock, 4, 8]]
```

---

## 10. Pseudo Parameters

Pseudo parameters are built-in parameters that CloudFormation automatically provides. You don't declare them — they are always available.

| Pseudo Parameter | Returns | Example Value |
|:-----------------|:--------|:--------------|
| `AWS::AccountId` | Your AWS account ID | `123456789012` |
| `AWS::Region` | Region being deployed to | `us-east-1` |
| `AWS::StackName` | Name of the current stack | `my-app-prod` |
| `AWS::StackId` | Full ARN of the stack | `arn:aws:cloudformation:...` |
| `AWS::NoValue` | Removes a property (used with `!If`) | — |
| `AWS::Partition` | AWS partition | `aws` (or `aws-cn` for China) |
| `AWS::URLSuffix` | Domain suffix | `amazonaws.com` |

### Common uses

```yaml
# Include account ID in bucket name to ensure global uniqueness
BucketName: !Sub "my-app-logs-${AWS::AccountId}"

# Include stack name in resource names so multiple stacks don't conflict
Name: !Sub "${AWS::StackName}-web-server"

# Build IAM ARN for the current region and account
Resource: !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:*"
```

---

## 11. Creating and Uploading Templates

### Method 1: AWS Management Console

```
1. Open CloudFormation Console
   → Services → CloudFormation → Stacks → Create Stack

2. Choose template source:
   ├── Upload a template file   ← browse for your .yaml file
   ├── Amazon S3 URL            ← paste S3 URL if template is in S3
   └── Use Infrastructure Composer  ← visual designer (see Section 13)

3. Specify stack details:
   ├── Stack name: my-app-prod
   └── Parameters: fill in the form (one field per parameter)

4. Configure stack options:
   ├── Tags: add tags to all resources
   ├── Permissions: IAM role for CloudFormation
   └── Stack failure options: Roll back / Preserve resources

5. Review → Create stack
```

### Method 2: AWS CLI

```bash
# Deploy (create or update) a stack
aws cloudformation deploy \
  --stack-name my-app-prod \
  --template-file template.yaml \
  --parameter-overrides \
    Environment=production \
    InstanceType=m5.large \
  --capabilities CAPABILITY_IAM \
  --region us-east-1

# Create a new stack (fails if it already exists)
aws cloudformation create-stack \
  --stack-name my-app-dev \
  --template-file template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=dev

# Update an existing stack
aws cloudformation update-stack \
  --stack-name my-app-dev \
  --template-file template.yaml

# Describe stack events (useful for debugging)
aws cloudformation describe-stack-events \
  --stack-name my-app-prod

# Delete a stack
aws cloudformation delete-stack \
  --stack-name my-app-dev
```

### Method 3: Upload template to S3 first (large templates)

CloudFormation has a 51,200 byte limit for direct uploads. For large templates, put them in S3 first:

```bash
# Upload template to S3
aws s3 cp template.yaml s3://my-cfn-templates/template.yaml

# Deploy from S3
aws cloudformation deploy \
  --stack-name my-app \
  --template-url https://s3.amazonaws.com/my-cfn-templates/template.yaml
```

### Template validation before deploy

Always validate before deploying to catch YAML/schema errors early:

```bash
# Validate syntax and schema
aws cloudformation validate-template \
  --template-body file://template.yaml

# Lint with cfn-lint (more thorough)
cfn-lint template.yaml
```

---

## 12. VS Code AWS Toolkit — Visualize While You Write

### Install AWS Toolkit

1. Open VS Code Extensions (`Ctrl+Shift+X`)
2. Search for **AWS Toolkit**
3. Install (publisher: Amazon Web Services)
4. Sign in: open AWS Explorer panel (left sidebar) → Connect to AWS → choose your profile

### CloudFormation template features in AWS Toolkit

```
While editing a .yaml file:
─────────────────────────────────────────────────────────────────────────
  ✓ Hover over a resource Type → see documentation inline
  ✓ Auto-complete for resource types (AWS::EC2::...)
  ✓ Auto-complete for properties (start typing under Properties:)
  ✓ Error highlights for missing required properties
  ✓ Open in Infrastructure Composer from the editor title bar
  ✓ Deploy directly: right-click file → Deploy SAM Application
```

### cfn-lint integration

With cfn-lint installed and the Red Hat YAML extension:

```
editor shows:
  Red underline: YAML syntax error
  Yellow underline: cfn-lint warning (deprecated property, etc.)
  Orange underline: cfn-lint error (wrong resource type, bad property)
```

---

## 13. Infrastructure Composer — Visual Template Builder

Infrastructure Composer (previously CloudFormation Designer) is the AWS browser-based visual editor. It shows your template as a drag-and-drop diagram.

### Accessing Infrastructure Composer

```
Option 1: AWS Console
  CloudFormation → Create Stack → Design template in Infrastructure Composer

Option 2: From existing template
  CloudFormation → Stacks → select a stack → Template tab
  → View in Infrastructure Composer

Option 3: Directly
  https://console.aws.amazon.com/composer/home
```

### What you can do in Infrastructure Composer

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Infrastructure Composer                          │
│                                                                      │
│  Resource Panel    Canvas                        Template Panel      │
│  ┌────────────┐   ┌──────────────────────────┐  ┌────────────────┐  │
│  │ Search...  │   │                          │  │ AWSTemplate..  │  │
│  │            │   │  [VPC]                   │  │ FormatVersion  │  │
│  │ ▶ EC2      │   │    │                     │  │ Resources:     │  │
│  │ ▶ VPC      │   │  [Subnet]  [Subnet]      │  │   MyVPC:       │  │
│  │ ▶ RDS      │   │    │          │          │  │     Type: ...  │  │
│  │ ▶ Lambda   │   │  [EC2]  [ALB]───[EC2]   │  │                │  │
│  │ ▶ S3       │   │                          │  │ ← live YAML    │  │
│  │ ▶ SQS      │   │  Drag resources here     │  │   editing      │  │
│  └────────────┘   └──────────────────────────┘  └────────────────┘  │
│                                                                      │
│  ← Drag & drop resources   Click connections to set relationships    │
└──────────────────────────────────────────────────────────────────────┘
```

### Key features

- Drag resources from the left panel onto the canvas
- Draw connections between resources (auto-generates `!Ref`)
- Edit YAML in the right panel — canvas updates in real time
- Click a resource on the canvas → configure its properties
- Export as YAML or JSON

> **Tip:** Infrastructure Composer is excellent for understanding existing templates. Paste a complex template and let it render visually. For writing templates, VS Code + cfn-lint gives you more control.

---

## 14. Deploying and Updating Stacks

### Stack creation — what CloudFormation does

CloudFormation analyzes all `!Ref` and `!GetAtt` calls in your template to build a dependency graph. Resources that depend on each other are created in order; independent resources are created in parallel.

```
Example: VPC template dependency order
──────────────────────────────────────────────────
Step 1 (parallel):  VPC, InternetGateway
Step 2 (parallel):  PublicSubnet, PrivateSubnet, RouteTable
                    (all depend on VPC being ready)
Step 3 (parallel):  VPCGatewayAttachment, NatGatewayEIP
                    (VPCGatewayAttachment needs both VPC and IGW)
Step 4:             NatGateway (needs subnet + EIP)
Step 5 (parallel):  PublicRoute, PrivateRoute
Step 6 (parallel):  SubnetRouteTableAssociation × N
```

### Updating a stack

1. Edit your template (change a property, add a resource, etc.)
2. Deploy the updated template to the same stack name
3. CloudFormation compares the old and new template — this is called a **change set**
4. Only affected resources are updated; unaffected resources are left alone

### Update behaviors — how resources get updated

```
Resource update behavior depends on the property changed:

  No interruption:   Update in place, resource ID stays the same
                     e.g., change EC2 instance tags

  Some interruption: Brief downtime but same resource ID
                     e.g., change EC2 instance type (requires restart)

  Replacement:       Old resource deleted, new one created with new ID
                     e.g., change VPC CIDR block
                     WARNING: This causes downtime and data loss for stateful resources
```

CloudFormation shows you which update behavior applies in the Change Set preview.

---

## 15. Change Sets — Preview Before You Apply

A Change Set is a preview of what will happen to your stack when you apply an updated template. Always use Change Sets in production.

### Create a Change Set via Console

```
CloudFormation → Stacks → select stack
→ Stack actions → Create change set for current stack
→ Upload new template → give the change set a name
→ Review → CloudFormation shows you:
    - Added resources
    - Modified resources (and whether update causes replacement)
    - Removed resources
→ Execute change set → changes are applied
```

### Create a Change Set via CLI

```bash
# Create the change set
aws cloudformation create-change-set \
  --stack-name my-app-prod \
  --template-file updated-template.yaml \
  --change-set-name my-update-v2 \
  --parameters ParameterKey=Environment,ParameterValue=production

# Describe the change set (see what will change)
aws cloudformation describe-change-set \
  --stack-name my-app-prod \
  --change-set-name my-update-v2

# Execute it (apply the changes)
aws cloudformation execute-change-set \
  --stack-name my-app-prod \
  --change-set-name my-update-v2
```

---

## 16. Stack Drift Detection

Drift occurs when someone manually changes a resource that CloudFormation manages — through the console, CLI, or SDK — without going through CloudFormation. The stack template and the actual resource are now out of sync.

```
CloudFormation template says:  SecurityGroup allows port 80
Manual change in console:      Admin added port 22 inbound rule
→ DRIFT: the actual resource differs from the template
```

### Detecting drift

```
Console:
  CloudFormation → Stacks → select stack
  → Stack actions → Detect drift
  → Wait for detection to complete
  → View drift results

Result shows:
  IN_SYNC:  resource matches template
  MODIFIED: resource was changed outside CloudFormation
  DELETED:  resource was deleted outside CloudFormation
  NOT_CHECKED: drift detection not supported for this resource type
```

### Fixing drift

Options:
1. **Update the template** to match the current actual state (accept the manual change)
2. **Re-deploy the template** to revert the manual change back to what the template says
3. **Import the resource** into the stack with the new configuration

---

## 17. Nested Stacks

For large environments, a single template becomes hard to manage. Nested stacks let you split one big template into multiple smaller ones.

```yaml
# Parent stack references child stack templates
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/network.yaml
      Parameters:
        Environment: !Ref Environment
        VpcCIDR: "10.0.0.0/16"

  AppStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/app.yaml
      Parameters:
        Environment: !Ref Environment
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        SubnetId: !GetAtt NetworkStack.Outputs.PublicSubnet1Id
```

The parent passes the child's output (`NetworkStack.Outputs.VpcId`) as parameters to the next child. Child templates must be in S3 — they cannot be local files when used in nested stacks.

---

## Complete Examples → See Part 3

Ready-to-deploy templates for all core services are in **Part 3 CloudFormation Complete Examples\.md**:

- Example 1 — Full VPC with Subnets, NAT, NACLs
- Example 2 — EC2 with Security Group and UserData
- Example 3 — ALB + Target Group + EC2
- Example 4 — RDS MySQL with Multi-AZ

---

## Tasks and Solutions → See Part 4

Hands-on tasks and collapsible solutions are in **Part 4 CloudFormation Tasks and Solutions\.md**:

- Task 1 — VPC + EC2 + S3 Foundation Stack
- Task 2 — Lambda + SQS + S3 Event Pipeline
- Task 3 — Full Production Stack (VPC + ALB + EC2 ASG + RDS)
- Task 4 — Basic Infrastructure (VPC + EC2 + IAM + httpd)

---

*End of Part 2 — Concepts Reference*
