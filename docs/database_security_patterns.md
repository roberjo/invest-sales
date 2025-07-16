# Investment Product Sales Tool - Database Security Patterns

## Overview

This document outlines the security patterns, encryption strategies, and compliance measures implemented in the Investment Product Sales Tool database design.

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                 Security Layers                                                       │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                           Application Layer                            │ │
│  ├─────────────────────────────────────────────────────────────────────────┤ │
│  │ • Input Validation        • Authentication        • Authorization      │ │
│  │ • SQL Injection Prevention • XSS Protection      • CSRF Protection    │ │
│  │ • Rate Limiting          • Audit Logging         • Error Handling     │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                           Network Layer                                │ │
│  ├─────────────────────────────────────────────────────────────────────────┤ │
│  │ • TLS/SSL Encryption    • VPN Access        • Firewall Rules         │ │
│  │ • Network Segmentation  • Intrusion Detection • DDoS Protection       │ │
│  │ • Load Balancer SSL     • API Gateway       • WAF (Web App Firewall) │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                           Database Layer                               │ │
│  ├─────────────────────────────────────────────────────────────────────────┤ │
│  │ • Row-Level Security    • Column Encryption • Transparent Encryption  │ │
│  │ • Audit Triggers        • Data Masking      • Backup Encryption       │ │
│  │ • Connection Encryption • Query Logging     • Access Controls         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                           Storage Layer                                │ │
│  ├─────────────────────────────────────────────────────────────────────────┤ │
│  │ • Disk Encryption       • Backup Encryption • Key Management          │ │
│  │ • Access Logging        • Data Retention    • Secure Deletion         │ │
│  │ • Compliance Monitoring • Audit Trails      • Incident Response       │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Row-Level Security (RLS)

### User-Based RLS Policies

```sql
-- Enable RLS on tables
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Create policies for products table
CREATE POLICY "products_select_policy" ON products
    FOR SELECT USING (
        is_active = true OR 
        current_user = created_by OR 
        EXISTS (
            SELECT 1 FROM user_roles ur 
            JOIN users u ON ur.user_id = u.user_id 
            WHERE u.okta_user_id = current_user 
            AND ur.role_name IN ('ProductAdministrator', 'SystemAdministrator')
        )
    );

CREATE POLICY "products_insert_policy" ON products
    FOR INSERT WITH CHECK (
        EXISTS (
            SELECT 1 FROM user_roles ur 
            JOIN users u ON ur.user_id = u.user_id 
            WHERE u.okta_user_id = current_user 
            AND ur.role_name IN ('ProductAdministrator', 'SystemAdministrator')
        )
    );

CREATE POLICY "products_update_policy" ON products
    FOR UPDATE USING (
        current_user = created_by OR 
        EXISTS (
            SELECT 1 FROM user_roles ur 
            JOIN users u ON ur.user_id = u.user_id 
            WHERE u.okta_user_id = current_user 
            AND ur.role_name IN ('ProductAdministrator', 'SystemAdministrator')
        )
    );

-- Create policies for audit_log table
CREATE POLICY "audit_log_select_policy" ON audit_log
    FOR SELECT USING (
        user_id = (
            SELECT user_id FROM users WHERE okta_user_id = current_user
        ) OR 
        EXISTS (
            SELECT 1 FROM user_roles ur 
            JOIN users u ON ur.user_id = u.user_id 
            WHERE u.okta_user_id = current_user 
            AND ur.role_name IN ('SystemAdministrator')
        )
    );

-- Create policies for users table
CREATE POLICY "users_select_policy" ON users
    FOR SELECT USING (
        user_id = (
            SELECT user_id FROM users WHERE okta_user_id = current_user
        ) OR 
        EXISTS (
            SELECT 1 FROM user_roles ur 
            JOIN users u ON ur.user_id = u.user_id 
            WHERE u.okta_user_id = current_user 
            AND ur.role_name IN ('SystemAdministrator')
        )
    );
```

### Role-Based Access Control

```sql
-- Create role-based access functions
CREATE OR REPLACE FUNCTION get_user_roles(p_user_id UUID)
RETURNS TABLE(role_name VARCHAR(100)) AS $$
BEGIN
    RETURN QUERY
    SELECT ur.role_name
    FROM user_roles ur
    WHERE ur.user_id = p_user_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create function to check if user has specific role
CREATE OR REPLACE FUNCTION has_role(p_role_name VARCHAR(100))
RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1 
        FROM user_roles ur
        JOIN users u ON ur.user_id = u.user_id
        WHERE u.okta_user_id = current_user
        AND ur.role_name = p_role_name
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create function to check if user has any of the specified roles
CREATE OR REPLACE FUNCTION has_any_role(p_role_names VARCHAR(100)[])
RETURNS BOOLEAN AS $$
BEGIN
    RETURN EXISTS (
        SELECT 1 
        FROM user_roles ur
        JOIN users u ON ur.user_id = u.user_id
        WHERE u.okta_user_id = current_user
        AND ur.role_name = ANY(p_role_names)
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Column-Level Encryption

### Sensitive Data Encryption

```sql
-- Create encryption extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Create encrypted columns for sensitive data
ALTER TABLE users ADD COLUMN encrypted_ssn BYTEA;
ALTER TABLE users ADD COLUMN encrypted_tax_id BYTEA;

-- Create function to encrypt sensitive data
CREATE OR REPLACE FUNCTION encrypt_sensitive_data(
    p_data TEXT,
    p_key TEXT DEFAULT current_setting('app.encryption_key')
)
RETURNS BYTEA AS $$
BEGIN
    RETURN pgp_sym_encrypt(p_data, p_key);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create function to decrypt sensitive data
CREATE OR REPLACE FUNCTION decrypt_sensitive_data(
    p_encrypted_data BYTEA,
    p_key TEXT DEFAULT current_setting('app.encryption_key')
)
RETURNS TEXT AS $$
BEGIN
    RETURN pgp_sym_decrypt(p_encrypted_data, p_key);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create trigger to automatically encrypt sensitive data
CREATE OR REPLACE FUNCTION encrypt_user_sensitive_data()
RETURNS TRIGGER AS $$
BEGIN
    -- Encrypt SSN if provided
    IF NEW.ssn IS NOT NULL THEN
        NEW.encrypted_ssn := encrypt_sensitive_data(NEW.ssn);
        NEW.ssn := NULL; -- Clear plain text
    END IF;
    
    -- Encrypt tax ID if provided
    IF NEW.tax_id IS NOT NULL THEN
        NEW.encrypted_tax_id := encrypt_sensitive_data(NEW.tax_id);
        NEW.tax_id := NULL; -- Clear plain text
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_encrypt_user_data
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION encrypt_user_sensitive_data();
```

### Data Masking for Non-Production

```sql
-- Create data masking function for development/staging
CREATE OR REPLACE FUNCTION mask_sensitive_data(p_data TEXT)
RETURNS TEXT AS $$
BEGIN
    -- Mask email addresses
    IF p_data LIKE '%@%' THEN
        RETURN regexp_replace(p_data, '(.*)@(.*)', '***@\2');
    END IF;
    
    -- Mask phone numbers
    IF p_data ~ '^\d{10}$' THEN
        RETURN '***-***-' || substring(p_data from 8 for 4);
    END IF;
    
    -- Mask SSN
    IF p_data ~ '^\d{3}-\d{2}-\d{4}$' THEN
        RETURN '***-**-' || substring(p_data from 8 for 4);
    END IF;
    
    -- Default masking
    RETURN '***' || substring(p_data from length(p_data) - 2);
END;
$$ LANGUAGE plpgsql;

-- Create view for masked data in non-production
CREATE OR REPLACE VIEW users_masked AS
SELECT 
    user_id,
    okta_user_id,
    mask_sensitive_data(email) as email,
    first_name,
    last_name,
    is_active,
    last_login,
    created_at,
    updated_at
FROM users;
```

## Audit and Compliance

### Comprehensive Audit Triggers

```sql
-- Create audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
DECLARE
    v_old_data JSONB;
    v_new_data JSONB;
    v_user_id UUID;
    v_session_id VARCHAR(255);
BEGIN
    -- Get current user ID
    SELECT u.user_id INTO v_user_id
    FROM users u
    WHERE u.okta_user_id = current_user;
    
    -- Get session ID from application context
    v_session_id := current_setting('app.session_id', true);
    
    -- Prepare old and new data
    IF TG_OP = 'DELETE' THEN
        v_old_data = to_jsonb(OLD);
        v_new_data = NULL;
    ELSIF TG_OP = 'UPDATE' THEN
        v_old_data = to_jsonb(OLD);
        v_new_data = to_jsonb(NEW);
    ELSIF TG_OP = 'INSERT' THEN
        v_old_data = NULL;
        v_new_data = to_jsonb(NEW);
    END IF;
    
    -- Insert audit record
    INSERT INTO audit_log (
        audit_id,
        user_id,
        action_type,
        table_name,
        record_id,
        old_values,
        new_values,
        ip_address,
        user_agent,
        timestamp,
        session_id
    ) VALUES (
        gen_random_uuid(),
        v_user_id,
        TG_OP,
        TG_TABLE_NAME,
        CASE 
            WHEN TG_OP = 'DELETE' THEN OLD.user_id
            ELSE NEW.user_id
        END,
        v_old_data,
        v_new_data,
        inet_client_addr(),
        current_setting('app.user_agent', true),
        CURRENT_TIMESTAMP,
        v_session_id
    );
    
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Create audit triggers for all tables
CREATE TRIGGER audit_users_trigger
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER audit_products_trigger
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER audit_rate_history_trigger
    AFTER INSERT OR UPDATE OR DELETE ON rate_history
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER audit_user_roles_trigger
    AFTER INSERT OR UPDATE OR DELETE ON user_roles
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

### Compliance Reporting

```sql
-- Create compliance reporting functions
CREATE OR REPLACE FUNCTION get_audit_report(
    p_start_date TIMESTAMP DEFAULT CURRENT_DATE - INTERVAL '30 days',
    p_end_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
RETURNS TABLE(
    action_type VARCHAR(50),
    table_name VARCHAR(100),
    user_email VARCHAR(255),
    record_count BIGINT,
    last_action TIMESTAMP
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        al.action_type,
        al.table_name,
        u.email,
        COUNT(*) as record_count,
        MAX(al.timestamp) as last_action
    FROM audit_log al
    LEFT JOIN users u ON al.user_id = u.user_id
    WHERE al.timestamp BETWEEN p_start_date AND p_end_date
    GROUP BY al.action_type, al.table_name, u.email
    ORDER BY record_count DESC;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create function to check data access compliance
CREATE OR REPLACE FUNCTION check_data_access_compliance()
RETURNS TABLE(
    compliance_check VARCHAR(100),
    status VARCHAR(20),
    details TEXT
) AS $$
BEGIN
    -- Check for unauthorized access attempts
    RETURN QUERY
    SELECT 
        'Unauthorized Access Attempts'::VARCHAR(100) as compliance_check,
        CASE 
            WHEN COUNT(*) > 0 THEN 'FAIL'::VARCHAR(20)
            ELSE 'PASS'::VARCHAR(20)
        END as status,
        'Found ' || COUNT(*) || ' unauthorized access attempts'::TEXT as details
    FROM audit_log al
    WHERE al.action_type = 'SELECT'
    AND al.user_id IS NULL
    AND al.timestamp > CURRENT_DATE - INTERVAL '7 days';
    
    -- Check for data modification without proper authorization
    RETURN QUERY
    SELECT 
        'Unauthorized Data Modifications'::VARCHAR(100) as compliance_check,
        CASE 
            WHEN COUNT(*) > 0 THEN 'FAIL'::VARCHAR(20)
            ELSE 'PASS'::VARCHAR(20)
        END as status,
        'Found ' || COUNT(*) || ' unauthorized modifications'::TEXT as details
    FROM audit_log al
    WHERE al.action_type IN ('INSERT', 'UPDATE', 'DELETE')
    AND al.user_id IS NULL
    AND al.timestamp > CURRENT_DATE - INTERVAL '7 days';
    
    -- Check for sensitive data access
    RETURN QUERY
    SELECT 
        'Sensitive Data Access'::VARCHAR(100) as compliance_check,
        'INFO'::VARCHAR(20) as status,
        'Audit trail maintained for all sensitive data access'::TEXT as details;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Data Retention and PII Handling

### Data Retention Policies

```sql
-- Create data retention function
CREATE OR REPLACE FUNCTION cleanup_expired_data()
RETURNS INTEGER AS $$
DECLARE
    v_deleted_count INTEGER := 0;
BEGIN
    -- Clean up old audit logs (retain for 7 years)
    DELETE FROM audit_log 
    WHERE timestamp < CURRENT_DATE - INTERVAL '7 years';
    
    GET DIAGNOSTICS v_deleted_count = ROW_COUNT;
    
    -- Clean up old user sessions (retain for 1 year)
    DELETE FROM user_sessions 
    WHERE last_activity < CURRENT_DATE - INTERVAL '1 year';
    
    -- Clean up old rate history (retain for 3 years)
    DELETE FROM rate_history 
    WHERE effective_date < CURRENT_DATE - INTERVAL '3 years';
    
    -- Archive old data to separate table
    INSERT INTO audit_log_archive 
    SELECT * FROM audit_log 
    WHERE timestamp < CURRENT_DATE - INTERVAL '6 years';
    
    RETURN v_deleted_count;
END;
$$ LANGUAGE plpgsql;

-- Create scheduled job for data cleanup
SELECT cron.schedule(
    'cleanup-expired-data',
    '0 2 * * 0', -- Every Sunday at 2 AM
    'SELECT cleanup_expired_data();'
);
```

### PII Data Handling

```sql
-- Create PII data classification
CREATE TYPE pii_classification AS ENUM (
    'public',
    'internal',
    'confidential',
    'restricted'
);

-- Add PII classification to columns
ALTER TABLE users ADD COLUMN email_classification pii_classification DEFAULT 'confidential';
ALTER TABLE users ADD COLUMN phone_classification pii_classification DEFAULT 'confidential';

-- Create PII data access function
CREATE OR REPLACE FUNCTION get_pii_data(
    p_user_id UUID,
    p_required_classification pii_classification DEFAULT 'internal'
)
RETURNS TABLE(
    field_name VARCHAR(100),
    field_value TEXT,
    classification pii_classification
) AS $$
BEGIN
    -- Check if user has permission to access PII data
    IF NOT has_any_role(ARRAY['SystemAdministrator', 'ProductAdministrator']) THEN
        RAISE EXCEPTION 'Insufficient permissions to access PII data';
    END IF;
    
    RETURN QUERY
    SELECT 
        'email'::VARCHAR(100) as field_name,
        u.email as field_value,
        u.email_classification
    FROM users u
    WHERE u.user_id = p_user_id
    AND u.email_classification <= p_required_classification;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Security Monitoring

### Real-time Security Monitoring

```sql
-- Create security monitoring function
CREATE OR REPLACE FUNCTION monitor_security_events()
RETURNS TABLE(
    event_type VARCHAR(100),
    severity VARCHAR(20),
    event_count INTEGER,
    last_occurrence TIMESTAMP
) AS $$
BEGIN
    -- Monitor failed login attempts
    RETURN QUERY
    SELECT 
        'Failed Login Attempts'::VARCHAR(100) as event_type,
        CASE 
            WHEN COUNT(*) > 10 THEN 'HIGH'::VARCHAR(20)
            WHEN COUNT(*) > 5 THEN 'MEDIUM'::VARCHAR(20)
            ELSE 'LOW'::VARCHAR(20)
        END as severity,
        COUNT(*)::INTEGER as event_count,
        MAX(timestamp) as last_occurrence
    FROM audit_log al
    WHERE al.action_type = 'LOGIN_FAILED'
    AND al.timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour';
    
    -- Monitor unusual data access patterns
    RETURN QUERY
    SELECT 
        'Unusual Data Access'::VARCHAR(100) as event_type,
        CASE 
            WHEN COUNT(*) > 100 THEN 'HIGH'::VARCHAR(20)
            WHEN COUNT(*) > 50 THEN 'MEDIUM'::VARCHAR(20)
            ELSE 'LOW'::VARCHAR(20)
        END as severity,
        COUNT(*)::INTEGER as event_count,
        MAX(timestamp) as last_occurrence
    FROM audit_log al
    WHERE al.action_type = 'SELECT'
    AND al.timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
    GROUP BY al.user_id
    HAVING COUNT(*) > 20;
    
    -- Monitor data modification outside business hours
    RETURN QUERY
    SELECT 
        'After Hours Data Modification'::VARCHAR(100) as event_type,
        'MEDIUM'::VARCHAR(20) as severity,
        COUNT(*)::INTEGER as event_count,
        MAX(timestamp) as last_occurrence
    FROM audit_log al
    WHERE al.action_type IN ('INSERT', 'UPDATE', 'DELETE')
    AND EXTRACT(HOUR FROM al.timestamp) NOT BETWEEN 6 AND 22
    AND al.timestamp > CURRENT_DATE;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Security Alert System

```sql
-- Create security alert function
CREATE OR REPLACE FUNCTION create_security_alert(
    p_alert_type VARCHAR(100),
    p_severity VARCHAR(20),
    p_description TEXT,
    p_user_id UUID DEFAULT NULL
)
RETURNS UUID AS $$
DECLARE
    v_alert_id UUID;
BEGIN
    v_alert_id := gen_random_uuid();
    
    INSERT INTO security_alerts (
        alert_id,
        alert_type,
        severity,
        description,
        user_id,
        created_at,
        status
    ) VALUES (
        v_alert_id,
        p_alert_type,
        p_severity,
        p_description,
        p_user_id,
        CURRENT_TIMESTAMP,
        'OPEN'
    );
    
    -- Log the alert
    INSERT INTO audit_log (
        audit_id,
        user_id,
        action_type,
        table_name,
        record_id,
        new_values,
        timestamp
    ) VALUES (
        gen_random_uuid(),
        p_user_id,
        'SECURITY_ALERT',
        'security_alerts',
        v_alert_id,
        jsonb_build_object(
            'alert_type', p_alert_type,
            'severity', p_severity,
            'description', p_description
        ),
        CURRENT_TIMESTAMP
    );
    
    RETURN v_alert_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Backup and Recovery Security

### Encrypted Backup Strategy

```sql
-- Create encrypted backup function
CREATE OR REPLACE FUNCTION create_encrypted_backup(p_backup_path TEXT)
RETURNS TEXT AS $$
DECLARE
    v_backup_file TEXT;
    v_encrypted_file TEXT;
    v_backup_command TEXT;
BEGIN
    v_backup_file := p_backup_path || '/backup_' || to_char(CURRENT_TIMESTAMP, 'YYYYMMDD_HH24MISS') || '.sql';
    v_encrypted_file := v_backup_file || '.gpg';
    
    -- Create backup command
    v_backup_command := 'pg_dump -h ' || current_setting('app.db_host') || 
                       ' -U ' || current_setting('app.db_user') || 
                       ' -d ' || current_database() || 
                       ' --no-password > ' || v_backup_file;
    
    -- Execute backup
    PERFORM dblink_exec('host=' || current_setting('app.db_host'), v_backup_command);
    
    -- Encrypt backup file
    PERFORM dblink_exec('host=' || current_setting('app.db_host'), 
                       'gpg --encrypt --recipient ' || current_setting('app.gpg_recipient') || 
                       ' ' || v_backup_file);
    
    -- Remove unencrypted backup
    PERFORM dblink_exec('host=' || current_setting('app.db_host'), 
                       'rm ' || v_backup_file);
    
    RETURN v_encrypted_file;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

**Last Updated**: 2024-01-15
**Version**: 1.0.0
**Next Review**: 2024-04-15 