# Terraform and Infrastructure as Code Interview Questions

## Table of Contents

1. [Terraform Fundamentals](#terraform-fundamentals)
2. [State Management](#state-management)
3. [Variables and Data Sources](#variables-and-data-sources)
4. [Modules](#modules)
5. [Advanced Features](#advanced-features)
6. [Best Practices](#best-practices)
7. [Ansible Basics](#ansible-basics)

---

## Terraform Fundamentals

### Q1: What is Terraform and how does it work?

**Answer:**

Terraform is a declarative Infrastructure as Code (IaC) tool that manages infrastructure through configuration files.

**Workflow:**

```
Write --> Plan --> Apply --> Destroy
  |         |        |         |
  |         |        |         +-- terraform destroy
  |         |        +-- terraform apply
  |         +-- terraform plan
  +-- .tf files
```

**Key concepts:**

| Concept | Description |
|---------|-------------|
| Provider | Plugin to interact with APIs (AWS, Azure, GCP) |
| Resource | Infrastructure component to create |
| Data Source | Read existing infrastructure |
| State | Mapping between config and real resources |
| Module | Reusable configuration package |

**Basic structure:**

```hcl
# Provider configuration
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}

# Resource
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}

# Output
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

---

### Q2: Explain Terraform commands.

**Answer:**

| Command | Description |
|---------|-------------|
| `terraform init` | Initialize working directory |
| `terraform plan` | Preview changes |
| `terraform apply` | Apply changes |
| `terraform destroy` | Destroy infrastructure |
| `terraform fmt` | Format configuration |
| `terraform validate` | Validate configuration |
| `terraform state` | State management |
| `terraform import` | Import existing resources |
| `terraform output` | Show outputs |
| `terraform refresh` | Update state |

```bash
# Initialize with backend
terraform init -backend-config="bucket=my-state"

# Plan with variable file
terraform plan -var-file="prod.tfvars" -out=plan.tfplan

# Apply specific plan
terraform apply plan.tfplan

# Destroy specific resource
terraform destroy -target=aws_instance.web

# Format all files
terraform fmt -recursive

# Validate configuration
terraform validate
```

---

### Q3: What is the difference between Terraform and Ansible?

**Answer:**

| Terraform | Ansible |
|-----------|---------|
| Declarative | Procedural/Declarative |
| Infrastructure provisioning | Configuration management |
| State-based | Agentless, SSH-based |
| Immutable approach | Mutable by default |
| Provider plugins | Modules and roles |
| HCL language | YAML playbooks |

**When to use:**
- Terraform: Create infrastructure (VMs, networks, databases)
- Ansible: Configure infrastructure (install packages, manage files)
- Often used together: Terraform provisions, Ansible configures

---

## State Management

### Q4: Explain Terraform state file.

**Answer:**

State file (`terraform.tfstate`) maps configuration to real resources.

**Contents:**
- Resource IDs and attributes
- Metadata
- Dependencies
- Provider configuration

**Why important:**
- Tracks what Terraform manages
- Enables accurate plan/diff
- Stores sensitive data (handle carefully)

```bash
# State commands
terraform state list                    # List resources
terraform state show aws_instance.web   # Show details
terraform state mv old.name new.name    # Rename
terraform state rm aws_instance.web     # Remove from state
terraform state pull                    # Download remote state
```

**Never:**
- Edit state file manually
- Commit state to Git
- Run concurrent operations without locking

---

### Q5: How do you manage Terraform state in a team?

**Answer:**

Use remote backend with locking.

**S3 + DynamoDB backend:**

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**DynamoDB table for locking:**

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Best practices:**
- Enable versioning on S3 bucket
- Use encryption at rest
- Restrict access via IAM
- Use workspaces or separate state files per environment

---

### Q6: Explain Terraform workspaces.

**Answer:**

Workspaces provide isolated state files within same configuration.

```bash
# Create workspace
terraform workspace new staging
terraform workspace new production

# Switch workspace
terraform workspace select staging

# List workspaces
terraform workspace list

# Show current workspace
terraform workspace show
```

**Using in configuration:**

```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"

  tags = {
    Environment = terraform.workspace
  }
}

locals {
  env_config = {
    staging = {
      instance_type = "t3.micro"
      replicas      = 1
    }
    production = {
      instance_type = "t3.large"
      replicas      = 3
    }
  }
  config = local.env_config[terraform.workspace]
}
```

---

## Variables and Data Sources

### Q7: Explain Terraform variable types.

**Answer:**

```hcl
# String
variable "region" {
  type        = string
  default     = "eu-west-1"
  description = "AWS region"
}

# Number
variable "instance_count" {
  type    = number
  default = 2
}

# Boolean
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List
variable "availability_zones" {
  type    = list(string)
  default = ["eu-west-1a", "eu-west-1b"]
}

# Map
variable "instance_tags" {
  type = map(string)
  default = {
    Environment = "production"
    Team        = "devops"
  }
}

# Object
variable "instance_config" {
  type = object({
    instance_type = string
    volume_size   = number
    tags          = map(string)
  })
  default = {
    instance_type = "t3.micro"
    volume_size   = 20
    tags          = {}
  }
}

# Validation
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Sensitive
variable "db_password" {
  type      = string
  sensitive = true
}
```

**Variable precedence (lowest to highest):**
1. Default value
2. Environment variables (TF_VAR_name)
3. terraform.tfvars
4. *.auto.tfvars
5. -var-file flag
6. -var flag

---

### Q8: What is the difference between data source and resource?

**Answer:**

| Resource | Data Source |
|----------|-------------|
| Creates/manages infrastructure | Reads existing infrastructure |
| `resource` block | `data` block |
| Terraform owns lifecycle | Read-only |
| Stored in state | Queried each plan/apply |

```hcl
# Resource - Terraform creates
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnet.existing.id
}

# Data source - Query existing
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_subnet" "existing" {
  filter {
    name   = "tag:Name"
    values = ["existing-subnet"]
  }
}

data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

---

### Q9: Explain Terraform locals.

**Answer:**

Named expressions for reusable values within a module.

```hcl
locals {
  # Simple values
  environment = "production"
  project     = "myapp"

  # Computed values
  name_prefix = "${local.project}-${local.environment}"

  # Complex expressions
  common_tags = {
    Environment = local.environment
    Project     = local.project
    ManagedBy   = "Terraform"
  }

  # Conditional logic
  instance_type = local.environment == "production" ? "t3.large" : "t3.micro"

  # Merged maps
  all_tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-server"
  })
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type
  tags          = local.all_tags
}
```

---

## Modules

### Q10: What are Terraform modules?

**Answer:**

Modules are reusable, self-contained packages of Terraform configuration.

**Module structure:**

```
modules/
+-- vpc/
    +-- main.tf
    +-- variables.tf
    +-- outputs.tf
    +-- README.md
```

**Module definition (modules/vpc/main.tf):**

```hcl
variable "cidr_block" {
  type = string
}

variable "environment" {
  type = string
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = aws_subnet.public[*].id
}
```

**Using the module:**

```hcl
module "vpc" {
  source = "./modules/vpc"

  cidr_block  = "10.0.0.0/16"
  environment = "production"
}

# Access module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.subnet_ids[0]
}
```

---

### Q11: How do you version Terraform modules?

**Answer:**

```hcl
# Local module
module "vpc" {
  source = "./modules/vpc"
}

# Git with tag
module "vpc" {
  source = "git::https://github.com/org/modules.git//vpc?ref=v1.2.0"
}

# Git with branch
module "vpc" {
  source = "git::https://github.com/org/modules.git//vpc?ref=develop"
}

# Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# S3 bucket
module "vpc" {
  source = "s3::https://s3-eu-west-1.amazonaws.com/bucket/vpc.zip"
}
```

**Version constraints:**

| Constraint | Description |
|------------|-------------|
| `= 1.0.0` | Exact version |
| `>= 1.0` | Minimum version |
| `~> 1.0` | Allow 1.x but not 2.0 |
| `>= 1.0, < 2.0` | Range |

---

## Advanced Features

### Q12: Explain `count` vs `for_each`.

**Answer:**

| count | for_each |
|-------|----------|
| Index-based ([0], [1]) | Key-based |
| Removing item shifts indexes | Only affects that key |
| Simple duplication | More control |

**Count:**

```hcl
variable "instance_count" {
  default = 3
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"
  }
}
```

**For_each with set:**

```hcl
variable "instance_names" {
  default = ["web", "api", "worker"]
}

resource "aws_instance" "servers" {
  for_each      = toset(var.instance_names)
  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = each.value
  }
}
```

**For_each with map:**

```hcl
variable "instances" {
  default = {
    web = {
      type = "t3.small"
      az   = "eu-west-1a"
    }
    api = {
      type = "t3.micro"
      az   = "eu-west-1b"
    }
  }
}

resource "aws_instance" "servers" {
  for_each          = var.instances
  ami               = "ami-12345"
  instance_type     = each.value.type
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}
```

---

### Q13: What is Terraform `dynamic` block?

**Answer:**

Generates repeated nested blocks dynamically.

```hcl
variable "ingress_rules" {
  default = [
    { port = 80, cidr = "0.0.0.0/0", description = "HTTP" },
    { port = 443, cidr = "0.0.0.0/0", description = "HTTPS" },
    { port = 22, cidr = "10.0.0.0/8", description = "SSH" }
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

### Q14: Explain `depends_on` and implicit dependencies.

**Answer:**

**Implicit dependencies (automatic):**

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # Implicit dependency
  cidr_block = "10.0.1.0/24"
}
# Terraform knows subnet depends on VPC
```

**Explicit depends_on:**

```hcl
resource "aws_iam_role_policy" "s3_access" {
  name   = "s3-access"
  role   = aws_iam_role.lambda.id
  policy = jsonencode({...})
}

resource "aws_lambda_function" "processor" {
  function_name = "processor"
  role          = aws_iam_role.lambda.arn

  # Lambda needs policy attached before it works
  depends_on = [aws_iam_role_policy.s3_access]
}
```

---

### Q15: How do you import existing resources?

**Answer:**

```bash
# Import command
terraform import aws_instance.web i-1234567890abcdef0

# Import into module
terraform import module.vpc.aws_vpc.main vpc-123456
```

**Terraform 1.5+ import blocks:**

```hcl
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}

resource "aws_instance" "web" {
  # Configuration must match imported resource
  ami           = "ami-12345"
  instance_type = "t3.micro"
}
```

**Limitations:**
- Only imports state, not configuration
- One resource at a time
- Must write matching HCL manually

---

## Best Practices

### Q16: What are Terraform best practices?

**Answer:**

1. **Use remote state with locking**

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

2. **Use modules for reusability**

3. **Version pin providers and modules**

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

4. **Use variables and locals**

5. **Format and validate**

```bash
terraform fmt -recursive
terraform validate
```

6. **Use workspaces or directories for environments**

7. **Implement CI/CD for Terraform**

8. **Never commit secrets**

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

9. **Use consistent naming conventions**

10. **Document with comments and README**

---

## Ansible Basics

### Q17: Explain Ansible architecture.

**Answer:**

```
Control Node (Ansible installed)
       |
       +-- Inventory (hosts)
       +-- Playbooks (YAML tasks)
       +-- Roles (reusable components)
       +-- Modules (units of work)
       |
       v (SSH/WinRM - agentless)
Managed Nodes (no agent needed)
```

**Inventory:**

```ini
[webservers]
web1.example.com ansible_host=10.0.1.10
web2.example.com ansible_host=10.0.1.11

[dbservers]
db1.example.com ansible_host=10.0.2.10

[production:children]
webservers
dbservers

[webservers:vars]
ansible_user=ubuntu
```

---

### Q18: Explain Ansible playbooks.

**Answer:**

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    app_version: "2.0.0"

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

---

### Q19: Explain Ansible roles.

**Answer:**

**Role structure:**

```
roles/
+-- nginx/
    +-- tasks/
    |   +-- main.yml
    +-- handlers/
    |   +-- main.yml
    +-- templates/
    |   +-- nginx.conf.j2
    +-- files/
    +-- vars/
    |   +-- main.yml
    +-- defaults/
    |   +-- main.yml
    +-- meta/
        +-- main.yml
```

**Using roles:**

```yaml
- hosts: webservers
  roles:
    - nginx
    - { role: app, tags: ['app'] }
    - role: monitoring
      vars:
        monitor_port: 9090
```

---

### Q20: Explain Ansible Vault.

**Answer:**

Encrypts sensitive data in playbooks.

```bash
# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Encrypt existing file
ansible-vault encrypt vars/passwords.yml

# View encrypted file
ansible-vault view secrets.yml

# Run playbook with vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

**Encrypt single string:**

```bash
ansible-vault encrypt_string 'mysecretpassword' --name 'db_password'
```

---

## Summary

| Topic | Key Concepts |
|-------|--------------|
| Terraform | Declarative IaC, providers, resources |
| State | Remote backend, locking, workspaces |
| Variables | Types, precedence, validation |
| Modules | Reusability, versioning |
| Advanced | count, for_each, dynamic, depends_on |
| Ansible | Agentless, playbooks, roles, vault |

**Essential Terraform commands:**

```bash
terraform init      # Initialize
terraform plan      # Preview
terraform apply     # Deploy
terraform destroy   # Remove
terraform fmt       # Format
terraform validate  # Validate
terraform state     # State management
terraform import    # Import existing
```