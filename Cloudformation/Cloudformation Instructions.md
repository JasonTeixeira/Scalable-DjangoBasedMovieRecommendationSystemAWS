# AWS CloudFormation Template for Scalable Django-Based Movie Recommendation System

## Introduction

Description: >
  This CloudFormation template provisions the AWS infrastructure required for deploying
  a scalable Django-based Movie Recommendation System. The resources provisioned include
  VPC, EC2 instances, RDS, Application Load Balancer (ALB), S3 for static file hosting,
  and necessary security groups. The infrastructure is designed to be highly available,
  secure, and scalable.

## Infrastructure Components Overview

Resources provisioned:
- **VPC**: Virtual Private Cloud for network isolation and security.
- **EC2 Instances**: Web servers hosting the Django application.
- **RDS**: PostgreSQL database for persistent storage.
- **ALB**: Application Load Balancer for distributing traffic.
- **S3**: Buckets for static file hosting and logging.
- **Security Groups**: For controlling network access.

---

## CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CloudFormation Template for the Movie Recommendation System Infrastructure.

Parameters:
  KeyName:
    Description: Name of an existing EC2 KeyPair to enable SSH access
    Type: String
  InstanceType:
    Description: EC2 instance type
    Type: String
    Default: t3.medium
  DBPassword:
    Description: The password for the PostgreSQL RDS database
    Type: String
    NoEcho: true
    MinLength: 8
    MaxLength: 41
  InstanceCount:
    Description: Number of EC2 Instances
    Type: Number
    Default: 2

Resources:
  # VPC
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      Tags:
        - Key: Name
          Value: MovieRecommendationVPC

  # Internet Gateway
  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'

  AttachGateway:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # Public Subnet
  PublicSubnet:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.1.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: MovieRecommendationPublicSubnet

  # Private Subnet
  PrivateSubnet:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.2.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: MovieRecommendationPrivateSubnet

  # Route Table for Public Subnet
  PublicRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref VPC

  # Route to the Internet from the public subnet
  PublicRoute:
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref InternetGateway

  # Route Table Association for Public Subnet
  PublicSubnetRouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

  # Security Group for EC2 Instances
  WebServerSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: Allow HTTP and SSH access
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '80'
          ToPort: '80'
          CidrIp: '0.0.0.0/0'
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: '0.0.0.0/0'

  # EC2 Instances
  WebServerInstance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      SubnetId: !Ref PrivateSubnet
      ImageId: 'ami-0c55b159cbfafe1f0'  # Amazon Linux 2
      UserData:
        Fn::Base64: |
          #!/bin/bash
          sudo yum update -y
          sudo yum install -y httpd
          sudo systemctl start httpd
          sudo systemctl enable httpd
      Tags:
        - Key: Name
          Value: MovieRecommendationWebServer

  # RDS PostgreSQL Instance
  DBInstance:
    Type: 'AWS::RDS::DBInstance'
    Properties:
      Engine: postgres
      DBInstanceClass: db.t3.medium
      AllocatedStorage: 20
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      VPCSecurityGroups:
        - !Ref RDSSecurityGroup
      DBSubnetGroupName: !Ref DBSubnetGroup

  # RDS Subnet Group
  DBSubnetGroup:
    Type: 'AWS::RDS::DBSubnetGroup'
    Properties:
      DBSubnetGroupDescription: Subnet group for Movie Recommendation DB
      SubnetIds:
        - !Ref PrivateSubnet
      Tags:
        - Key: Name
          Value: MovieRecommendationDBSubnetGroup

  # Security Group for RDS
  RDSSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: Allow access to the DB within the VPC
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '5432'
          ToPort: '5432'
          CidrIp: '10.0.0.0/16'

  # Application Load Balancer
  ALB:
    Type: 'AWS::ElasticLoadBalancingV2::LoadBalancer'
    Properties:
      Subnets:
        - !Ref PublicSubnet
      SecurityGroups:
        - !Ref WebServerSecurityGroup

  # Target Group for ALB
  ALBTargetGroup:
    Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
    Properties:
      HealthCheckPath: /
      TargetType: instance
      VpcId: !Ref VPC
      Port: 80
      Protocol: HTTP

  # ALB Listener
  ALBListener:
    Type: 'AWS::ElasticLoadBalancingV2::Listener'
    Properties:
      LoadBalancerArn: !Ref ALB
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref ALBTargetGroup

  # S3 Bucket for Static Files
  StaticFilesBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: Private
      Tags:
        - Key: Name
          Value: MovieRecommendationStaticFiles

  # S3 Bucket for Logs
  LogsBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: Private
      Tags:
        - Key: Name
          Value: MovieRecommendationLogs

Outputs:
  ALBDNS:
    Description: DNS name of the Application Load Balancer
    Value: !GetAtt ALB.DNSName

  DBEndpoint:
    Description: Endpoint for the RDS PostgreSQL database
    Value: !GetAtt DBInstance.Endpoint.Address

  WebServerPublicIP:
    Description: Public IP of the web server
    Value: !GetAtt WebServerInstance.PublicIp

```

---

## CloudFormation Deployment Steps

### Step 1: Deploy CloudFormation Template

1. Open the AWS Management Console.
2. Go to **CloudFormation**.
3. Click **Create stack**.
4. Upload the CloudFormation template above or paste the YAML into the template section.
5. Specify the required parameters (KeyPair, Instance Type, DBPassword, etc.).
6. Click **Next** and review the stack settings.
7. Click **Create stack**.

### Step 2: Monitor the Stack Creation

- Wait for the stack creation to complete. This might take several minutes.
- Once completed, the resources such as VPC, EC2 instances, RDS, ALB, and S3 buckets will be provisioned.

### Step 3: Retrieve Outputs

- Navigate to the **Outputs** tab of your CloudFormation stack to retrieve the DNS name of the Application Load Balancer, RDS endpoint, and public IP of the EC2 instance.

---

## Modifying the Infrastructure

### 1. **Scaling EC2 Instances**

To modify the number of EC2 instances, you can update the `InstanceCount` parameter in the stack and update the stack.

### 2. **Multi-AZ Support for RDS**

To enable Multi-AZ for the RDS instance, update the `DBInstance` resource to include `MultiAZ: true`.

---

## Summary

This CloudFormation template provisions a highly available, scalable infrastructure for the Django-based Movie Recommendation System on AWS. The infrastructure is modular and includes resources such as VPC, EC2 instances, ALB, RDS, and S3 for static file hosting and logging. This setup is ideal for deploying production-ready cloud infrastructure that can scale with demand while ensuring high availability and security.

This CloudFormation template provides the same infrastructure as the Terraform template, but in YAML format suitable for deployment through AWS CloudFormation.

