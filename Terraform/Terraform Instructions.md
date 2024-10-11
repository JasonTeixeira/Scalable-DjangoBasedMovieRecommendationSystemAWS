## Introduction

**Project Context**

The Movie Recommendation System is built using the Django framework and hosted on AWS infrastructure. By using Terraform, the infrastructure can be provisioned in a consistent, automated, and scalable manner across environments. This document outlines how to set up the project’s cloud resources using Terraform.

---

## Infrastructure Components Overview

The AWS infrastructure for this project consists of:

- **Amazon VPC**: Virtual Private Cloud setup with public and private subnets for secure networking.
- **Amazon EC2 Instances**: Web and application servers for the Django app.
- **Amazon RDS**: PostgreSQL for persistent data storage.
- **Application Load Balancer (ALB)**: Distributes traffic across EC2 instances.
- **Amazon CloudFront**: CDN to optimize content delivery.
- **Security Groups**: Control access between resources.
- **IAM Roles and Policies**: Manage identity and access control.

---

## Terraform File Structure

The Terraform files are structured as follows:

```bash
bash
Copy code
/terraform
  ├── main.tf
  ├── variables.tf
  ├── outputs.tf
  ├── vpc.tf
  ├── ec2.tf
  ├── rds.tf
  ├── alb.tf
  ├── security-groups.tf
  ├── s3.tf
  └── provider.tf

```

- **main.tf**: Executes resource provisioning.
- **variables.tf**: Contains input variables (e.g., instance types).
- **outputs.tf**: Outputs DNS names, IP addresses.
- **vpc.tf**: Defines networking components.
- **ec2.tf**: Configures EC2 instances.
- **rds.tf**: Manages the PostgreSQL instance.
- **alb.tf**: Sets up the application load balancer.
- **security-groups.tf**: Manages security groups.
- **s3.tf**: Manages S3 for static files.

---

## Terraform Deployment Steps

### Step 1: Install Terraform and AWS CLI

Ensure you have Terraform and the AWS CLI installed.

```bash
bash
Copy code
# Install AWS CLI
$ aws configure

# Install Terraform
$ brew install terraform

```

### Step 2: Initialize Terraform

In the project directory, initialize Terraform.

```bash
bash
Copy code
$ cd terraform
$ terraform init

```

### Step 3: Define Provider and Variables

In `provider.tf`, define the AWS provider and region.

```
hcl
Copy code
provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  default = "us-east-1"
}

```

In `variables.tf`, define variables for flexibility.

```
hcl
Copy code
variable "ec2_instance_type" {
  default = "t3.medium"
}

```

### Step 4: Create the VPC

Create a VPC with public and private subnets.

```
hcl
Copy code
resource "aws_vpc" "movie_recommendation_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.movie_recommendation_vpc.id
}

```

### Step 5: Launch EC2 Instances

Deploy EC2 instances in the private subnet for the Django application.

```
hcl
Copy code
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.ec2_instance_type
  subnet_id     = aws_subnet.private_subnet.id

  tags = {
    Name = "MovieRecommendationWebServer"
  }
}

```

### Step 6: Setup RDS

Create an RDS PostgreSQL database.

```
hcl
Copy code
resource "aws_db_instance" "movie_db" {
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  allocated_storage = 20
  name              = "moviesdb"
  username          = "admin"
  password          = var.db_password
  publicly_accessible = false
}

```

### Step 7: Configure the Application Load Balancer

Set up the Application Load Balancer (ALB).

```
hcl
Copy code
resource "aws_lb" "app_lb" {
  name               = "movie-recommendation-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.lb_sg.id]
}

```

### Step 8: Set Up Security Groups

Define security groups to restrict access to EC2 instances and RDS.

```
hcl
Copy code
resource "aws_security_group" "ec2_sg" {
  name_prefix = "ec2-sg"
  description = "Allow HTTP and SSH"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

```

### Step 9: Apply the Terraform Configuration

Apply the Terraform configuration to provision the infrastructure.

```bash
bash
Copy code
$ terraform apply

```

### Step 10: Retrieve Outputs

Retrieve the critical information such as DNS names and IP addresses.

```
hcl
Copy code
output "app_load_balancer_dns" {
  value = aws_lb.app_lb.dns_name
}

```

---

## Modifying the Terraform Infrastructure

To make changes to the infrastructure, follow these steps:

### 1. **Scaling EC2 Instances:**

Adjust the `count` parameter to scale the number of EC2 instances.

```
hcl
Copy code
resource "aws_instance" "web_server" {
  count = var.instance_count
}

```

### 2. **Multi-AZ Support for RDS:**

Enable high availability for the RDS instance by updating the `multi_az` option.

```
hcl
Copy code
resource "aws_db_instance" "movie_db" {
  multi_az = true
}

```

### 3. **Auto-scaling Configuration:**

Define auto-scaling policies for EC2 instances.

```
hcl
Copy code
resource "aws_autoscaling_group" "asg" {
  desired_capacity     = 2
  max_size             = 5
  min_size             = 1
}

```

---

## Full Output

```markdown
markdown
Copy code
# Terraform Infrastructure for Scalable Django-Based Movie Recommendation System on AWS

## Introduction

This Terraform configuration document provides the infrastructure-as-code (IaC) solution for deploying a scalable Django-based Movie Recommendation System on AWS. The infrastructure leverages AWS services such as EC2, RDS, ALB (Application Load Balancer), VPC (Virtual Private Cloud), S3, and Security Groups. The purpose of this setup is to automate infrastructure provisioning, ensuring high availability, security, and scalability for a production-ready web application.

The infrastructure is designed to support flexible scaling of EC2 instances, a relational PostgreSQL database (RDS), static file hosting via S3, and efficient load balancing. All resources are defined in modular Terraform files to enable seamless management and future enhancements.

---

## Terraform Configuration

```hcl
# provider.tf

# Specify the AWS provider and region
provider "aws" {
  region = var.aws_region
}

```

```
hcl
Copy code
# variables.tf

# Define configurable variables for the infrastructure
variable "aws_region" {
  default = "us-east-1"
}

variable "ec2_instance_type" {
  description = "EC2 instance type"
  default     = "t3.medium"
}

variable "db_password" {
  description = "The password for the RDS instance"
  type        = string
}

variable "instance_count" {
  description = "Number of EC2 instances to launch"
  default     = 2
}

```

```
hcl
Copy code
# vpc.tf

# Create a Virtual Private Cloud (VPC) for network isolation
resource "aws_vpc" "movie_recommendation_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "MovieRecommendationVPC"
  }
}

# Internet Gateway for public access to the VPC
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.movie_recommendation_vpc.id
}

# Public subnet for EC2 instances and ALB
resource "aws_subnet" "public_subnet" {
  vpc_id            = aws_vpc.movie_recommendation_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = {
    Name = "MovieRecommendationPublicSubnet"
  }
}

# Private subnet for secure resources (RDS)
resource "aws_subnet" "private_subnet" {
  vpc_id            = aws_vpc.movie_recommendation_vpc.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1a"
  tags = {
    Name = "MovieRecommendationPrivateSubnet"
  }
}

# Route table for the public subnet
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.movie_recommendation_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
}

# Associate route table with the public subnet
resource "aws_route_table_association" "public_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

# Associate route table with the private subnet
resource "aws_route_table_association" "private_association" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

```

```
hcl
Copy code
# security-groups.tf

# Security group for EC2 instances allowing HTTP and SSH access
resource "aws_security_group" "ec2_sg" {
  vpc_id = aws_vpc.movie_recommendation_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "EC2SecurityGroup"
  }
}

# Security group for RDS allowing access from the VPC
resource "aws_security_group" "rds_sg" {
  vpc_id = aws_vpc.movie_recommendation_vpc.id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "RDSSecurityGroup"
  }
}

```

```
hcl
Copy code
# ec2.tf

# EC2 instances to run the Django web application
resource "aws_instance" "web_server" {
  count         = var.instance_count
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI
  instance_type = var.ec2_instance_type
  subnet_id     = aws_subnet.private_subnet.id
  key_name      = "my-key"

  # Security group for EC2 instances
  security_groups = [aws_security_group.ec2_sg.name]

  # User data to install and start Apache
  user_data = <<-EOF
              #!/bin/bash
              sudo yum update -y
              sudo yum install -y httpd
              sudo systemctl start httpd
              sudo systemctl enable httpd
              EOF

  tags = {
    Name = "DjangoWebServer"
  }
}

```

```
hcl
Copy code
# rds.tf

# PostgreSQL RDS instance for database storage
resource "aws_db_instance" "movie_db" {
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  allocated_storage = 20
  name              = "moviesdb"
  username          = "admin"
  password          = var.db_password
  publicly_accessible = false
  vpc_security_group_ids = [aws_security_group.rds_sg.id]

  # Subnet group for RDS
  db_subnet_group_name = aws_db_subnet_group.db_subnet_group.name
  multi_az             = false

  tags = {
    Name = "MovieRecommendationDB"
  }
}

# Subnet group for the RDS instance
resource "aws_db_subnet_group" "db_subnet_group" {
  name       = "movie-db-subnet-group"
  subnet_ids = [aws_subnet.private_subnet.id]

  tags = {
    Name = "MovieRecommendationDBSubnetGroup"
  }
}

```

```
hcl
Copy code
# alb.tf

# Application Load Balancer for distributing traffic
resource "aws_lb" "app_lb" {
  name               = "movie-recommendation-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.ec2_sg.id]
  subnets            = [aws_subnet.public_subnet.id]

  tags = {
    Name = "MovieRecommendationALB"
  }
}

# Target group for the ALB
resource "aws_lb_target_group" "tg" {
  name     = "movie-recommendation-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.movie_recommendation_vpc.id

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }

  tags = {
    Name = "MovieRecommendationTG"
  }
}

# Listener for the ALB
resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}

```

```
hcl
Copy code
# s3.tf

# S3 bucket for storing static files
resource "aws_s3_bucket" "static_files" {
  bucket = "movie-recommendation-static-files"

  acl    = "private"

  tags = {
    Name = "MovieRecommendationStaticFiles"
  }
}

# S3 bucket for storing logs
resource "aws_s3_bucket" "logs_bucket" {
  bucket = "movie-recommendation-logs"

  acl    = "private"

  tags = {
    Name = "MovieRecommendationLogs"
  }
}

```

```
hcl
Copy code
# outputs.tf

# Output the DNS of the Application Load Balancer
output "alb_dns_name" {
  description = "The DNS name of the ALB"
  value       = aws_lb.app_lb.dns_name
}

# Output the endpoint of the RDS database
output "db_endpoint" {
  description = "RDS database endpoint"
  value       = aws_db_instance.movie_db.endpoint
}

# Output the public IP addresses of EC2 instances
output "ec2_instance_ips" {
  description = "IP addresses of EC2 instances"
  value       = aws_instance.web_server.*.public_ip
}

```

---

## Instructions for Running Terraform

1. **Initialize Terraform**:
    
    ```bash
    bash
    Copy code
    terraform init
    
    ```
    
2. **Plan the infrastructure**:
    
    ```bash
    bash
    Copy code
    terraform plan
    
    ```
    
3. **Apply the configuration**:
    
    ```bash
    bash
    Copy code
    terraform apply
    
    ```
    
4. **View Outputs**:
After running `terraform apply`, use the following command to view the output information:
    
    ```bash
    bash
    Copy code
    terraform output
    
    ```
    

---

## Summary

This Terraform configuration provides a complete, scalable, and secure infrastructure for the Django-Based Movie Recommendation System on AWS. The infrastructure includes key AWS services like EC2 for web servers, RDS for the PostgreSQL database, an ALB for traffic distribution, and S3 for static file storage. It is designed with modular components to allow flexibility, scalability, and easy management through Terraform.

Adjustments, such as adding more EC2 instances or enabling Multi-AZ for RDS, can be made by modifying the respective Terraform files. This setup is ideal for deploying production-ready cloud infrastructure that can scale with demand while ensuring high availability and security.
