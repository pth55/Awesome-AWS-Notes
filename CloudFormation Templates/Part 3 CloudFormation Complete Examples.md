# Part 3: CloudFormation — Complete Examples

---

## Table of Contents

1. [Example 1 — Full VPC with Subnets, NAT, NACLs](#1-example-1--full-vpc-with-subnets-nat-nacls)
2. [Example 2 — EC2 with Security Group and UserData](#2-example-2--ec2-with-security-group-and-userdata)
3. [Example 3 — ALB + Target Group + EC2](#3-example-3--alb--target-group--ec2)
4. [Example 4 — RDS MySQL with Multi-AZ](#4-example-4--rds-mysql-with-multi-az)

---

## 1. Example 1 — Full VPC with Subnets, NAT, NACLs

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: VPC with public/private subnets, IGW, NAT Gateway, and basic NACLs

Parameters:
  VpcCIDR:
    Type: String
    Default: "10.0.0.0/16"
  PublicSubnet1CIDR:
    Type: String
    Default: "10.0.1.0/24"
  PrivateSubnet1CIDR:
    Type: String
    Default: "10.0.2.0/24"
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, production]

Resources:
  # ─── VPC ─────────────────────────────────────────────────────────────
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-vpc"

  # ─── Internet Gateway ─────────────────────────────────────────────────
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-igw"

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  # ─── Subnets ──────────────────────────────────────────────────────────
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PublicSubnet1CIDR
      AvailabilityZone: !Select [0, !GetAZs !Ref "AWS::Region"]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-subnet-1"

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PrivateSubnet1CIDR
      AvailabilityZone: !Select [0, !GetAZs !Ref "AWS::Region"]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-private-subnet-1"

  # ─── NAT Gateway ──────────────────────────────────────────────────────
  NatGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: VPCGatewayAttachment
    Properties:
      Domain: vpc

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-nat-gw"

  # ─── Route Tables ─────────────────────────────────────────────────────
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-rt"

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-private-rt"

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGateway

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable

  # ─── Network ACL (public subnet) ──────────────────────────────────────
  PublicNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-nacl"

  PublicNACLInboundHTTP:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PublicNACL
      RuleNumber: 100
      Protocol: 6    # TCP
      RuleAction: allow
      Egress: false
      CidrBlock: "0.0.0.0/0"
      PortRange:
        From: 80
        To: 80

  PublicNACLInboundHTTPS:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PublicNACL
      RuleNumber: 110
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: "0.0.0.0/0"
      PortRange:
        From: 443
        To: 443

  PublicNACLInboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PublicNACL
      RuleNumber: 120
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: "0.0.0.0/0"
      PortRange:
        From: 1024
        To: 65535

  PublicNACLOutboundAll:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PublicNACL
      RuleNumber: 100
      Protocol: -1    # all
      RuleAction: allow
      Egress: true
      CidrBlock: "0.0.0.0/0"

  PublicSubnetNACLAssociation:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      NetworkAclId: !Ref PublicNACL

Outputs:
  VpcId:
    Value: !Ref MyVPC
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"
  PublicSubnet1Id:
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnet1Id"
  PrivateSubnet1Id:
    Value: !Ref PrivateSubnet1
    Export:
      Name: !Sub "${AWS::StackName}-PrivateSubnet1Id"
```

---

## 2. Example 2 — EC2 with Security Group and UserData

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 web server with security group and Nginx bootstrapped via UserData

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Name of an existing EC2 key pair
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetId:
    Type: AWS::EC2::Subnet::Id

Mappings:
  RegionAMI:
    us-east-1:
      AL2: ami-0c02fb55956c7d316
    us-west-2:
      AL2: ami-0ca285d4c2cda3300
    eu-west-1:
      AL2: ami-0d71ea30463e0ff49

Resources:
  WebServerSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server security group
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: "0.0.0.0/0"
          Description: SSH access
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: "0.0.0.0/0"
          Description: HTTP access
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: "0.0.0.0/0"
          Description: HTTPS access
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web-sg"

  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionAMI, !Ref "AWS::Region", AL2]
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SubnetId: !Ref SubnetId
      SecurityGroupIds:
        - !Ref WebServerSG
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          amazon-linux-extras install -y nginx1
          systemctl start nginx
          systemctl enable nginx
          echo "<h1>Deployed via CloudFormation</h1>" > /usr/share/nginx/html/index.html
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web-server"

Outputs:
  PublicIP:
    Description: Public IP of the web server
    Value: !GetAtt WebServerInstance.PublicIp
  PublicDNS:
    Description: Public DNS name
    Value: !GetAtt WebServerInstance.PublicDnsName
```

---

## 3. Example 3 — ALB + Target Group + EC2

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: ALB with two EC2 targets

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
  PublicSubnet1:
    Type: AWS::EC2::Subnet::Id
  PublicSubnet2:
    Type: AWS::EC2::Subnet::Id
  PrivateSubnet1:
    Type: AWS::EC2::Subnet::Id
  PrivateSubnet2:
    Type: AWS::EC2::Subnet::Id
  AmiId:
    Type: AWS::EC2::Image::Id
    Default: ami-0c02fb55956c7d316

Resources:
  # Security group for ALB
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB security group
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: "0.0.0.0/0"

  # Security group for EC2 — only allows traffic from ALB
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App server security group
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  # EC2 instances
  AppServer1:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AmiId
      InstanceType: t3.micro
      SubnetId: !Ref PrivateSubnet1
      SecurityGroupIds: [!Ref AppSecurityGroup]
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum install -y httpd
          systemctl start httpd
          echo "Server 1" > /var/www/html/index.html
      Tags:
        - Key: Name
          Value: app-server-1

  AppServer2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AmiId
      InstanceType: t3.micro
      SubnetId: !Ref PrivateSubnet2
      SecurityGroupIds: [!Ref AppSecurityGroup]
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum install -y httpd
          systemctl start httpd
          echo "Server 2" > /var/www/html/index.html
      Tags:
        - Key: Name
          Value: app-server-2

  # Target Group
  AppTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub "${AWS::StackName}-tg"
      VpcId: !Ref VpcId
      Protocol: HTTP
      Port: 80
      TargetType: instance
      HealthCheckPath: /
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      Targets:
        - Id: !Ref AppServer1
          Port: 80
        - Id: !Ref AppServer2
          Port: 80

  # Application Load Balancer
  AppLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub "${AWS::StackName}-alb"
      Scheme: internet-facing
      Type: application
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup

  # Listener
  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref AppLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref AppTargetGroup

Outputs:
  LoadBalancerDNS:
    Description: DNS name of the ALB
    Value: !GetAtt AppLoadBalancer.DNSName
```

---

## 4. Example 4 — RDS MySQL with Multi-AZ

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: RDS MySQL with subnet group and parameter group

Parameters:
  DBName:
    Type: String
    Default: myappdb
  DBUsername:
    Type: String
    Default: admin
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
  DBInstanceClass:
    Type: String
    Default: db.t3.micro
    AllowedValues: [db.t3.micro, db.t3.small, db.m5.large]
  MultiAZ:
    Type: String
    Default: "false"
    AllowedValues: ["true", "false"]
  VpcId:
    Type: AWS::EC2::VPC::Id
  PrivateSubnet1:
    Type: AWS::EC2::Subnet::Id
  PrivateSubnet2:
    Type: AWS::EC2::Subnet::Id

Resources:
  # Security group for RDS — only allows traffic from app servers
  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: RDS MySQL security group
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          CidrIp: "10.0.0.0/16"   # allow from entire VPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-rds-sg"

  # DB Subnet Group — RDS requires subnets in at least 2 AZs
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: RDS subnet group
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-db-subnet-group"

  # RDS Parameter Group (optional — use default MySQL 8.0)
  DBParameterGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties:
      Family: mysql8.0
      Description: Custom MySQL 8.0 parameter group
      Parameters:
        max_connections: "200"
        slow_query_log: "1"
        long_query_time: "2"

  # RDS Instance
  DBInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      DBInstanceIdentifier: !Sub "${AWS::StackName}-mysql"
      DBName: !Ref DBName
      DBInstanceClass: !Ref DBInstanceClass
      Engine: MySQL
      EngineVersion: "8.0"
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      AllocatedStorage: "20"
      StorageType: gp3
      MultiAZ: !Ref MultiAZ
      DBSubnetGroupName: !Ref DBSubnetGroup
      DBParameterGroupName: !Ref DBParameterGroup
      VPCSecurityGroups:
        - !Ref DBSecurityGroup
      BackupRetentionPeriod: 7
      StorageEncrypted: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-mysql"

Outputs:
  DBEndpoint:
    Description: RDS endpoint address
    Value: !GetAtt DBInstance.Endpoint.Address
    Export:
      Name: !Sub "${AWS::StackName}-DBEndpoint"
  DBPort:
    Description: RDS port
    Value: !GetAtt DBInstance.Endpoint.Port
```
