# Investment Product Sales Tool - Terraform Infrastructure

## Overview

This document outlines the Terraform Cloud infrastructure configuration for the Investment Product Sales Tool, replacing AWS CloudFormation with a more robust Infrastructure as Code (IaC) solution.

## Terraform Cloud Organization Structure

```
investment-tool/
├── workspaces/
│   ├── dev/          # Development environment
│   ├── staging/      # Staging environment
│   └── prod/         # Production environment
├── modules/
│   ├── networking/   # VPC, subnets, security groups
│   ├── database/     # Aurora cluster configuration
│   ├── compute/      # Lambda functions
│   ├── security/     # WAF, IAM, certificates
│   └── monitoring/   # CloudWatch, alarms
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

## Root Module Configuration

### main.tf
```hcl
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  cloud {
    organization = "investment-tool"
    workspaces {
      name = "investment-tool-${var.environment}"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = "investment-tool"
      ManagedBy   = "terraform"
    }
  }
}

# Data sources
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Local values
locals {
  name_prefix = "investment-tool-${var.environment}"
  common_tags = {
    Environment = var.environment
    Project     = "investment-tool"
    ManagedBy   = "terraform"
  }
}

# Networking module
module "networking" {
  source = "./modules/networking"
  
  environment    = var.environment
  vpc_cidr      = var.vpc_cidr
  azs           = var.availability_zones
  name_prefix   = local.name_prefix
  common_tags   = local.common_tags
}

# Database module
module "database" {
  source = "./modules/database"
  
  environment           = var.environment
  vpc_id              = module.networking.vpc_id
  private_subnet_ids  = module.networking.private_subnet_ids
  security_group_ids  = [module.networking.database_security_group_id]
  name_prefix         = local.name_prefix
  common_tags         = local.common_tags
  
  # Database configuration
  db_name             = var.database_name
  db_username         = var.database_username
  db_password         = var.database_password
  db_instance_class   = var.database_instance_class
  db_cluster_size     = var.database_cluster_size
  backup_retention    = var.database_backup_retention
}

# Security module
module "security" {
  source = "./modules/security"
  
  environment     = var.environment
  vpc_id         = module.networking.vpc_id
  name_prefix    = local.name_prefix
  common_tags    = local.common_tags
  
  # WAF configuration
  waf_rate_limit = var.waf_rate_limit
  
  # Certificate configuration
  domain_name    = var.domain_name
  certificate_arn = var.certificate_arn
}

# Compute module (Lambda functions)
module "compute" {
  source = "./modules/compute"
  
  environment           = var.environment
  vpc_id              = module.networking.vpc_id
  private_subnet_ids  = module.networking.private_subnet_ids
  security_group_ids  = [module.networking.lambda_security_group_id]
  name_prefix         = local.name_prefix
  common_tags         = local.common_tags
  
  # Lambda configuration
  lambda_memory_size  = var.lambda_memory_size
  lambda_timeout      = var.lambda_timeout
  lambda_runtime      = "dotnet8"
  
  # Database connection
  database_endpoint   = module.database.cluster_endpoint
  database_name       = module.database.database_name
  database_username   = module.database.database_username
  database_password   = module.database.database_password
}

# Monitoring module
module "monitoring" {
  source = "./modules/monitoring"
  
  environment     = var.environment
  name_prefix    = local.name_prefix
  common_tags    = local.common_tags
  
  # CloudWatch configuration
  log_retention_days = var.log_retention_days
  
  # SNS topics for alerts
  alert_email     = var.alert_email
}
```

### variables.tf
```hcl
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "domain_name" {
  description = "Domain name for the application"
  type        = string
}

variable "certificate_arn" {
  description = "SSL certificate ARN"
  type        = string
}

variable "database_name" {
  description = "Database name"
  type        = string
  default     = "investment_tool"
}

variable "database_username" {
  description = "Database master username"
  type        = string
  default     = "admin"
}

variable "database_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}

variable "database_instance_class" {
  description = "Database instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "database_cluster_size" {
  description = "Number of database instances"
  type        = number
  default     = 1
}

variable "database_backup_retention" {
  description = "Database backup retention in days"
  type        = number
  default     = 7
}

variable "lambda_memory_size" {
  description = "Lambda function memory size"
  type        = number
  default     = 512
}

variable "lambda_timeout" {
  description = "Lambda function timeout in seconds"
  type        = number
  default     = 30
}

variable "waf_rate_limit" {
  description = "WAF rate limit per IP"
  type        = number
  default     = 1000
}

variable "log_retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 30
}

variable "alert_email" {
  description = "Email for alerts"
  type        = string
}
```

### outputs.tf
```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = module.networking.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = module.networking.private_subnet_ids
}

output "database_endpoint" {
  description = "Database cluster endpoint"
  value       = module.database.cluster_endpoint
  sensitive   = true
}

output "database_name" {
  description = "Database name"
  value       = module.database.database_name
}

output "api_gateway_url" {
  description = "API Gateway URL"
  value       = module.compute.api_gateway_url
}

output "lambda_function_arn" {
  description = "Lambda function ARN"
  value       = module.compute.lambda_function_arn
}

output "cloudwatch_log_group" {
  description = "CloudWatch log group name"
  value       = module.monitoring.log_group_name
}
```

## Module Configurations

### Networking Module
```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-vpc"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.azs[count.index]
  map_public_ip_on_launch = false
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-private-subnet-${count.index + 1}"
  })
}

resource "aws_security_group" "lambda" {
  name_prefix = "${var.name_prefix}-lambda-sg"
  vpc_id      = aws_vpc.main.id
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-lambda-sg"
  })
}

resource "aws_security_group" "database" {
  name_prefix = "${var.name_prefix}-database-sg"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.lambda.id]
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-database-sg"
  })
}
```

### Database Module
```hcl
# modules/database/main.tf
resource "aws_db_subnet_group" "main" {
  name_prefix = "${var.name_prefix}-db-subnet-group"
  subnet_ids  = var.private_subnet_ids
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-db-subnet-group"
  })
}

resource "aws_rds_cluster" "main" {
  cluster_identifier     = "${var.name_prefix}-cluster"
  engine                = "aurora-postgresql"
  engine_version        = "14.7"
  database_name         = var.db_name
  master_username       = var.db_username
  master_password       = var.db_password
  db_subnet_group_name  = aws_db_subnet_group.main.name
  vpc_security_group_ids = var.security_group_ids
  backup_retention_period = var.backup_retention
  storage_encrypted      = true
  deletion_protection   = var.environment == "prod"
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-cluster"
  })
}

resource "aws_rds_cluster_instance" "main" {
  count               = var.cluster_size
  identifier          = "${var.name_prefix}-instance-${count.index + 1}"
  cluster_identifier  = aws_rds_cluster.main.id
  instance_class      = var.instance_class
  engine              = aws_rds_cluster.main.engine
  engine_version      = aws_rds_cluster.main.engine_version
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-instance-${count.index + 1}"
  })
}
```

### Security Module
```hcl
# modules/security/main.tf
resource "aws_wafv2_web_acl" "main" {
  name        = "${var.name_prefix}-waf"
  description = "WAF for Investment Tool"
  scope       = "REGIONAL"
  
  default_action {
    allow {}
  }
  
  rule {
    name     = "RateLimitRule"
    priority = 1
    
    action {
      block {}
    }
    
    statement {
      rate_based_statement {
        limit              = var.waf_rate_limit
        aggregate_key_type = "IP"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name               = "RateLimitRule"
      sampled_requests_enabled  = true
    }
  }
  
  rule {
    name     = "SQLInjectionRule"
    priority = 2
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name               = "SQLInjectionRule"
      sampled_requests_enabled  = true
    }
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-waf"
  })
}

resource "aws_iam_role" "lambda_execution" {
  name = "${var.name_prefix}-lambda-execution-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-lambda-execution-role"
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}
```

## Environment-Specific Configurations

### Development Environment
```hcl
# environments/dev.tfvars
environment = "dev"
aws_region = "us-east-1"

vpc_cidr = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]

domain_name = "dev.investment-tool.com"
certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/dev-cert"

database_instance_class = "db.t3.micro"
database_cluster_size = 1
database_backup_retention = 7

lambda_memory_size = 512
lambda_timeout = 30

waf_rate_limit = 1000
log_retention_days = 7

alert_email = "dev-alerts@investment-tool.com"
```

### Staging Environment
```hcl
# environments/staging.tfvars
environment = "staging"
aws_region = "us-east-1"

vpc_cidr = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b"]

domain_name = "staging.investment-tool.com"
certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/staging-cert"

database_instance_class = "db.t3.small"
database_cluster_size = 1
database_backup_retention = 30

lambda_memory_size = 1024
lambda_timeout = 60

waf_rate_limit = 2000
log_retention_days = 30

alert_email = "staging-alerts@investment-tool.com"
```

### Production Environment
```hcl
# environments/prod.tfvars
environment = "prod"
aws_region = "us-east-1"

vpc_cidr = "10.0.0.0/16"
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

domain_name = "investment-tool.com"
certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/prod-cert"

database_instance_class = "db.r6g.large"
database_cluster_size = 2
database_backup_retention = 35

lambda_memory_size = 2048
lambda_timeout = 120

waf_rate_limit = 5000
log_retention_days = 90

alert_email = "prod-alerts@investment-tool.com"
```

## Terraform Cloud Workspace Configuration

### Workspace Variables
```hcl
# Terraform Cloud Variables (set in web UI)
# Environment Variables
AWS_ACCESS_KEY_ID     = "AKIA..." (sensitive)
AWS_SECRET_ACCESS_KEY = "..." (sensitive)

# Terraform Variables
environment            = "dev"
aws_region            = "us-east-1"
database_password     = "..." (sensitive)
alert_email           = "alerts@investment-tool.com"
```

### Workspace Settings
- **Execution Mode**: Remote
- **Terraform Version**: 1.5.0
- **Auto Apply**: Disabled (manual approval required)
- **VCS Integration**: GitHub
- **Branch**: main
- **Working Directory**: infrastructure

## Deployment Workflow

### Initial Setup
```bash
# 1. Initialize Terraform
cd infrastructure
terraform init

# 2. Login to Terraform Cloud
terraform login

# 3. Select workspace
terraform workspace select dev

# 4. Plan deployment
terraform plan -var-file="environments/dev.tfvars"

# 5. Apply changes
terraform apply -var-file="environments/dev.tfvars"
```

### CI/CD Integration
```yaml
# .github/workflows/terraform.yml
name: Terraform Infrastructure

on:
  push:
    branches: [ main ]
    paths: [ 'infrastructure/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'infrastructure/**' ]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      
      - name: Terraform Init
        run: |
          cd infrastructure
          terraform init
      
      - name: Terraform Plan
        run: |
          cd infrastructure
          terraform plan -var-file="environments/dev.tfvars"
        env:
          TF_VAR_database_password: ${{ secrets.DATABASE_PASSWORD }}
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: |
          cd infrastructure
          terraform apply -var-file="environments/dev.tfvars" -auto-approve
        env:
          TF_VAR_database_password: ${{ secrets.DATABASE_PASSWORD }}
```

## State Management

### Remote State Configuration
```hcl
# Backend configuration is handled by Terraform Cloud
terraform {
  cloud {
    organization = "investment-tool"
    workspaces {
      name = "investment-tool-${var.environment}"
    }
  }
}
```

### State Security
- State files stored securely in Terraform Cloud
- Encryption at rest and in transit
- Access controlled via Terraform Cloud permissions
- Audit logs for all state changes

## Cost Optimization

### Resource Tagging
```hcl
# Automatic cost allocation tags
default_tags {
  tags = {
    Environment = var.environment
    Project     = "investment-tool"
    ManagedBy   = "terraform"
    CostCenter  = "engineering"
  }
}
```

### Resource Sizing
```hcl
# Environment-specific sizing
locals {
  sizing = {
    dev = {
      lambda_memory = 512
      db_instance   = "db.t3.micro"
      db_cluster_size = 1
    }
    staging = {
      lambda_memory = 1024
      db_instance   = "db.t3.small"
      db_cluster_size = 1
    }
    prod = {
      lambda_memory = 2048
      db_instance   = "db.r6g.large"
      db_cluster_size = 2
    }
  }
}
```

## Security Best Practices

### IAM Least Privilege
```hcl
# Custom IAM policies for specific permissions
resource "aws_iam_policy" "lambda_secrets_access" {
  name = "${var.name_prefix}-lambda-secrets-access"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          "arn:aws:secretsmanager:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:secret:investment-tool/*"
        ]
      }
    ]
  })
}
```

### Network Security
```hcl
# Private subnets only
resource "aws_subnet" "private" {
  count                   = length(var.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = false
  
  tags = merge(var.common_tags, {
    Name = "${var.name_prefix}-private-subnet-${count.index + 1}"
  })
}
```

## Monitoring and Alerting

### CloudWatch Integration
```hcl
# CloudWatch alarms for infrastructure
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "${var.name_prefix}-lambda-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "Lambda function error rate"
  
  dimensions = {
    FunctionName = module.compute.lambda_function_name
  }
}
```

---

**Last Updated**: 2024-01-15
**Version**: 1.0.0
**Next Review**: 2024-04-15 