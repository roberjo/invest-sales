# Investment Product Sales Tool - Security Guide

## Overview

This document outlines the comprehensive security implementation for the Investment Product Sales Tool, covering authentication, authorization, data protection, compliance requirements, and security best practices.

## Security Architecture

### High-Level Security Model

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Okta Auth     │    │   API Gateway   │    │   Application   │
│   (OAuth 2.0)   │◄──►│   (Rate Limit)  │◄──►│   (JWT Valid)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   WAF/Shield    │    │   VPC/Private   │    │   Encrypted DB  │
│   (DDoS Prot)   │    │   Subnets       │    │   (AES-256)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Authentication & Authorization

### Okta Integration

#### Configuration
```json
{
  "okta": {
    "domain": "your-company.okta.com",
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "audience": "api://default",
    "issuer": "https://your-company.okta.com/oauth2/default"
  }
}
```

#### JWT Token Validation
```csharp
public class JwtValidationMiddleware
{
    private readonly OktaSettings _oktaSettings;
    
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var token = ExtractToken(context);
        
        if (string.IsNullOrEmpty(token))
        {
            context.Response.StatusCode = 401;
            return;
        }
        
        try
        {
            var principal = await ValidateTokenAsync(token);
            context.User = principal;
            await next(context);
        }
        catch (Exception)
        {
            context.Response.StatusCode = 401;
        }
    }
    
    private async Task<ClaimsPrincipal> ValidateTokenAsync(string token)
    {
        var configurationManager = new ConfigurationManager<OpenIdConnectConfiguration>(
            $"{_oktaSettings.Issuer}/.well-known/openid_configuration",
            new OpenIdConnectConfigurationRetriever());
            
        var configuration = await configurationManager.GetConfigurationAsync();
        
        var tokenHandler = new JwtSecurityTokenHandler();
        var validationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKeys = configuration.SigningKeys,
            ValidateIssuer = true,
            ValidIssuer = _oktaSettings.Issuer,
            ValidateAudience = true,
            ValidAudience = _oktaSettings.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(5)
        };
        
        return tokenHandler.ValidateToken(token, validationParameters, out _);
    }
}
```

### Role-Based Access Control (RBAC)

#### Role Definitions
```csharp
public static class Roles
{
    public const string SalesRepresentative = "SalesRepresentative";
    public const string SalesManager = "SalesManager";
    public const string ProductAdministrator = "ProductAdministrator";
    public const string SystemAdministrator = "SystemAdministrator";
}

public static class Permissions
{
    // Product permissions
    public const string ViewProducts = "products:read";
    public const string CreateProducts = "products:create";
    public const string UpdateProducts = "products:update";
    public const string DeleteProducts = "products:delete";
    
    // Comparison permissions
    public const string CreateComparisons = "comparisons:create";
    public const string ViewComparisons = "comparisons:read";
    
    // User management permissions
    public const string ViewUsers = "users:read";
    public const string UpdateUsers = "users:update";
    
    // Audit permissions
    public const string ViewAudit = "audit:read";
    public const string ExportAudit = "audit:export";
}
```

#### Authorization Attributes
```csharp
[Authorize(Roles = Roles.ProductAdministrator)]
[HttpPost("products")]
public async Task<IActionResult> CreateProduct([FromBody] CreateProductRequest request)
{
    // Implementation
}

[Authorize(Policy = "CanViewAudit")]
[HttpGet("audit")]
public async Task<IActionResult> GetAuditLog([FromQuery] AuditFilterRequest request)
{
    // Implementation
}
```

#### Policy-Based Authorization
```csharp
public class AuthorizationPolicies
{
    public static void Configure(AuthorizationOptions options)
    {
        options.AddPolicy("CanViewAudit", policy =>
            policy.RequireRole(Roles.ProductAdministrator, Roles.SystemAdministrator));
            
        options.AddPolicy("CanManageUsers", policy =>
            policy.RequireRole(Roles.SystemAdministrator));
            
        options.AddPolicy("CanExportData", policy =>
            policy.RequireRole(Roles.ProductAdministrator, Roles.SystemAdministrator));
    }
}
```

## Data Protection

### Encryption Standards

#### Data at Rest
- **Database**: AES-256 encryption using AWS Aurora encryption
- **Backup**: Encrypted backups with AWS KMS
- **File Storage**: S3 server-side encryption with AES-256

#### Data in Transit
- **HTTPS**: TLS 1.3 for all web traffic
- **API**: TLS 1.3 for all API communications
- **Database**: SSL/TLS for database connections

#### Implementation
```csharp
public class EncryptionService
{
    private readonly IKeyVaultService _keyVault;
    
    public async Task<string> EncryptSensitiveDataAsync(string plaintext)
    {
        var key = await _keyVault.GetEncryptionKeyAsync();
        using var aes = Aes.Create();
        aes.Key = key;
        aes.GenerateIV();
        
        using var encryptor = aes.CreateEncryptor();
        using var msEncrypt = new MemoryStream();
        using var csEncrypt = new CryptoStream(msEncrypt, encryptor, CryptoStreamMode.Write);
        using var swEncrypt = new StreamWriter(csEncrypt);
        
        await swEncrypt.WriteAsync(plaintext);
        await swEncrypt.FlushAsync();
        
        var encrypted = msEncrypt.ToArray();
        var result = new byte[aes.IV.Length + encrypted.Length];
        Array.Copy(aes.IV, 0, result, 0, aes.IV.Length);
        Array.Copy(encrypted, 0, result, aes.IV.Length, encrypted.Length);
        
        return Convert.ToBase64String(result);
    }
    
    public async Task<string> DecryptSensitiveDataAsync(string ciphertext)
    {
        var key = await _keyVault.GetEncryptionKeyAsync();
        var fullCipher = Convert.FromBase64String(ciphertext);
        
        using var aes = Aes.Create();
        aes.Key = key;
        
        var iv = new byte[aes.IV.Length];
        var cipher = new byte[fullCipher.Length - aes.IV.Length];
        
        Array.Copy(fullCipher, 0, iv, 0, iv.Length);
        Array.Copy(fullCipher, iv.Length, cipher, 0, cipher.Length);
        
        aes.IV = iv;
        
        using var decryptor = aes.CreateDecryptor();
        using var msDecrypt = new MemoryStream(cipher);
        using var csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read);
        using var srDecrypt = new StreamReader(csDecrypt);
        
        return await srDecrypt.ReadToEndAsync();
    }
}
```

### Input Validation & Sanitization

#### Request Validation
```csharp
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        RuleFor(x => x.Cusip)
            .NotEmpty()
            .Length(9)
            .Matches(@"^[A-Z0-9]{9}$")
            .WithMessage("CUSIP must be exactly 9 alphanumeric characters");
            
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(255)
            .Matches(@"^[a-zA-Z0-9\s\-_\.]+$")
            .WithMessage("Product name contains invalid characters");
            
        RuleFor(x => x.BaseRate)
            .InclusiveBetween(0, 100)
            .WithMessage("Base rate must be between 0 and 100");
            
        RuleFor(x => x.MinimumInvestment)
            .GreaterThan(0)
            .WithMessage("Minimum investment must be greater than 0");
    }
}
```

#### SQL Injection Prevention
```csharp
public class ProductRepository
{
    private readonly DbContext _context;
    
    public async Task<IEnumerable<Product>> GetProductsAsync(ProductFilter filter)
    {
        var query = _context.Products.AsQueryable();
        
        if (!string.IsNullOrEmpty(filter.Type))
        {
            query = query.Where(p => p.ProductType == filter.Type);
        }
        
        if (filter.MinRate.HasValue)
        {
            query = query.Where(p => p.BaseRate >= filter.MinRate.Value);
        }
        
        if (filter.MaxRate.HasValue)
        {
            query = query.Where(p => p.BaseRate <= filter.MaxRate.Value);
        }
        
        return await query.ToListAsync();
    }
}
```

#### XSS Prevention
```csharp
public class XssPreventionMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // Validate and sanitize input
        if (context.Request.HasFormContentType)
        {
            var form = await context.Request.ReadFormAsync();
            foreach (var field in form)
            {
                if (ContainsXss(field.Value))
                {
                    context.Response.StatusCode = 400;
                    await context.Response.WriteAsync("Invalid input detected");
                    return;
                }
            }
        }
        
        await next(context);
    }
    
    private bool ContainsXss(string input)
    {
        var xssPatterns = new[]
        {
            @"<script[^>]*>.*?</script>",
            @"javascript:",
            @"on\w+\s*=",
            @"<iframe[^>]*>",
            @"<object[^>]*>"
        };
        
        return xssPatterns.Any(pattern => Regex.IsMatch(input, pattern, RegexOptions.IgnoreCase));
    }
}
```

## Network Security

### AWS Security Configuration

#### VPC Configuration
```yaml
# vpc-config.yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: 10.0.0.0/16
    EnableDnsHostnames: true
    EnableDnsSupport: true
    Tags:
      - Key: Name
        Value: InvestmentToolVPC

PrivateSubnet1:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.1.0/24
    AvailabilityZone: !Select [0, !GetAZs '']
    Tags:
      - Key: Name
        Value: PrivateSubnet1

PrivateSubnet2:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.2.0/24
    AvailabilityZone: !Select [1, !GetAZs '']
    Tags:
      - Key: Name
        Value: PrivateSubnet2
```

#### Security Groups
```yaml
LambdaSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for Lambda functions
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
    SecurityGroupEgress:
      - IpProtocol: tcp
        FromPort: 5432
        ToPort: 5432
        DestinationSecurityGroupId: !Ref DatabaseSecurityGroup

DatabaseSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for Aurora database
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 5432
        ToPort: 5432
        SourceSecurityGroupId: !Ref LambdaSecurityGroup
```

### Web Application Firewall (WAF)

#### WAF Rules
```yaml
WAFWebACL:
  Type: AWS::WAFv2::WebACL
  Properties:
    Name: InvestmentToolWAF
    Description: WAF for Investment Tool
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
```

## Compliance & Audit

### SOC 2 Type II Compliance

#### Control Objectives
1. **CC1 - Control Environment**
   - Security policies and procedures
   - Security awareness training
   - Background checks for personnel

2. **CC2 - Communication and Information**
   - Security incident reporting
   - Change management procedures
   - Vendor management

3. **CC3 - Risk Assessment**
   - Regular security assessments
   - Vulnerability management
   - Threat modeling

4. **CC4 - Monitoring Activities**
   - Continuous monitoring
   - Log analysis
   - Performance metrics

5. **CC5 - Control Activities**
   - Access controls
   - Change management
   - Incident response

#### Implementation Checklist
- [ ] Security policies documented and reviewed annually
- [ ] Background checks for all personnel
- [ ] Security awareness training completed
- [ ] Regular vulnerability assessments
- [ ] Incident response plan tested
- [ ] Access controls implemented and monitored
- [ ] Change management procedures followed
- [ ] Vendor security assessments completed
- [ ] Continuous monitoring implemented
- [ ] Regular security audits conducted

### Audit Trail Implementation

#### Comprehensive Logging
```csharp
public class AuditService
{
    private readonly DbContext _context;
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public async Task LogActionAsync(string actionType, string tableName, 
        Guid? recordId, object oldValues = null, object newValues = null)
    {
        var user = _httpContextAccessor.HttpContext?.User;
        var userId = user?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        var auditEntry = new AuditLog
        {
            UserId = userId != null ? Guid.Parse(userId) : null,
            ActionType = actionType,
            TableName = tableName,
            RecordId = recordId,
            OldValues = oldValues != null ? JsonSerializer.Serialize(oldValues) : null,
            NewValues = newValues != null ? JsonSerializer.Serialize(newValues) : null,
            IpAddress = GetClientIpAddress(),
            UserAgent = GetUserAgent(),
            SessionId = GetSessionId(),
            Timestamp = DateTime.UtcNow
        };
        
        _context.AuditLogs.Add(auditEntry);
        await _context.SaveChangesAsync();
    }
    
    private string GetClientIpAddress()
    {
        return _httpContextAccessor.HttpContext?.Connection?.RemoteIpAddress?.ToString();
    }
    
    private string GetUserAgent()
    {
        return _httpContextAccessor.HttpContext?.Request?.Headers["User-Agent"].ToString();
    }
    
    private string GetSessionId()
    {
        return _httpContextAccessor.HttpContext?.Session?.Id;
    }
}
```

#### Data Retention Policy
```csharp
public class DataRetentionService
{
    private readonly DbContext _context;
    
    public async Task ArchiveOldAuditLogsAsync()
    {
        var cutoffDate = DateTime.UtcNow.AddYears(-7);
        
        var oldLogs = await _context.AuditLogs
            .Where(log => log.Timestamp < cutoffDate)
            .ToListAsync();
            
        // Archive to S3 Glacier
        foreach (var log in oldLogs)
        {
            await ArchiveToS3Async(log);
        }
        
        // Remove from database
        _context.AuditLogs.RemoveRange(oldLogs);
        await _context.SaveChangesAsync();
    }
    
    private async Task ArchiveToS3Async(AuditLog log)
    {
        // Implementation for S3 Glacier archival
    }
}
```

## Security Monitoring

### CloudWatch Alarms

#### Security Alarms
```yaml
SecurityAlarms:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: SecurityBreachAlarm
    AlarmDescription: Alert on potential security breaches
    MetricName: SecurityBreachCount
    Namespace: InvestmentTool/Security
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    AlarmActions:
      - !Ref SecuritySNSTopic
```

#### Performance Alarms
```yaml
PerformanceAlarms:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: HighErrorRateAlarm
    AlarmDescription: Alert on high error rates
    MetricName: ErrorRate
    Namespace: InvestmentTool/API
    Statistic: Average
    Period: 300
    EvaluationPeriods: 2
    Threshold: 5
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref OperationsSNSTopic
```

### Security Incident Response

#### Incident Response Plan
1. **Detection**
   - Automated monitoring alerts
   - Manual incident reports
   - Security tool notifications

2. **Assessment**
   - Severity classification
   - Impact assessment
   - Stakeholder notification

3. **Containment**
   - Isolate affected systems
   - Block malicious traffic
   - Preserve evidence

4. **Eradication**
   - Remove threat
   - Patch vulnerabilities
   - Update security controls

5. **Recovery**
   - Restore services
   - Verify security
   - Monitor for recurrence

6. **Post-Incident**
   - Document lessons learned
   - Update procedures
   - Conduct review

#### Incident Response Team
- **Incident Commander**: Technical Lead
- **Security Analyst**: Security Engineer
- **Communications**: Product Manager
- **Legal**: Legal Counsel
- **External Relations**: PR Team

## Penetration Testing

### Testing Schedule
- **Quarterly**: External penetration testing
- **Monthly**: Internal vulnerability scans
- **Weekly**: Automated security scans
- **Continuous**: Real-time security monitoring

### Testing Scope
- Web application security
- API security
- Network security
- Database security
- Social engineering

### Remediation Process
1. **Vulnerability Assessment**
   - Severity classification
   - Risk assessment
   - Remediation timeline

2. **Patch Management**
   - Security patch testing
   - Deployment procedures
   - Rollback plans

3. **Verification**
   - Retesting after fixes
   - Validation of remediation
   - Documentation updates

## Security Best Practices

### Development Security

#### Secure Coding Guidelines
```csharp
// DO: Use parameterized queries
var products = await context.Products
    .Where(p => p.Type == productType)
    .ToListAsync();

// DON'T: Use string concatenation for SQL
var query = $"SELECT * FROM Products WHERE Type = '{productType}'";

// DO: Validate and sanitize input
public async Task<IActionResult> CreateProduct([FromBody] CreateProductRequest request)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    
    // Additional validation
    if (string.IsNullOrEmpty(request.Cusip) || request.Cusip.Length != 9)
    {
        return BadRequest("Invalid CUSIP");
    }
}

// DON'T: Trust user input without validation
public async Task<IActionResult> CreateProduct(string cusip, string name)
{
    // Direct use without validation
    var product = new Product { Cusip = cusip, Name = name };
}
```

### Infrastructure Security with Terraform Cloud

#### State Management Security
```hcl
# Secure remote state configuration
terraform {
  cloud {
    organization = "investment-tool"
    workspaces {
      name = "investment-tool-${var.environment}"
    }
  }
}
```

#### Access Control Best Practices
- **Workspace Permissions**: Use Terraform Cloud teams for role-based access
- **Variable Encryption**: Store sensitive data in Terraform Cloud variables (not in code)
- **Audit Logging**: Enable comprehensive audit trails for all infrastructure changes
- **Environment Isolation**: Use separate workspaces for dev/staging/prod

#### Security Configuration
```hcl
# IAM least privilege policies
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

#### Dependency Management
```json
{
  "dependencies": {
    "Microsoft.AspNetCore.Authentication.JwtBearer": "8.0.0",
    "Microsoft.EntityFrameworkCore": "8.0.0",
    "FluentValidation.AspNetCore": "11.0.0"
  },
  "devDependencies": {
    "OWASP.Zap": "2.14.0",
    "SonarQube": "10.0.0"
  }
}
```

### Infrastructure Security

#### Secrets Management
```csharp
public class SecretsManager
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

#### Network Security
- Private subnets for Lambda functions
- Security groups with minimal required access
- VPC endpoints for AWS services
- Network ACLs for additional protection

## Compliance Reporting

### Monthly Security Reports
- Security incident summary
- Vulnerability assessment results
- Access control review
- Compliance status update

### Quarterly Compliance Reviews
- SOC 2 control testing
- Security policy updates
- Risk assessment review
- Training completion verification

### Annual Security Assessment
- Comprehensive security audit
- Penetration testing results
- Security architecture review
- Compliance certification renewal

## Security Training

### Required Training
- **Annual**: Security awareness training
- **Quarterly**: Phishing simulation exercises
- **Monthly**: Security updates and reminders
- **Onboarding**: Security orientation for new employees

### Training Topics
- Password security
- Phishing awareness
- Social engineering
- Data protection
- Incident reporting
- Secure development practices

## Contact Information

### Security Team
- **Security Lead**: security@investment-tool.com
- **Incident Response**: security-incident@investment-tool.com
- **Compliance**: compliance@investment-tool.com

### Emergency Contacts
- **24/7 Security Hotline**: +1-555-SECURITY
- **After Hours**: security-emergency@investment-tool.com

---

**Last Updated**: 2024-01-15
**Version**: 1.0.0
**Next Review**: 2024-04-15 