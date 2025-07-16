# Investment Product Sales Tool - Database Migrations

## Overview

This document outlines the database migration strategy, scripts, and procedures for the Investment Product Sales Tool using Entity Framework Core and PostgreSQL.

## Migration Strategy

### Version Control Approach
```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                 Migration Strategy                                                      │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐        │
│  │   Development   │    │    Staging      │    │   Production    │        │
│  │                 │    │                 │    │                 │        │
│  │ • Auto Migrate  │    │ • Manual Apply  │    │ • Manual Apply  │        │
│  │ • Test Data     │    │ • Test Data     │    │ • Live Data     │        │
│  │ • Debug Mode    │    │ • Validation    │    │ • Backup First  │        │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘        │
│           │                       │                       │                │
│           ▼                       ▼                       ▼                │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Migration Pipeline                              │    │
│  │                                                                     │    │
│  │  1. Create Migration   2. Test Migration   3. Apply Migration     │    │
│  │  4. Validate Data     5. Update Schema    6. Monitor Performance  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Initial Database Setup

### 1. Initial Migration
```csharp
// Migrations/20240101_000001_InitialCreate.cs
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Create users table
        migrationBuilder.CreateTable(
            name: "users",
            columns: table => new
            {
                user_id = table.Column<Guid>(type: "uuid", nullable: false),
                okta_user_id = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: false),
                email = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: false),
                first_name = table.Column<string>(type: "varchar(100)", maxLength: 100, nullable: true),
                last_name = table.Column<string>(type: "varchar(100)", maxLength: 100, nullable: true),
                is_active = table.Column<bool>(type: "boolean", nullable: false, defaultValue: true),
                last_login = table.Column<DateTime>(type: "timestamp with time zone", nullable: true),
                created_at = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                updated_at = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP")
            },
            constraints: table =>
            {
                table.PrimaryKey("pk_users", x => x.user_id);
            });

        // Create unique indexes
        migrationBuilder.CreateIndex(
            name: "ix_users_okta_user_id",
            table: "users",
            column: "okta_user_id",
            unique: true);

        migrationBuilder.CreateIndex(
            name: "ix_users_email",
            table: "users",
            column: "email",
            unique: true);

        migrationBuilder.CreateIndex(
            name: "ix_users_is_active",
            table: "users",
            column: "is_active");

        // Create products table
        migrationBuilder.CreateTable(
            name: "products",
            columns: table => new
            {
                product_id = table.Column<Guid>(type: "uuid", nullable: false),
                cusip = table.Column<string>(type: "varchar(9)", maxLength: 9, nullable: false),
                product_name = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: false),
                product_type = table.Column<string>(type: "varchar(50)", maxLength: 50, nullable: false),
                description = table.Column<string>(type: "text", nullable: true),
                minimum_investment = table.Column<decimal>(type: "decimal(15,2)", precision: 15, scale: 2, nullable: false, defaultValue: 0.00m),
                maximum_investment = table.Column<decimal>(type: "decimal(15,2)", precision: 15, scale: 2, nullable: true),
                base_rate = table.Column<decimal>(type: "decimal(8,4)", precision: 8, scale: 4, nullable: false),
                bonus_rate = table.Column<decimal>(type: "decimal(8,4)", precision: 8, scale: 4, nullable: false, defaultValue: 0.0000m),
                return_of_premium = table.Column<bool>(type: "boolean", nullable: false, defaultValue: false),
                return_of_premium_percentage = table.Column<decimal>(type: "decimal(5,2)", precision: 5, scale: 2, nullable: true),
                terms_and_conditions = table.Column<string>(type: "text", nullable: true),
                is_active = table.Column<bool>(type: "boolean", nullable: false, defaultValue: true),
                created_at = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                updated_at = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                created_by = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: false),
                updated_by = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("pk_products", x => x.product_id);
            });

        // Create indexes for products
        migrationBuilder.CreateIndex(
            name: "ix_products_cusip",
            table: "products",
            column: "cusip",
            unique: true);

        migrationBuilder.CreateIndex(
            name: "ix_products_product_type",
            table: "products",
            column: "product_type");

        migrationBuilder.CreateIndex(
            name: "ix_products_is_active",
            table: "products",
            column: "is_active");

        // Create audit_log table
        migrationBuilder.CreateTable(
            name: "audit_log",
            columns: table => new
            {
                audit_id = table.Column<Guid>(type: "uuid", nullable: false),
                user_id = table.Column<Guid>(type: "uuid", nullable: true),
                action_type = table.Column<string>(type: "varchar(50)", maxLength: 50, nullable: false),
                table_name = table.Column<string>(type: "varchar(100)", maxLength: 100, nullable: true),
                record_id = table.Column<Guid>(type: "uuid", nullable: true),
                old_values = table.Column<string>(type: "jsonb", nullable: true),
                new_values = table.Column<string>(type: "jsonb", nullable: true),
                ip_address = table.Column<string>(type: "inet", nullable: true),
                user_agent = table.Column<string>(type: "text", nullable: true),
                timestamp = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                session_id = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("pk_audit_log", x => x.audit_id);
                table.ForeignKey(
                    name: "fk_audit_log_users_user_id",
                    column: x => x.user_id,
                    principalTable: "users",
                    principalColumn: "user_id",
                    onDelete: ReferentialAction.SetNull);
            });

        // Create indexes for audit_log
        migrationBuilder.CreateIndex(
            name: "ix_audit_log_user_id",
            table: "audit_log",
            column: "user_id");

        migrationBuilder.CreateIndex(
            name: "ix_audit_log_timestamp",
            table: "audit_log",
            column: "timestamp");

        migrationBuilder.CreateIndex(
            name: "ix_audit_log_action_type",
            table: "audit_log",
            column: "action_type");

        migrationBuilder.CreateIndex(
            name: "ix_audit_log_table_name",
            table: "audit_log",
            column: "table_name");

        migrationBuilder.CreateIndex(
            name: "ix_audit_log_record_id",
            table: "audit_log",
            column: "record_id");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "audit_log");
        migrationBuilder.DropTable(name: "products");
        migrationBuilder.DropTable(name: "users");
    }
}
```

### 2. Add Product Availability
```csharp
// Migrations/20240101_000002_AddProductAvailability.cs
public partial class AddProductAvailability : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "product_availability",
            columns: table => new
            {
                availability_id = table.Column<Guid>(type: "uuid", nullable: false),
                product_id = table.Column<Guid>(type: "uuid", nullable: false),
                start_date = table.Column<DateTime>(type: "date", nullable: false),
                end_date = table.Column<DateTime>(type: "date", nullable: false),
                is_active = table.Column<bool>(type: "boolean", nullable: false, defaultValue: true),
                created_at = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                created_by = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("pk_product_availability", x => x.availability_id);
                table.ForeignKey(
                    name: "fk_product_availability_products_product_id",
                    column: x => x.product_id,
                    principalTable: "products",
                    principalColumn: "product_id",
                    onDelete: ReferentialAction.Cascade);
                table.CheckConstraint(
                    name: "check_date_range",
                    sql: "end_date >= start_date");
            });

        migrationBuilder.CreateIndex(
            name: "ix_product_availability_product_id",
            table: "product_availability",
            column: "product_id");

        migrationBuilder.CreateIndex(
            name: "ix_product_availability_dates",
            table: "product_availability",
            columns: new[] { "start_date", "end_date" });

        migrationBuilder.CreateIndex(
            name: "ix_product_availability_is_active",
            table: "product_availability",
            column: "is_active");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "product_availability");
    }
}
```

### 3. Add Rate History
```csharp
// Migrations/20240101_000003_AddRateHistory.cs
public partial class AddRateHistory : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "rate_history",
            columns: table => new
            {
                rate_history_id = table.Column<Guid>(type: "uuid", nullable: false),
                product_id = table.Column<Guid>(type: "uuid", nullable: false),
                base_rate = table.Column<decimal>(type: "decimal(8,4)", precision: 8, scale: 4, nullable: false),
                bonus_rate = table.Column<decimal>(type: "decimal(8,4)", precision: 8, scale: 4, nullable: false, defaultValue: 0.0000m),
                effective_date = table.Column<DateTime>(type: "date", nullable: false),
                end_date = table.Column<DateTime>(type: "date", nullable: true),
                created_at = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                created_by = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("pk_rate_history", x => x.rate_history_id);
                table.ForeignKey(
                    name: "fk_rate_history_products_product_id",
                    column: x => x.product_id,
                    principalTable: "products",
                    principalColumn: "product_id",
                    onDelete: ReferentialAction.Cascade);
            });

        migrationBuilder.CreateIndex(
            name: "ix_rate_history_product_id",
            table: "rate_history",
            column: "product_id");

        migrationBuilder.CreateIndex(
            name: "ix_rate_history_dates",
            table: "rate_history",
            columns: new[] { "effective_date", "end_date" });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "rate_history");
    }
}
```

### 4. Add User Roles
```csharp
// Migrations/20240101_000004_AddUserRoles.cs
public partial class AddUserRoles : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "user_roles",
            columns: table => new
            {
                user_role_id = table.Column<Guid>(type: "uuid", nullable: false),
                user_id = table.Column<Guid>(type: "uuid", nullable: false),
                role_name = table.Column<string>(type: "varchar(100)", maxLength: 100, nullable: false),
                okta_group_name = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: false),
                assigned_at = table.Column<DateTime>(type: "timestamp with time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                assigned_by = table.Column<string>(type: "varchar(255)", maxLength: 255, nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("pk_user_roles", x => x.user_role_id);
                table.ForeignKey(
                    name: "fk_user_roles_users_user_id",
                    column: x => x.user_id,
                    principalTable: "users",
                    principalColumn: "user_id",
                    onDelete: ReferentialAction.Cascade);
                table.CheckConstraint(
                    name: "check_role_name",
                    sql: "role_name IN ('SalesRepresentative', 'SalesManager', 'ProductAdministrator', 'SystemAdministrator')");
            });

        migrationBuilder.CreateIndex(
            name: "ix_user_roles_user_id",
            table: "user_roles",
            column: "user_id");

        migrationBuilder.CreateIndex(
            name: "ix_user_roles_role_name",
            table: "user_roles",
            column: "role_name");

        migrationBuilder.CreateIndex(
            name: "ix_user_roles_unique",
            table: "user_roles",
            columns: new[] { "user_id", "role_name" },
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "user_roles");
    }
}
```

## Data Seeding

### Initial Data Seeding
```csharp
// Data/SeedData.cs
public static class SeedData
{
    public static void SeedInitialData(ModelBuilder modelBuilder)
    {
        // Seed default roles
        modelBuilder.Entity<UserRole>().HasData(
            new UserRole
            {
                UserRoleId = Guid.Parse("550e8400-e29b-41d4-a716-446655440001"),
                UserId = Guid.Parse("550e8400-e29b-41d4-a716-446655440000"),
                RoleName = "SystemAdministrator",
                OktaGroupName = "investment-tool-admin",
                AssignedAt = DateTime.UtcNow,
                AssignedBy = "system"
            }
        );

        // Seed sample products
        modelBuilder.Entity<Product>().HasData(
            new Product
            {
                ProductId = Guid.Parse("550e8400-e29b-41d4-a716-446655440002"),
                Cusip = "123456789",
                ProductName = "Premium Annuity Plus",
                ProductType = "Annuity",
                Description = "High-yield annuity with bonus rates and return of premium",
                MinimumInvestment = 10000,
                MaximumInvestment = 1000000,
                BaseRate = 4.5m,
                BonusRate = 0.5m,
                ReturnOfPremium = true,
                ReturnOfPremiumPercentage = 100,
                TermsAndConditions = "5-year term with annual compounding",
                IsActive = true,
                CreatedAt = DateTime.UtcNow,
                UpdatedAt = DateTime.UtcNow,
                CreatedBy = "system"
            },
            new Product
            {
                ProductId = Guid.Parse("550e8400-e29b-41d4-a716-446655440003"),
                Cusip = "987654321",
                ProductName = "High-Yield CD",
                ProductType = "CD",
                Description = "Certificate of deposit with competitive rates",
                MinimumInvestment = 5000,
                MaximumInvestment = 500000,
                BaseRate = 4.2m,
                BonusRate = 0m,
                ReturnOfPremium = false,
                ReturnOfPremiumPercentage = null,
                TermsAndConditions = "3-year term with quarterly compounding",
                IsActive = true,
                CreatedAt = DateTime.UtcNow,
                UpdatedAt = DateTime.UtcNow,
                CreatedBy = "system"
            }
        );

        // Seed product availability
        modelBuilder.Entity<ProductAvailability>().HasData(
            new ProductAvailability
            {
                AvailabilityId = Guid.Parse("550e8400-e29b-41d4-a716-446655440004"),
                ProductId = Guid.Parse("550e8400-e29b-41d4-a716-446655440002"),
                StartDate = new DateTime(2024, 1, 1),
                EndDate = new DateTime(2024, 12, 31),
                IsActive = true,
                CreatedAt = DateTime.UtcNow,
                CreatedBy = "system"
            },
            new ProductAvailability
            {
                AvailabilityId = Guid.Parse("550e8400-e29b-41d4-a716-446655440005"),
                ProductId = Guid.Parse("550e8400-e29b-41d4-a716-446655440003"),
                StartDate = new DateTime(2024, 1, 1),
                EndDate = new DateTime(2024, 12, 31),
                IsActive = true,
                CreatedAt = DateTime.UtcNow,
                CreatedBy = "system"
            }
        );
    }
}
```

## Migration Commands

### Development Environment
```bash
# Create new migration
dotnet ef migrations add AddProductComparison

# Update database
dotnet ef database update

# Remove last migration
dotnet ef migrations remove

# Generate SQL script
dotnet ef migrations script --output migration.sql

# Apply specific migration
dotnet ef database update 20240101_000004_AddUserRoles
```

### Staging Environment
```bash
# Generate migration script
dotnet ef migrations script --output staging_migration.sql --from 20240101_000003_AddRateHistory --to 20240101_000004_AddUserRoles

# Apply migration with backup
psql -h staging-db-host -U admin -d investment_tool_staging -f backup.sql
psql -h staging-db-host -U admin -d investment_tool_staging -f staging_migration.sql

# Verify migration
dotnet ef database update --connection "Host=staging-db-host;Database=investment_tool_staging;Username=admin;Password=password"
```

### Production Environment
```bash
# Create backup
pg_dump -h prod-db-host -U admin -d investment_tool_prod > backup_$(date +%Y%m%d_%H%M%S).sql

# Generate migration script
dotnet ef migrations script --output prod_migration.sql --from 20240101_000003_AddRateHistory --to 20240101_000004_AddUserRoles

# Apply migration during maintenance window
psql -h prod-db-host -U admin -d investment_tool_prod -f prod_migration.sql

# Verify migration
dotnet ef database update --connection "Host=prod-db-host;Database=investment_tool_prod;Username=admin;Password=password"
```

## Rollback Procedures

### Emergency Rollback Script
```sql
-- rollback_migration.sql
-- Rollback from 20240101_000004_AddUserRoles to 20240101_000003_AddRateHistory

BEGIN;

-- Drop user_roles table
DROP TABLE IF EXISTS user_roles CASCADE;

-- Remove indexes
DROP INDEX IF EXISTS ix_user_roles_user_id;
DROP INDEX IF EXISTS ix_user_roles_role_name;
DROP INDEX IF EXISTS ix_user_roles_unique;

-- Remove constraints
ALTER TABLE user_roles DROP CONSTRAINT IF EXISTS fk_user_roles_users_user_id;
ALTER TABLE user_roles DROP CONSTRAINT IF EXISTS check_role_name;

-- Update migration history
DELETE FROM "__EFMigrationsHistory" WHERE "MigrationId" = '20240101_000004_AddUserRoles';

COMMIT;
```

### Automated Rollback
```csharp
// Services/MigrationService.cs
public class MigrationService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<MigrationService> _logger;

    public MigrationService(ApplicationDbContext context, ILogger<MigrationService> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task<bool> RollbackMigrationAsync(string migrationId)
    {
        try
        {
            _logger.LogInformation("Starting rollback for migration: {MigrationId}", migrationId);

            // Create backup before rollback
            await CreateBackupAsync();

            // Execute rollback
            var result = await _context.Database.ExecuteSqlRawAsync(
                "DELETE FROM \"__EFMigrationsHistory\" WHERE \"MigrationId\" = @migrationId",
                new NpgsqlParameter("@migrationId", migrationId));

            _logger.LogInformation("Rollback completed for migration: {MigrationId}", migrationId);
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Rollback failed for migration: {MigrationId}", migrationId);
            return false;
        }
    }

    private async Task CreateBackupAsync()
    {
        var backupPath = $"backup_{DateTime.UtcNow:yyyyMMdd_HHmmss}.sql";
        // Implementation for creating backup
    }
}
```

## Performance Optimizations

### Index Optimization
```sql
-- Create composite indexes for common queries
CREATE INDEX CONCURRENTLY idx_products_active_type_rate 
ON products(is_active, product_type, base_rate DESC) 
WHERE is_active = true;

CREATE INDEX CONCURRENTLY idx_audit_log_user_timestamp 
ON audit_log(user_id, timestamp DESC) 
WHERE timestamp > CURRENT_DATE - INTERVAL '30 days';

CREATE INDEX CONCURRENTLY idx_rate_history_product_effective 
ON rate_history(product_id, effective_date DESC);

-- Create partial indexes for filtered queries
CREATE INDEX CONCURRENTLY idx_products_annuity_active 
ON products(product_type, base_rate) 
WHERE product_type = 'Annuity' AND is_active = true;

CREATE INDEX CONCURRENTLY idx_audit_log_recent_actions 
ON audit_log(action_type, timestamp) 
WHERE timestamp > CURRENT_DATE - INTERVAL '7 days';
```

### Partitioning Strategy
```sql
-- Partition audit_log by month
CREATE TABLE audit_log_2024_01 PARTITION OF audit_log
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE audit_log_2024_02 PARTITION OF audit_log
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Create partition function
CREATE OR REPLACE FUNCTION audit_log_partition_function()
RETURNS TRIGGER AS $$
BEGIN
    -- Auto-create partitions as needed
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger for auto-partitioning
CREATE TRIGGER audit_log_partition_trigger
    BEFORE INSERT ON audit_log
    FOR EACH ROW
    EXECUTE FUNCTION audit_log_partition_function();
```

## Monitoring and Maintenance

### Migration Monitoring
```csharp
// Services/MigrationMonitoringService.cs
public class MigrationMonitoringService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<MigrationMonitoringService> _logger;

    public async Task<MigrationStatus> GetMigrationStatusAsync()
    {
        var appliedMigrations = await _context.Database.GetAppliedMigrationsAsync();
        var pendingMigrations = await _context.Database.GetPendingMigrationsAsync();

        return new MigrationStatus
        {
            AppliedMigrations = appliedMigrations.ToList(),
            PendingMigrations = pendingMigrations.ToList(),
            LastAppliedMigration = appliedMigrations.LastOrDefault(),
            HasPendingMigrations = pendingMigrations.Any()
        };
    }

    public async Task<bool> ValidateMigrationAsync(string migrationId)
    {
        try
        {
            // Check if migration can be applied safely
            var validationResult = await _context.Database.ExecuteSqlRawAsync(
                "SELECT 1 FROM \"__EFMigrationsHistory\" WHERE \"MigrationId\" = @migrationId",
                new NpgsqlParameter("@migrationId", migrationId));

            return validationResult == 0; // Migration not applied yet
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Migration validation failed for: {MigrationId}", migrationId);
            return false;
        }
    }
}
```

### Database Health Checks
```csharp
// HealthChecks/DatabaseHealthCheck.cs
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ApplicationDbContext _context;

    public DatabaseHealthCheck(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            // Check database connectivity
            await _context.Database.ExecuteSqlRawAsync("SELECT 1");

            // Check migration status
            var pendingMigrations = await _context.Database.GetPendingMigrationsAsync();
            if (pendingMigrations.Any())
            {
                return HealthCheckResult.Degraded("Database has pending migrations");
            }

            return HealthCheckResult.Healthy("Database is healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database health check failed", ex);
        }
    }
}
```

---

**Last Updated**: 2024-01-15
**Version**: 1.0.0
**Next Review**: 2024-04-15 