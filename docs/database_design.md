# Investment Product Sales Tool - Database Design

## Overview

This document outlines the database design for the Investment Product Sales Tool using Amazon Aurora V2 (PostgreSQL-compatible). The design focuses on scalability, performance, and data integrity while supporting the application's core functionality.

## Database Architecture

### Technology Stack
- **Database Engine**: Amazon Aurora V2 (PostgreSQL-compatible)
- **Version**: PostgreSQL 14.x compatible
- **Cluster Configuration**: Multi-AZ deployment
- **Backup Strategy**: Point-in-time recovery enabled
- **Encryption**: AES-256 encryption at rest

## Schema Design

### Core Tables

#### 1. products
Primary table for storing investment product information.

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cusip VARCHAR(9) UNIQUE NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    product_type VARCHAR(50) NOT NULL CHECK (product_type IN ('Annuity', 'CD', 'Other')),
    description TEXT,
    minimum_investment DECIMAL(15,2) DEFAULT 0.00,
    maximum_investment DECIMAL(15,2),
    base_rate DECIMAL(8,4) NOT NULL,
    bonus_rate DECIMAL(8,4) DEFAULT 0.0000,
    return_of_premium BOOLEAN DEFAULT FALSE,
    return_of_premium_percentage DECIMAL(5,2),
    terms_and_conditions TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(255) NOT NULL,
    updated_by VARCHAR(255)
);

CREATE INDEX idx_products_cusip ON products(cusip);
CREATE INDEX idx_products_type ON products(product_type);
CREATE INDEX idx_products_active ON products(is_active);
```

#### 2. product_availability
Manages time-based product availability periods.

```sql
CREATE TABLE product_availability (
    availability_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(255) NOT NULL,
    
    CONSTRAINT check_date_range CHECK (end_date >= start_date)
);

CREATE INDEX idx_product_availability_product_id ON product_availability(product_id);
CREATE INDEX idx_product_availability_dates ON product_availability(start_date, end_date);
CREATE INDEX idx_product_availability_active ON product_availability(is_active);
```

#### 3. rate_history
Tracks historical rate changes for products.

```sql
CREATE TABLE rate_history (
    rate_history_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    base_rate DECIMAL(8,4) NOT NULL,
    bonus_rate DECIMAL(8,4) DEFAULT 0.0000,
    effective_date DATE NOT NULL,
    end_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(255) NOT NULL
);

CREATE INDEX idx_rate_history_product_id ON rate_history(product_id);
CREATE INDEX idx_rate_history_dates ON rate_history(effective_date, end_date);
```

#### 4. users
Stores user information from Okta integration.

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    okta_user_id VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_okta_id ON users(okta_user_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(is_active);
```

#### 5. user_roles
Manages user role assignments based on Okta groups.

```sql
CREATE TABLE user_roles (
    user_role_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    role_name VARCHAR(100) NOT NULL,
    okta_group_name VARCHAR(255) NOT NULL,
    assigned_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    assigned_by VARCHAR(255),
    
    CONSTRAINT check_role_name CHECK (role_name IN ('SalesRepresentative', 'SalesManager', 'ProductAdministrator', 'SystemAdministrator'))
);

CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role_name ON user_roles(role_name);
CREATE UNIQUE INDEX idx_user_roles_unique ON user_roles(user_id, role_name);
```

#### 6. audit_log
Comprehensive audit trail for all system actions.

```sql
CREATE TABLE audit_log (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id),
    action_type VARCHAR(50) NOT NULL,
    table_name VARCHAR(100),
    record_id UUID,
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    session_id VARCHAR(255)
);

CREATE INDEX idx_audit_log_user_id ON audit_log(user_id);
CREATE INDEX idx_audit_log_timestamp ON audit_log(timestamp);
CREATE INDEX idx_audit_log_action_type ON audit_log(action_type);
CREATE INDEX idx_audit_log_table_name ON audit_log(table_name);
CREATE INDEX idx_audit_log_record_id ON audit_log(record_id);
```

#### 7. user_sessions
Tracks active user sessions for security monitoring.

```sql
CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    okta_session_id VARCHAR(255),
    jwt_token_hash VARCHAR(255),
    ip_address INET,
    user_agent TEXT,
    login_time TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_activity TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    logout_time TIMESTAMP WITH TIME ZONE,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_user_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_active ON user_sessions(is_active);
CREATE INDEX idx_user_sessions_last_activity ON user_sessions(last_activity);
```

#### 8. product_comparisons
Tracks product comparison activities for analytics.

```sql
CREATE TABLE product_comparisons (
    comparison_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    product_ids UUID[] NOT NULL,
    investment_amount DECIMAL(15,2) NOT NULL,
    comparison_date TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    session_id UUID REFERENCES user_sessions(session_id)
);

CREATE INDEX idx_product_comparisons_user_id ON product_comparisons(user_id);
CREATE INDEX idx_product_comparisons_date ON product_comparisons(comparison_date);
CREATE INDEX idx_product_comparisons_session_id ON product_comparisons(session_id);
```

## Views

### 1. active_products_view
Simplified view of currently active products with availability.

```sql
CREATE VIEW active_products_view AS
SELECT 
    p.product_id,
    p.cusip,
    p.product_name,
    p.product_type,
    p.description,
    p.minimum_investment,
    p.maximum_investment,
    p.base_rate,
    p.bonus_rate,
    p.return_of_premium,
    p.return_of_premium_percentage,
    pa.start_date,
    pa.end_date
FROM products p
JOIN product_availability pa ON p.product_id = pa.product_id
WHERE p.is_active = TRUE 
  AND pa.is_active = TRUE 
  AND CURRENT_DATE BETWEEN pa.start_date AND pa.end_date;
```

### 2. user_activity_summary
Summary of user activities for reporting.

```sql
CREATE VIEW user_activity_summary AS
SELECT 
    u.user_id,
    u.email,
    u.first_name,
    u.last_name,
    COUNT(DISTINCT us.session_id) as total_sessions,
    COUNT(DISTINCT pc.comparison_id) as total_comparisons,
    MAX(us.last_activity) as last_activity,
    COUNT(DISTINCT al.audit_id) as total_actions
FROM users u
LEFT JOIN user_sessions us ON u.user_id = us.user_id
LEFT JOIN product_comparisons pc ON u.user_id = pc.user_id
LEFT JOIN audit_log al ON u.user_id = al.user_id
GROUP BY u.user_id, u.email, u.first_name, u.last_name;
```

## Stored Procedures

### 1. Update Product Rates
```sql
CREATE OR REPLACE FUNCTION update_product_rates(
    p_product_id UUID,
    p_base_rate DECIMAL(8,4),

### 2. Create Product
```sql
CREATE OR REPLACE FUNCTION create_product(
    p_cusip VARCHAR(9),
    p_product_name VARCHAR(255),
    p_product_type VARCHAR(50),
    p_description TEXT,
    p_minimum_investment DECIMAL(15,2),
    p_maximum_investment DECIMAL(15,2),
    p_base_rate DECIMAL(8,4),
    p_bonus_rate DECIMAL(8,4),
    p_return_of_premium BOOLEAN,
    p_return_of_premium_percentage DECIMAL(5,2),
    p_terms_and_conditions TEXT,
    p_created_by VARCHAR(255)
) RETURNS UUID AS $$
DECLARE
    v_product_id UUID;
BEGIN
    -- Insert new product
    INSERT INTO products (
        cusip, product_name, product_type, description,
        minimum_investment, maximum_investment, base_rate, bonus_rate,
        return_of_premium, return_of_premium_percentage, terms_and_conditions, created_by
    ) VALUES (
        p_cusip, p_product_name, p_product_type, p_description,
        p_minimum_investment, p_maximum_investment, p_base_rate, p_bonus_rate,
        p_return_of_premium, p_return_of_premium_percentage, p_terms_and_conditions, p_created_by
    ) RETURNING product_id INTO v_product_id;
    
    -- Log the action
    PERFORM log_user_action(
        NULL,
        'CREATE_PRODUCT',
        'products',
        v_product_id,
        NULL,
        jsonb_build_object(
            'cusip', p_cusip,
            'product_name', p_product_name,
            'product_type', p_product_type
        ),
        NULL, NULL, NULL
    );
    
    RETURN v_product_id;
END;
$$ LANGUAGE plpgsql;
```

### 3. Deactivate Product
```sql
CREATE OR REPLACE FUNCTION deactivate_product(
    p_product_id UUID,
    p_updated_by VARCHAR(255)
) RETURNS VOID AS $$
BEGIN
    -- Deactivate the product
    UPDATE products 
    SET is_active = FALSE,
        updated_at = CURRENT_TIMESTAMP,
        updated_by = p_updated_by
    WHERE product_id = p_product_id;
    
    -- Log the action
    PERFORM log_user_action(
        NULL,
        'DEACTIVATE_PRODUCT',
        'products',
        p_product_id,
        jsonb_build_object('is_active', TRUE),
        jsonb_build_object('is_active', FALSE),
        NULL, NULL, NULL
    );
END;
$$ LANGUAGE plpgsql;
```

### 4. Get Active Products with Rates
```sql
CREATE OR REPLACE FUNCTION get_active_products_with_rates(
    p_product_type VARCHAR(50) DEFAULT NULL,
    p_min_rate DECIMAL(8,4) DEFAULT NULL,
    p_max_rate DECIMAL(8,4) DEFAULT NULL
) RETURNS TABLE (
    product_id UUID,
    cusip VARCHAR(9),
    product_name VARCHAR(255),
    product_type VARCHAR(50),
    description TEXT,
    minimum_investment DECIMAL(15,2),
    maximum_investment DECIMAL(15,2),
    base_rate DECIMAL(8,4),
    bonus_rate DECIMAL(8,4),
    return_of_premium BOOLEAN,
    return_of_premium_percentage DECIMAL(5,2),
    start_date DATE,
    end_date DATE
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        p.product_id,
        p.cusip,
        p.product_name,
        p.product_type,
        p.description,
        p.minimum_investment,
        p.maximum_investment,
        p.base_rate,
        p.bonus_rate,
        p.return_of_premium,
        p.return_of_premium_percentage,
        pa.start_date,
        pa.end_date
    FROM products p
    JOIN product_availability pa ON p.product_id = pa.product_id
    WHERE p.is_active = TRUE 
      AND pa.is_active = TRUE 
      AND CURRENT_DATE BETWEEN pa.start_date AND pa.end_date
      AND (p_product_type IS NULL OR p.product_type = p_product_type)
      AND (p_min_rate IS NULL OR p.base_rate >= p_min_rate)
      AND (p_max_rate IS NULL OR p.base_rate <= p_max_rate)
    ORDER BY p.product_type, p.base_rate DESC;
END;
$$ LANGUAGE plpgsql;
```
    p_bonus_rate DECIMAL(8,4),
    p_updated_by VARCHAR(255)
) RETURNS VOID AS $$
BEGIN
    -- Insert into rate history
    INSERT INTO rate_history (product_id, base_rate, bonus_rate, effective_date, created_by)
    VALUES (p_product_id, p_base_rate, p_bonus_rate, CURRENT_DATE, p_updated_by);
    
    -- Update current product rates
    UPDATE products 
    SET base_rate = p_base_rate,
        bonus_rate = p_bonus_rate,
        updated_at = CURRENT_TIMESTAMP,
        updated_by = p_updated_by
    WHERE product_id = p_product_id;
END;
$$ LANGUAGE plpgsql;
```

### 2. Log User Action
```sql
CREATE OR REPLACE FUNCTION log_user_action(
    p_user_id UUID,
    p_action_type VARCHAR(50),
    p_table_name VARCHAR(100),
    p_record_id UUID,
    p_old_values JSONB,
    p_new_values JSONB,
    p_ip_address INET,
    p_user_agent TEXT,
    p_session_id VARCHAR(255)
) RETURNS VOID AS $$
BEGIN
    INSERT INTO audit_log (
        user_id, action_type, table_name, record_id, 
        old_values, new_values, ip_address, user_agent, session_id
    ) VALUES (
        p_user_id, p_action_type, p_table_name, p_record_id,
        p_old_values, p_new_values, p_ip_address, p_user_agent, p_session_id
    );
END;
$$ LANGUAGE plpgsql;
```

## Triggers

### 1. Product Update Trigger
```sql
CREATE OR REPLACE FUNCTION product_update_trigger() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_update_trigger
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION product_update_trigger();
```

### 2. User Activity Trigger
```sql
CREATE OR REPLACE FUNCTION user_activity_trigger() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_update_trigger
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION user_activity_trigger();
```

## Security Considerations

### Row-Level Security (RLS)
```sql
-- Enable RLS on sensitive tables
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Create policies for product access
CREATE POLICY product_access_policy ON products
    FOR ALL
    TO application_role
    USING (is_active = true);

-- Create policies for audit log access
CREATE POLICY audit_log_access_policy ON audit_log
    FOR SELECT
    TO application_role
    USING (user_id = current_setting('app.current_user_id')::UUID OR 
           current_setting('app.current_user_role') = 'SystemAdministrator');
```

### Database Roles
```sql
-- Create application roles
CREATE ROLE app_read_only;
CREATE ROLE app_read_write;
CREATE ROLE app_admin;

-- Grant permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read_only;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO app_read_write;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_admin;
```

## Performance Optimization

### Indexing Strategy
- Primary keys on all tables using UUID
- Foreign key indexes for join performance
- Composite indexes for common query patterns
- Partial indexes for filtered queries

### Query Optimization
- Use of materialized views for complex aggregations
- Prepared statements for frequently executed queries
- Connection pooling for Lambda functions
- Query result caching where appropriate

## Backup and Recovery

### Backup Strategy
- Automated daily backups with 7-day retention
- Point-in-time recovery enabled
- Cross-region backup replication
- Regular backup restoration testing

### Disaster Recovery
- Multi-AZ deployment for high availability
- Read replicas for load distribution
- Automated failover configuration
- Recovery time objective (RTO): 15 minutes
- Recovery point objective (RPO): 1 hour

## Monitoring and Maintenance

### Performance Monitoring
- CloudWatch metrics for database performance
- Query performance insights
- Connection pool monitoring
- Slow query logging

### Maintenance Tasks
- Regular index maintenance
- Statistics updates
- Vacuum operations
- Audit log archival (monthly)
- Rate history cleanup (retain 7 years)

## Data Migration

### Initial Data Load
- CUSIP validation during import
- Rate history initialization
- User data synchronization with Okta
- Product availability setup

### Ongoing Synchronization
- Real-time rate updates from external systems
- User synchronization with Okta
- Audit log retention policies
- Data archival procedures