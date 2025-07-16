# Investment Product Sales Tool - Deployment Guide

## Overview

This document provides comprehensive deployment procedures for the Investment Product Sales Tool across development, staging, and production environments. The deployment process follows Infrastructure as Code (IaC) principles using AWS CloudFormation and includes automated CI/CD pipelines.

## Environment Architecture

### Environment Overview
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Development   │    │    Staging      │    │   Production    │
│                 │    │                 │    │                 │
│ • Single AZ     │    │ • Multi-AZ      │    │ • Multi-AZ      │
│ • Small Inst    │    │ • Medium Inst   │    │ • Large Inst    │
│ • Dev Data      │    │ • Test Data     │    │ • Live Data     │
│ • Debug Logs    │    │ • Info Logs     │    │ • Error Logs    │
│ • Terraform     │    │ • Terraform     │    │ • Terraform     │
│   Workspace     │    │   Workspace     │    │   Workspace     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### AWS Resource Mapping
| Resource | Development | Staging | Production |
|----------|-------------|---------|------------|
| Lambda Memory | 512 MB | 1024 MB | 2048 MB |
| Lambda Timeout | 30s | 60s | 120s |
| Aurora Instance | db.t3.micro | db.t3.small | db.r6g.large |
| Auto Scaling | Disabled | Enabled | Enabled |
| Backup Retention | 7 days | 30 days | 7 years |

## Prerequisites

### Required Tools
```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Terraform CLI
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# Install AWS SAM CLI
pip install aws-sam-cli

# Install .NET 8 SDK
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install -y apt-transport-https
sudo apt-get install -y dotnet-sdk-8.0

# Install Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### AWS Configuration
```bash
# Configure AWS credentials
aws configure

# Set up AWS profiles for different environments
aws configure --profile development
aws configure --profile staging
aws configure --profile production

# Configure Terraform Cloud
terraform login
```

### Required AWS Services
- AWS Lambda
- Amazon Aurora (PostgreSQL)
- Amazon API Gateway
- Amazon CloudWatch
- AWS Secrets Manager
- AWS Certificate Manager
- Amazon Route 53
- AWS WAF
- Amazon S3

### Terraform Cloud Setup
1. **Create Terraform Cloud Organization**
   - Sign up at https://app.terraform.io
   - Create organization: `investment-tool`
   - Set up team access and permissions

2. **Create Workspaces**
   - `investment-tool-dev` (Development)
   - `investment-tool-staging` (Staging)
   - `investment-tool-prod` (Production)

3. **Configure Variables**
   - AWS credentials (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
   - Environment-specific variables
   - Sensitive data (database passwords, API keys)

## Development Environment

### Local Development Setup

#### 1. Clone Repository
```bash
git clone https://github.com/your-org/investment-tool.git
cd investment-tool
```

#### 2. Environment Configuration
```bash
# Create environment files
cp .env.example .env.local
cp backend/appsettings.Example.json backend/appsettings.Development.json

# Configure local environment
cat > .env.local << EOF
VITE_API_BASE_URL=http://localhost:5000/api
VITE_OKTA_DOMAIN=your-dev-okta-domain
VITE_OKTA_CLIENT_ID=your-dev-client-id
VITE_ENVIRONMENT=development
EOF

cat > backend/appsettings.Development.json << EOF
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=investment_tool_dev;Username=postgres;Password=devpassword"
  },
  "Okta": {
    "Domain": "your-dev-okta-domain",
    "Audience": "api://default",
    "ClientId": "your-dev-client-id"
  },
  "AWS": {
    "Region": "us-east-1",
    "Profile": "development"
  }
}
EOF
```

#### 3. Database Setup
```bash
# Install PostgreSQL locally or use Docker
docker run --name postgres-dev \
  -e POSTGRES_DB=investment_tool_dev \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=devpassword \
  -p 5432:5432 \
  -d postgres:14

# Run database migrations
cd backend
dotnet ef database update
```

#### 4. Start Development Servers
```bash
# Terminal 1: Start backend
cd backend
dotnet run

# Terminal 2: Start frontend
cd frontend
npm install
npm run dev
```

### Development Deployment

#### 1. Build Application
```bash
# Build backend
cd backend
dotnet build --configuration Release
dotnet publish --configuration Release --output ./publish

# Build frontend
cd ../frontend
npm run build
```

#### 2. Deploy to Development Environment
```bash
# Deploy infrastructure with Terraform
cd infrastructure
terraform workspace select dev
terraform init
terraform plan -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/dev.tfvars" -auto-approve

# Deploy application
sam deploy \
  --template-file backend/template.yaml \
  --stack-name investment-tool-backend-dev \
  --parameter-overrides Environment=development \
  --capabilities CAPABILITY_IAM \
  --profile development
```

## Staging Environment

### Staging Configuration

#### 1. Environment Variables
```bash
# Staging environment configuration
cat > infrastructure/staging-config.yaml << EOF
Parameters:
  Environment:
    Default: staging
  DomainName:
    Default: staging.investment-tool.com
  CertificateArn:
    Default: arn:aws:acm:us-east-1:123456789012:certificate/staging-cert
  OktaDomain:
    Default: your-company.okta.com
  OktaClientId:
    Default: staging-client-id
  DatabaseInstanceClass:
    Default: db.t3.small
  LambdaMemorySize:
    Default: 1024
  LambdaTimeout:
    Default: 60
EOF
```

#### 2. Staging Infrastructure
```hcl
# infrastructure/environments/staging.tfvars
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

#### Deploy Staging Environment
```bash
# Deploy infrastructure with Terraform
cd infrastructure
terraform workspace select staging
terraform init
terraform plan -var-file="environments/staging.tfvars"
terraform apply -var-file="environments/staging.tfvars" -auto-approve

# Deploy backend application
sam deploy \
  --template-file backend/template.yaml \
  --stack-name investment-tool-backend-staging \
  --parameter-overrides Environment=staging \
  --capabilities CAPABILITY_IAM \
  --profile staging

# Deploy frontend
aws s3 sync frontend/dist/ s3://investment-tool-staging-frontend \
  --profile staging

aws cloudfront create-invalidation \
  --distribution-id E1234567890123 \
  --paths "/*" \
  --profile staging
```

#### 3. Deploy Staging Environment
```bash
# Deploy infrastructure
aws cloudformation deploy \
  --template-file infrastructure/staging-stack.yaml \
  --stack-name investment-tool-staging \
  --parameter-overrides \
    Environment=staging \
    DomainName=staging.investment-tool.com \
    CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/staging-cert \
  --capabilities CAPABILITY_IAM \
  --profile staging

# Deploy backend application
sam deploy \
  --template-file backend/template.yaml \
  --stack-name investment-tool-backend-staging \
  --parameter-overrides \
    Environment=staging \
    VpcId=$(aws cloudformation describe-stacks \
      --stack-name investment-tool-staging \
      --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' \
      --output text \
      --profile staging) \
    DatabaseEndpoint=$(aws cloudformation describe-stacks \
      --stack-name investment-tool-staging \
      --query 'Stacks[0].Outputs[?OutputKey==`DatabaseEndpoint`].OutputValue' \
      --output text \
      --profile staging) \
  --capabilities CAPABILITY_IAM \
  --profile staging

# Deploy frontend
aws s3 sync frontend/dist/ s3://investment-tool-staging-frontend \
  --profile staging

aws cloudfront create-invalidation \
  --distribution-id E1234567890123 \
  --paths "/*" \
  --profile staging
```

## Production Environment

### Production Configuration

#### 1. High Availability Setup
```yaml
# infrastructure/production-stack.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Investment Tool Production Environment'

Parameters:
  Environment:
    Type: String
    Default: production
  DomainName:
    Type: String
    Default: investment-tool.com
  CertificateArn:
    Type: String
    Description: SSL Certificate ARN

Resources:
  # VPC with multiple AZs
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-VPC'

  # Multiple private subnets across AZs
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-PrivateSubnet1'

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-PrivateSubnet2'

  PrivateSubnet3:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: !Select [2, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-PrivateSubnet3'

  # Aurora Multi-AZ Cluster
  AuroraCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      Engine: aurora-postgresql
      EngineVersion: 14.7
      DatabaseName: investment_tool_prod
      MasterUsername: admin
      MasterUserPassword: !Ref DatabasePassword
      DBSubnetGroupName: !Ref DatabaseSubnetGroup
      VpcSecurityGroupIds:
        - !Ref DatabaseSecurityGroup
      BackupRetentionPeriod: 35
      StorageEncrypted: true
      DeletionProtection: true
      MultiAZ: true
      EngineMode: provisioned
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-AuroraCluster'

  # Auto Scaling for Lambda
  LambdaAutoScaling:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      MaxCapacity: 100
      MinCapacity: 2
      ResourceId: !Sub 'function:${LambdaFunction}:${AWS::StackName}'
      RoleARN: !GetAtt LambdaAutoScalingRole.Arn
      ScalableDimension: lambda:function:ProvisionedConcurrency
      ServiceNamespace: lambda

  LambdaAutoScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: !Sub '${Environment}-LambdaScalingPolicy'
      PolicyType: TargetTrackingScaling
      ScalableTargetId: !Ref LambdaAutoScaling
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 70.0
        PredefinedMetricType: LambdaProvisionedConcurrencyUtilization

  # Enhanced WAF
  WAFWebACL:
    Type: AWS::WAFv2::WebACL
    Properties:
      Name: !Sub '${Environment}-WAF'
      Description: Enhanced WAF for Production
      Scope: REGIONAL
      DefaultAction:
        Allow: {}
      Rules:
        - Name: RateLimitRule
          Priority: 1
          Statement:
            RateBasedStatement:
              Limit: 2000
              AggregateKeyType: IP
          Action:
            Block: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: RateLimitRule
        
        - Name: SQLInjectionRule
          Priority: 2
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesSQLiRuleSet
          OverrideAction:
            None: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: SQLInjectionRule
        
        - Name: XSSRule
          Priority: 3
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesKnownBadInputsRuleSet
          OverrideAction:
            None: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: XSSRule

  # CloudWatch Alarms
  HighErrorRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-HighErrorRate'
      AlarmDescription: Alert on high error rates
      MetricName: 5XXError
      Namespace: AWS/ApiGateway
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 10
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref SNSTopic

  HighLatencyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-HighLatency'
      AlarmDescription: Alert on high latency
      MetricName: Latency
      Namespace: AWS/ApiGateway
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 2000
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref SNSTopic

  # SNS Topic for Alerts
  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${Environment}-Alerts'
      DisplayName: !Sub '${Environment} Environment Alerts'

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-VpcId'

  DatabaseEndpoint:
    Description: Database endpoint
    Value: !GetAtt AuroraCluster.Endpoint.Address
    Export:
      Name: !Sub '${Environment}-DatabaseEndpoint'

  ApiGatewayUrl:
    Description: API Gateway URL
    Value: !Sub 'https://${ApiGatewayDomain.DomainName}'
    Export:
      Name: !Sub '${Environment}-ApiGatewayUrl'
```

#### 2. Production Deployment Script
```bash
#!/bin/bash
# deploy-production.sh

set -e

ENVIRONMENT="production"
WORKSPACE="investment-tool-prod"
BACKEND_STACK_NAME="investment-tool-backend-production"

echo "Starting production deployment..."

# Validate Terraform configuration
echo "Validating Terraform configuration..."
cd infrastructure
terraform init
terraform validate

# Deploy infrastructure with Terraform
echo "Deploying infrastructure..."
terraform workspace select prod
terraform plan -var-file="environments/prod.tfvars"
terraform apply -var-file="environments/prod.tfvars" -auto-approve

# Get infrastructure outputs
VPC_ID=$(terraform output -raw vpc_id)
DATABASE_ENDPOINT=$(terraform output -raw database_endpoint)

# Deploy backend application
echo "Deploying backend application..."
sam deploy \
  --template-file backend/template.yaml \
  --stack-name $BACKEND_STACK_NAME \
  --parameter-overrides \
    Environment=$ENVIRONMENT \
    VpcId=$VPC_ID \
    DatabaseEndpoint=$DATABASE_ENDPOINT \
    LambdaMemorySize=2048 \
    LambdaTimeout=120 \
  --capabilities CAPABILITY_IAM \
  --profile production

# Build and deploy frontend
echo "Building frontend..."
cd frontend
npm ci
npm run build

echo "Deploying frontend..."
aws s3 sync dist/ s3://investment-tool-production-frontend \
  --profile production

aws cloudfront create-invalidation \
  --distribution-id E1234567890123 \
  --paths "/*" \
  --profile production

echo "Production deployment completed successfully!"
```

## CI/CD Pipeline

### GitHub Actions Workflow

#### 1. Main Workflow
```yaml
# .github/workflows/main.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: us-east-1

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: 8.0.x
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Restore dependencies
        run: |
          cd backend
          dotnet restore
        shell: bash
      
      - name: Build backend
        run: |
          cd backend
          dotnet build --no-restore
        shell: bash
      
      - name: Run backend tests
        run: |
          cd backend
          dotnet test --no-build --verbosity normal
        shell: bash
      
      - name: Install frontend dependencies
        run: |
          cd frontend
          npm ci
        shell: bash
      
      - name: Run frontend tests
        run: |
          cd frontend
          npm test
        shell: bash
      
      - name: Run security scan
        run: |
          cd backend
          dotnet list package --vulnerable
        shell: bash
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: infrastructure

  terraform-plan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Init
        run: |
          cd infrastructure
          terraform init
      
      - name: Terraform Plan
        run: |
          cd infrastructure
          terraform plan -var-file="environments/dev.tfvars" -out=tfplan
        env:
          TF_VAR_database_password: ${{ secrets.DATABASE_PASSWORD }}

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run OWASP ZAP scan
        uses: zaproxy/action-full-scan@v0.8.0
        with:
          target: 'https://staging.investment-tool.com'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'

  deploy-staging:
    needs: [test, terraform-plan, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_STAGING }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_STAGING }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy infrastructure
        run: |
          cd infrastructure
          terraform workspace select staging
          terraform apply -var-file="environments/staging.tfvars" -auto-approve
        env:
          TF_VAR_database_password: ${{ secrets.DATABASE_PASSWORD }}
      
      - name: Deploy application
        run: |
          chmod +x scripts/deploy-staging.sh
          ./scripts/deploy-staging.sh
        shell: bash

  deploy-production:
    needs: [test, terraform-plan, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PRODUCTION }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PRODUCTION }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy infrastructure
        run: |
          cd infrastructure
          terraform workspace select prod
          terraform apply -var-file="environments/prod.tfvars" -auto-approve
        env:
          TF_VAR_database_password: ${{ secrets.DATABASE_PASSWORD }}
      
      - name: Deploy application
        run: |
          chmod +x scripts/deploy-production.sh
          ./scripts/deploy-production.sh
        shell: bash
      
      - name: Run smoke tests
        run: |
          npm install -g newman
          newman run tests/smoke-tests.json
        shell: bash
```

#### 2. Deployment Scripts
```bash
# scripts/deploy-staging.sh
#!/bin/bash
set -e

echo "Deploying to staging environment..."

# Deploy infrastructure
aws cloudformation deploy \
  --template-file infrastructure/staging-stack.yaml \
  --stack-name investment-tool-staging \
  --parameter-overrides Environment=staging \
  --capabilities CAPABILITY_IAM

# Deploy backend
sam deploy \
  --template-file backend/template.yaml \
  --stack-name investment-tool-backend-staging \
  --parameter-overrides Environment=staging \
  --capabilities CAPABILITY_IAM

# Deploy frontend
cd frontend
npm run build
aws s3 sync dist/ s3://investment-tool-staging-frontend

echo "Staging deployment completed!"
```

## Monitoring & Alerting

### CloudWatch Dashboards

#### 1. Application Dashboard
```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          ["AWS/ApiGateway", "Count", "ApiName", "InvestmentToolAPI"],
          [".", "4XXError", ".", "."],
          [".", "5XXError", ".", "."]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "API Gateway Metrics"
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Duration", "FunctionName", "InvestmentToolAPI"],
          [".", "Errors", ".", "."],
          [".", "Throttles", ".", "."]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Lambda Metrics"
      }
    }
  ]
}
```

#### 2. Database Dashboard
```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          ["AWS/RDS", "CPUUtilization", "DBClusterIdentifier", "investment-tool-prod"],
          [".", "DatabaseConnections", ".", "."],
          [".", "FreeableMemory", ".", "."]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Aurora Metrics"
      }
    }
  ]
}
```

### Alerting Configuration

#### 1. High Error Rate Alert
```yaml
HighErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: HighErrorRate
    AlarmDescription: Alert when error rate exceeds threshold
    MetricName: 5XXError
    Namespace: AWS/ApiGateway
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 10
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref SNSTopic
```

#### 2. High Latency Alert
```yaml
HighLatencyAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: HighLatency
    AlarmDescription: Alert when latency exceeds threshold
    MetricName: Latency
    Namespace: AWS/ApiGateway
    Statistic: Average
    Period: 300
    EvaluationPeriods: 2
    Threshold: 2000
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref SNSTopic
```

## Rollback Procedures

### Emergency Rollback Script
```bash
#!/bin/bash
# rollback.sh

ENVIRONMENT=$1
VERSION=$2

if [ -z "$ENVIRONMENT" ] || [ -z "$VERSION" ]; then
  echo "Usage: ./rollback.sh <environment> <version>"
  exit 1
fi

echo "Rolling back $ENVIRONMENT to version $VERSION..."

# Rollback CloudFormation stack
aws cloudformation rollback-stack \
  --stack-name investment-tool-backend-$ENVIRONMENT \
  --profile $ENVIRONMENT

# Rollback frontend
aws s3 cp s3://investment-tool-$ENVIRONMENT-frontend/versions/$VERSION/ \
  s3://investment-tool-$ENVIRONMENT-frontend/ \
  --recursive \
  --profile $ENVIRONMENT

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id E1234567890123 \
  --paths "/*" \
  --profile $ENVIRONMENT

echo "Rollback completed successfully!"
```

### Database Rollback
```sql
-- Rollback database changes
BEGIN;

-- Restore from backup
RESTORE DATABASE investment_tool_prod 
FROM 's3://investment-tool-backups/production/2024-01-15-10-30-00.sql';

-- Verify data integrity
SELECT COUNT(*) FROM products WHERE is_active = true;
SELECT COUNT(*) FROM users WHERE is_active = true;

COMMIT;
```

## Performance Optimization

### Lambda Optimization

#### 1. Memory Configuration
```yaml
# backend/template.yaml
Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet8
      Handler: InvestmentTool::InvestmentTool.Function::FunctionHandler
      CodeUri: ./publish
      MemorySize: 2048
      Timeout: 120
      Environment:
        Variables:
          ASPNETCORE_ENVIRONMENT: Production
          POWERTOOLS_SERVICE_NAME: InvestmentTool
          POWERTOOLS_METRICS_NAMESPACE: InvestmentTool
```

#### 2. Connection Pooling
```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseNpgsql(Configuration.GetConnectionString("DefaultConnection"),
            npgsqlOptions =>
            {
                npgsqlOptions.MaxPoolSize(20);
                npgsqlOptions.MinPoolSize(5);
            }));
}
```

### Database Optimization

#### 1. Aurora Configuration
```yaml
AuroraCluster:
  Type: AWS::RDS::DBCluster
  Properties:
    Engine: aurora-postgresql
    EngineVersion: 14.7
    DatabaseName: investment_tool_prod
    MasterUsername: admin
    MasterUserPassword: !Ref DatabasePassword
    BackupRetentionPeriod: 35
    StorageEncrypted: true
    DeletionProtection: true
    MultiAZ: true
    EngineMode: provisioned
    ScalingConfiguration:
      MinCapacity: 1
      MaxCapacity: 16
      AutoPause: false
```

#### 2. Query Optimization
```sql
-- Create indexes for common queries
CREATE INDEX CONCURRENTLY idx_products_active_type 
ON products(is_active, product_type);

CREATE INDEX CONCURRENTLY idx_products_rate_range 
ON products(base_rate, bonus_rate) 
WHERE is_active = true;

CREATE INDEX CONCURRENTLY idx_audit_log_timestamp 
ON audit_log(timestamp) 
WHERE timestamp > CURRENT_DATE - INTERVAL '7 days';
```

## Security Hardening

### Network Security

#### 1. VPC Configuration
```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: 10.0.0.0/16
    EnableDnsHostnames: true
    EnableDnsSupport: true
    Tags:
      - Key: Name
        Value: Production-VPC

PrivateSubnet1:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.1.0/24
    AvailabilityZone: !Select [0, !GetAZs '']
    MapPublicIpOnLaunch: false
    Tags:
      - Key: Name
        Value: Production-PrivateSubnet1
```

#### 2. Security Groups
```yaml
LambdaSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for Lambda functions
    VpcId: !Ref VPC
    SecurityGroupEgress:
      - IpProtocol: tcp
        FromPort: 5432
        ToPort: 5432
        DestinationSecurityGroupId: !Ref DatabaseSecurityGroup
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
```

### Secrets Management

#### 1. AWS Secrets Manager
```csharp
public class SecretsService
{
    private readonly IAmazonSecretsManager _secretsManager;
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        var request = new GetSecretValueRequest
        {
            SecretId = secretName
        };
        
        var response = await _secretsManager.GetSecretValueAsync(request);
        return response.SecretString;
    }
}
```

#### 2. Environment Variables
```json
{
  "Secrets": {
    "DatabasePassword": "{{resolve:secretsmanager:investment-tool/db-password}}",
    "OktaClientSecret": "{{resolve:secretsmanager:investment-tool/okta-secret}}",
    "JwtSigningKey": "{{resolve:secretsmanager:investment-tool/jwt-key}}"
  }
}
```

## Troubleshooting

### Common Issues

#### 1. Lambda Cold Start
```csharp
// Startup.cs - Optimize for cold starts
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Pre-warm database connections
    using var scope = app.ApplicationServices.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    context.Database.EnsureCreated();
}
```

#### 2. Database Connection Issues
```bash
# Check database connectivity
aws rds describe-db-clusters \
  --db-cluster-identifier investment-tool-prod \
  --profile production

# Test connection
psql -h investment-tool-prod.cluster-xyz.us-east-1.rds.amazonaws.com \
  -U admin -d investment_tool_prod
```

#### 3. API Gateway Issues
```bash
# Check API Gateway logs
aws logs describe-log-groups \
  --log-group-name-prefix /aws/apigateway \
  --profile production

# Test API endpoint
curl -H "Authorization: Bearer $TOKEN" \
  https://api.investment-tool.com/health
```

### Debugging Tools

#### 1. CloudWatch Logs
```bash
# View Lambda logs
aws logs tail /aws/lambda/investment-tool-api \
  --follow \
  --profile production

# View API Gateway logs
aws logs tail /aws/apigateway/investment-tool-api \
  --follow \
  --profile production
```

#### 2. X-Ray Tracing
```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddXRay();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseXRay("InvestmentTool");
}
```

## Maintenance Procedures

### Regular Maintenance Tasks

#### 1. Weekly Tasks
- Review CloudWatch metrics and alarms
- Check security group rules
- Verify backup completion
- Review error logs

#### 2. Monthly Tasks
- Update security patches
- Review and rotate secrets
- Analyze performance metrics
- Update SSL certificates

#### 3. Quarterly Tasks
- Conduct security assessments
- Review and update IAM policies
- Test disaster recovery procedures
- Update documentation

### Backup and Recovery

#### 1. Automated Backups
```yaml
AuroraCluster:
  Type: AWS::RDS::DBCluster
  Properties:
    BackupRetentionPeriod: 35
    PreferredBackupWindow: 03:00-04:00
    PreferredMaintenanceWindow: sun:04:00-sun:05:00
```

#### 2. Manual Backup
```bash
# Create manual backup
aws rds create-db-cluster-snapshot \
  --db-cluster-identifier investment-tool-prod \
  --db-cluster-snapshot-identifier manual-backup-$(date +%Y%m%d-%H%M%S) \
  --profile production
```

#### 3. Restore from Backup
```bash
# Restore from snapshot
aws rds restore-db-cluster-from-snapshot \
  --db-cluster-identifier investment-tool-prod-restored \
  --snapshot-identifier manual-backup-20240115-103000 \
  --engine aurora-postgresql \
  --profile production
```

---

**Last Updated**: 2024-01-15
**Version**: 1.0.0
**Next Review**: 2024-04-15 