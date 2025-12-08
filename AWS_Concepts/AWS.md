# AWS Interview Questions and Answers

## Table of Contents

1. [AWS Fundamentals](#aws-fundamentals)
2. [IAM and Security](#iam-and-security)
3. [Networking (VPC)](#networking-vpc)
4. [Compute Services](#compute-services)
5. [Container Services](#container-services)
6. [Storage Services](#storage-services)
7. [Database Services](#database-services)

---

## AWS Fundamentals

### Q1: Explain AWS global infrastructure.

**Answer:**

| Component | Description |
|-----------|-------------|
| Region | Geographic area with multiple AZs |
| Availability Zone (AZ) | Isolated data center within a region |
| Edge Location | CDN endpoint for CloudFront |
| Local Zone | Extension of region closer to users |
| Wavelength Zone | 5G edge computing |

**Characteristics:**
- 30+ Regions worldwide
- Each region has 2-6 AZs
- AZs are physically separated
- Low latency between AZs in same region
- 400+ Edge Locations globally

---

### Q2: What is the AWS shared responsibility model?

**Answer:**

| AWS Responsibility | Customer Responsibility |
|--------------------|-------------------------|
| Physical security | Data encryption |
| Network infrastructure | IAM management |
| Hypervisor | Security groups, NACLs |
| Managed services | Application security |
| Hardware maintenance | OS patching (EC2) |
| Compliance certifications | Customer data |

**Summary:**
- AWS: Security OF the cloud
- Customer: Security IN the cloud

---

## IAM and Security

### Q3: Explain AWS IAM components.

**Answer:**

| Component | Description | Use Case |
|-----------|-------------|----------|
| User | Identity for person/app | Individual developers |
| Group | Collection of users | Team permissions |
| Role | Assumable identity | EC2, Lambda, cross-account |
| Policy | JSON permissions | Attached to above |

**IAM Policy structure:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

---

### Q4: What is the difference between IAM roles and users?

**Answer:**

| IAM User | IAM Role |
|----------|----------|
| Long-term credentials | Temporary credentials |
| For people | For services/applications |
| Has password/access keys | No permanent credentials |
| Cannot be assumed | Can be assumed |
| Belongs to one account | Cross-account possible |

**Role trust policy:**

```json
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
```

**Best practices:**
- Use roles for applications (not access keys)
- Enable MFA for users
- Follow least privilege principle
- Rotate credentials regularly
- Use IAM Access Analyzer

---

### Q5: Explain AWS security services.

**Answer:**

| Service | Purpose |
|---------|---------|
| IAM | Identity and access management |
| KMS | Key management for encryption |
| Secrets Manager | Secrets storage and rotation |
| WAF | Web application firewall |
| Shield | DDoS protection |
| GuardDuty | Threat detection |
| Security Hub | Security posture management |
| Inspector | Vulnerability assessment |
| CloudTrail | API audit logging |
| Config | Resource compliance |

---

## Networking (VPC)

### Q6: Explain VPC components.

**Answer:**

```
+----------------------------------------------------------+
|                          VPC                              |
|                     (10.0.0.0/16)                         |
|                                                           |
|   +------------------+    +------------------+            |
|   |  Public Subnet   |    |  Private Subnet  |            |
|   |  (10.0.1.0/24)   |    |  (10.0.2.0/24)   |            |
|   |                  |    |                  |            |
|   |  +------------+  |    |  +------------+  |            |
|   |  |    EC2     |  |    |  |    EC2     |  |            |
|   |  +------------+  |    |  +------------+  |            |
|   |                  |    |                  |            |
|   +--------+---------+    +--------+---------+            |
|            |                       |                      |
|            |                       |                      |
|   +--------v---------+    +--------v---------+            |
|   | Internet Gateway |    |   NAT Gateway    |            |
|   +--------+---------+    +------------------+            |
|            |                                              |
+------------|----------------------------------------------+
             |
         Internet
```

**Components:**

| Component | Description |
|-----------|-------------|
| VPC | Virtual network (CIDR block) |
| Subnet | Network segment within AZ |
| Internet Gateway | Public internet access |
| NAT Gateway | Private subnet outbound |
| Route Table | Traffic routing rules |
| Security Group | Instance-level firewall (stateful) |
| NACL | Subnet-level firewall (stateless) |
| VPC Peering | Connect VPCs |
| Transit Gateway | Hub for multiple VPCs |
| VPC Endpoint | Private AWS service access |

---

### Q7: Explain Security Groups vs NACLs.

**Answer:**

| Security Group | NACL |
|----------------|------|
| Instance level | Subnet level |
| Stateful | Stateless |
| Allow rules only | Allow and deny rules |
| All rules evaluated | Rules evaluated in order |
| Applied to ENI | Applied to subnet |

**Security Group example:**

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
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

**NACL example:**

```hcl
resource "aws_network_acl" "main" {
  vpc_id = aws_vpc.main.id

  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  ingress {
    rule_no    = 200
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535  # Ephemeral ports for responses
  }

  egress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
}
```

---

### Q8: What are VPC Endpoints?

**Answer:**

VPC Endpoints allow private access to AWS services without internet.

| Type | Description | Use Case |
|------|-------------|----------|
| Gateway | S3 and DynamoDB | Free, route table entry |
| Interface | Other services | ENI with private IP |

**Gateway Endpoint:**

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.eu-west-1.s3"
  
  route_table_ids = [
    aws_route_table.private.id
  ]
}
```

**Interface Endpoint:**

```hcl
resource "aws_vpc_endpoint" "ecr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.eu-west-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  
  subnet_ids          = [aws_subnet.private.id]
  security_group_ids  = [aws_security_group.endpoint.id]
  private_dns_enabled = true
}
```

---

## Compute Services

### Q9: Explain EC2 instance types and purchasing options.

**Answer:**

**Instance families:**

| Family | Use Case |
|--------|----------|
| T | Burstable, general purpose |
| M | Balanced compute/memory |
| C | Compute optimized |
| R | Memory optimized |
| I | Storage optimized |
| G/P | GPU instances |

**Purchasing options:**

| Option | Discount | Commitment |
|--------|----------|------------|
| On-Demand | 0% | None |
| Reserved (1yr) | ~40% | 1 year |
| Reserved (3yr) | ~60% | 3 years |
| Spot | Up to 90% | Can be interrupted |
| Savings Plans | Up to 72% | $/hour commitment |

---

### Q10: What is AWS Lambda?

**Answer:**

Serverless compute - run code without managing servers.

**Characteristics:**
- Pay per invocation (100ms increments)
- Auto-scaling (0 to thousands)
- Max 15 minutes execution
- Max 10GB memory
- Ephemeral storage (512MB-10GB)

**Use cases:**
- API backends (with API Gateway)
- Event processing (S3, SQS, DynamoDB)
- Scheduled tasks (EventBridge)
- Data transformation

**Terraform example:**

```hcl
resource "aws_lambda_function" "api" {
  filename         = "lambda.zip"
  function_name    = "my-api"
  role             = aws_iam_role.lambda.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  timeout          = 30
  memory_size      = 256

  environment {
    variables = {
      DB_HOST = aws_rds_cluster.main.endpoint
    }
  }

  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }
}
```

---

## Container Services

### Q11: Explain AWS EKS architecture.

**Answer:**

```
+----------------------------------------------------------+
|                        AWS Cloud                          |
|                                                           |
|  +---------------------------------------------------+   |
|  |              EKS Control Plane                     |   |
|  |         (Managed by AWS - HA across 3 AZs)        |   |
|  |  +-------------+ +------+ +------------+          |   |
|  |  | API Server  | | etcd | | Controller |          |   |
|  |  +-------------+ +------+ +------------+          |   |
|  +-------------------------+-------------------------+   |
|                            |                             |
|  +-------------------------v-------------------------+   |
|  |                    Your VPC                        |   |
|  |  +-----------+ +-----------+ +-----------+        |   |
|  |  |   AZ-1    | |   AZ-2    | |   AZ-3    |        |   |
|  |  | +-------+ | | +-------+ | | +-------+ |        |   |
|  |  | |Worker | | | |Worker | | | |Worker | |        |   |
|  |  | | Node  | | | | Node  | | | | Node  | |        |   |
|  |  | +-------+ | | +-------+ | | +-------+ |        |   |
|  |  +-----------+ +-----------+ +-----------+        |   |
|  +---------------------------------------------------+   |
+----------------------------------------------------------+
```

**EKS vs Self-managed:**

| EKS | Self-Managed |
|-----|--------------|
| AWS manages control plane | You manage everything |
| HA built-in | Must configure HA |
| Automatic upgrades | Manual upgrades |
| AWS integration | More integration work |
| $0.10/hr per cluster | No control plane cost |

---

### Q12: Explain AWS ECR.

**Answer:**

Elastic Container Registry - managed Docker registry.

```bash
# Authenticate
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS \
  --password-stdin 123456789.dkr.ecr.eu-west-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp

# Tag and push
docker tag myapp:latest \
  123456789.dkr.ecr.eu-west-1.amazonaws.com/myapp:latest
docker push \
  123456789.dkr.ecr.eu-west-1.amazonaws.com/myapp:latest
```

**Features:**
- Image scanning for vulnerabilities
- Lifecycle policies
- Cross-region replication
- IAM-based access
- Encryption at rest

**Lifecycle policy:**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

## Storage Services

### Q13: Explain S3 storage classes.

**Answer:**

| Class | Use Case | Retrieval |
|-------|----------|-----------|
| Standard | Frequently accessed | Immediate |
| Intelligent-Tiering | Unknown patterns | Automatic |
| Standard-IA | Infrequent access | Immediate |
| One Zone-IA | Non-critical | Immediate |
| Glacier Instant | Archive, instant | Immediate |
| Glacier Flexible | Archive | Minutes-hours |
| Glacier Deep Archive | Long-term | 12-48 hours |

**Lifecycle policy:**

```json
{
  "Rules": [
    {
      "ID": "ArchiveOldLogs",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 }
    }
  ]
}
```

---

### Q14: Explain S3 security features.

**Answer:**

| Feature | Description |
|---------|-------------|
| Bucket Policy | Resource-based policy |
| ACLs | Legacy access control |
| Block Public Access | Account-level protection |
| Encryption | SSE-S3, SSE-KMS, SSE-C |
| Object Lock | WORM compliance |
| Access Points | Simplified access |
| VPC Endpoints | Private access |

**Bucket policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowVPCAccess",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:sourceVpc": "vpc-12345"
        }
      }
    }
  ]
}
```

---

### Q15: Explain EBS volume types.

**Answer:**

| Type | Use Case | IOPS | Throughput |
|------|----------|------|------------|
| gp3 | General SSD | 16,000 | 1,000 MB/s |
| gp2 | General SSD | 16,000 | 250 MB/s |
| io2 | High perf | 256,000 | 4,000 MB/s |
| io1 | High perf | 64,000 | 1,000 MB/s |
| st1 | Throughput HDD | 500 | 500 MB/s |
| sc1 | Cold HDD | 250 | 250 MB/s |

**Terraform example:**

```hcl
resource "aws_ebs_volume" "data" {
  availability_zone = "eu-west-1a"
  size              = 100
  type              = "gp3"
  iops              = 3000
  throughput        = 125
  encrypted         = true
  kms_key_id        = aws_kms_key.ebs.arn

  tags = {
    Name = "data-volume"
  }
}
```

---

## Database Services

### Q16: Explain RDS vs Aurora.

**Answer:**

| Feature | RDS | Aurora |
|---------|-----|--------|
| Engines | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server | MySQL, PostgreSQL compatible |
| Storage | EBS-based | Distributed, auto-scaling |
| Read replicas | Up to 5 | Up to 15 |
| Failover | 60-120 seconds | ~30 seconds |
| Storage scaling | Manual | Automatic (10GB-128TB) |
| Cost | Lower | Higher |

**Aurora advantages:**
- 5x MySQL, 3x PostgreSQL performance
- Auto-scaling storage
- Continuous backup to S3
- Global database for multi-region
- Serverless option

---

### Q17: Explain DynamoDB.

**Answer:**

Fully managed NoSQL database.

**Characteristics:**
- Single-digit millisecond latency
- Automatic scaling
- Global tables (multi-region)
- Point-in-time recovery
- Encryption at rest

**Capacity modes:**

| Mode | Description |
|------|-------------|
| On-Demand | Pay per request |
| Provisioned | Set RCU/WCU |

**Terraform example:**

```hcl
resource "aws_dynamodb_table" "users" {
  name           = "users"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "user_id"
  range_key      = "timestamp"

  attribute {
    name = "user_id"
    type = "S"
  }

  attribute {
    name = "timestamp"
    type = "N"
  }

  global_secondary_index {
    name            = "email-index"
    hash_key        = "email"
    projection_type = "ALL"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Environment = "production"
  }
}
```

---

## Summary

| Category | Key Services |
|----------|--------------|
| Compute | EC2, Lambda, ECS, EKS |
| Storage | S3, EBS, EFS |
| Database | RDS, Aurora, DynamoDB |
| Networking | VPC, Route 53, CloudFront |
| Security | IAM, KMS, WAF, GuardDuty |
| Monitoring | CloudWatch, CloudTrail |
| Integration | SQS, SNS, EventBridge |

**AWS CLI essentials:**

```bash
# Configure credentials
aws configure

# EC2 operations
aws ec2 describe-instances
aws ec2 start-instances --instance-ids i-xxx
aws ec2 stop-instances --instance-ids i-xxx

# S3 operations
aws s3 ls
aws s3 cp file.txt s3://bucket/
aws s3 sync ./local s3://bucket/

# EKS operations
aws eks update-kubeconfig --name cluster-name
```