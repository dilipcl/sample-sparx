# Security Requirements, Logging, and Reconciliation Strategy

## Overview

This document establishes comprehensive security, audit logging, and data reconciliation standards for the gold layer in the SAS migration project. These controls ensure data is protected, access is auditable, and data accuracy is continuously validated.

## Security Architecture

### Security Principles

1. **Defense in Depth**: Multiple layers of security controls
2. **Least Privilege**: Users granted minimum access necessary
3. **Separation of Duties**: Segregation between data roles
4. **Data Classification**: Risk-based security controls
5. **Audit and Accountability**: Comprehensive logging of all access
6. **Privacy by Design**: Data protection embedded from inception

### Security Layers

```
┌─────────────────────────────────────────┐
│  Application Layer Security             │
│  - Application authentication           │
│  - Session management                   │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Network Layer Security                 │
│  - VPC/Network segmentation             │
│  - Firewall rules                       │
│  - Private endpoints                    │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Platform Layer Security                │
│  - Identity & Access Management (IAM)   │
│  - Service authentication               │
│  - Encryption in transit (TLS 1.3)      │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Database Layer Security                │
│  - Row-level security (RLS)             │
│  - Column-level security                │
│  - Dynamic data masking                 │
│  - Database roles and grants            │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Storage Layer Security                 │
│  - Encryption at rest (AES-256)         │
│  - Key management (KMS)                 │
│  - Immutable backups                    │
└─────────────────────────────────────────┘
```

## Data Classification Framework

### Classification Levels

| Level | Definition | Examples | Controls Required |
|-------|------------|----------|-------------------|
| **Public** | No confidentiality impact if disclosed | Anonymized statistics, public reports | Standard access controls |
| **Internal** | Moderate impact if disclosed | General business data, aggregated metrics | Authentication required, access logged |
| **Confidential** | Significant impact if disclosed | Customer data, financial details, employee records | RLS/CLS, encryption, audit logging |
| **Restricted** | Severe impact if disclosed | PII, PHI, PCI data, trade secrets | Strict RLS/CLS, masking, comprehensive auditing, DLP |

### Classification Process

```
FOR EACH gold layer table:
    
    1. Identify data elements
        ├── Personal identifiers (name, email, SSN, etc.)
        ├── Financial data (account numbers, balances, transactions)
        ├── Health information (diagnoses, treatments, claims)
        └── Proprietary information (pricing, margins, strategies)
    
    2. Determine highest classification
        → Table inherits highest classification of any column
    
    3. Document classification
        → Add classification to table metadata
        → Tag in data catalog
        → Configure access controls
    
    4. Review quarterly
        → Re-assess if data usage changes
        → Update controls as needed
```

**Metadata Example**:
```sql
COMMENT ON TABLE gold_sales_retail.fact_sales_transaction IS 
'...
SECURITY CLASSIFICATION: Confidential
SENSITIVE COLUMNS: customer_key, customer_email
COMPLIANCE REQUIREMENTS: CCPA, GDPR
ACCESS APPROVAL REQUIRED: Yes
DATA OWNER: VP Sales Operations
...';
```

## Access Control Framework

### Role-Based Access Control (RBAC)

**Standard Roles**:

```sql
-- Read-only analyst role
CREATE ROLE gold_sales_analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA gold_sales_retail TO gold_sales_analyst;
GRANT USAGE ON SCHEMA gold_sales_retail TO gold_sales_analyst;

-- Power user with limited write access
CREATE ROLE gold_sales_power_user;
GRANT SELECT, INSERT, UPDATE ON TABLE gold_sales_retail.metric_* TO gold_sales_power_user;
GRANT SELECT ON ALL TABLES IN SCHEMA gold_sales_retail TO gold_sales_power_user;

-- Data engineer with full access
CREATE ROLE gold_data_engineer;
GRANT ALL PRIVILEGES ON SCHEMA gold_sales_retail TO gold_data_engineer;

-- Service account for ETL processes
CREATE ROLE gold_etl_service_account;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA gold_sales_retail TO gold_etl_service_account;
GRANT USAGE, CREATE ON SCHEMA gold_sales_retail TO gold_etl_service_account;
```

### Attribute-Based Access Control (ABAC)

**Policy Examples**:

```sql
-- Sales users can only see their region's data
CREATE POLICY sales_region_policy ON gold_sales_retail.fact_sales_transaction
FOR SELECT
USING (
    store_region IN (
        SELECT region FROM user_attributes 
        WHERE username = CURRENT_USER
    )
);

-- Managers can see their team's data
CREATE POLICY manager_hierarchy_policy ON gold_hr_compensation.fact_salary
FOR SELECT
USING (
    employee_key IN (
        SELECT employee_key FROM employee_hierarchy
        WHERE manager_username = CURRENT_USER
    )
);

-- Time-based access control
CREATE POLICY historical_data_policy ON gold_finance_accounting.fact_gl_transaction
FOR SELECT
USING (
    CASE 
        WHEN HAS_ROLE('finance_analyst') THEN transaction_date >= CURRENT_DATE - INTERVAL '3 years'
        WHEN HAS_ROLE('finance_manager') THEN transaction_date >= CURRENT_DATE - INTERVAL '7 years'
        WHEN HAS_ROLE('compliance_officer') THEN TRUE
        ELSE FALSE
    END
);
```

### Row-Level Security (RLS) Implementation

**Standard RLS Pattern**:

```sql
-- 1. Add security column to table
ALTER TABLE gold_customer_360.dim_customer 
ADD COLUMN accessible_to TEXT[];

-- 2. Populate security attributes
UPDATE gold_customer_360.dim_customer
SET accessible_to = ARRAY['customer_service', 'sales_team', 'marketing_team']
WHERE customer_segment = 'Standard';

UPDATE gold_customer_360.dim_customer
SET accessible_to = ARRAY['sales_team', 'account_management', 'executives']
WHERE customer_segment = 'Enterprise';

-- 3. Create RLS policy
CREATE POLICY customer_access_policy ON gold_customer_360.dim_customer
FOR SELECT
USING (
    -- Admins can see all
    HAS_ROLE('gold_admin') 
    OR
    -- Others see based on role membership
    EXISTS (
        SELECT 1 FROM user_roles ur
        WHERE ur.username = CURRENT_USER
        AND ur.role_name = ANY(accessible_to)
    )
);

-- 4. Enable RLS on table
ALTER TABLE gold_customer_360.dim_customer ENABLE ROW LEVEL SECURITY;
```

**Dynamic RLS with Context**:

```sql
-- Using session context for RLS
CREATE POLICY dynamic_sales_territory_policy 
ON gold_sales_retail.fact_sales_transaction
FOR SELECT
USING (
    sales_territory_code = CURRENT_SETTING('app.user_territory', true)
    OR
    HAS_ROLE('sales_director')  -- Directors see all territories
);

-- Application sets context on connection
-- SET app.user_territory = 'WEST';
```

### Column-Level Security (CLS)

**Sensitive Column Protection**:

```sql
-- Restrict access to sensitive columns
REVOKE SELECT ON gold_customer_360.dim_customer(
    customer_ssn,
    customer_credit_score,
    customer_income_range
) FROM PUBLIC;

-- Grant only to authorized roles
GRANT SELECT(customer_ssn) ON gold_customer_360.dim_customer 
TO compliance_team, finance_manager;

-- Create secured view for general access
CREATE VIEW gold_customer_360.v_customer_public AS
SELECT 
    customer_key,
    customer_id,
    customer_name,
    customer_segment,
    -- Exclude sensitive columns
    -- customer_ssn,           -- Excluded
    -- customer_credit_score,  -- Excluded
    -- customer_income_range,  -- Excluded
    effective_date,
    is_current
FROM gold_customer_360.dim_customer;

GRANT SELECT ON gold_customer_360.v_customer_public TO gold_sales_analyst;
```

## Data Masking and Anonymization

### Dynamic Data Masking

**Masking Strategies by Data Type**:

```sql
-- Create masking function for PII
CREATE OR REPLACE FUNCTION mask_pii(
    value TEXT,
    user_role TEXT,
    mask_type TEXT DEFAULT 'partial'
)
RETURNS TEXT AS $$
BEGIN
    -- Full access for privileged roles
    IF user_role IN ('compliance_officer', 'data_steward') THEN
        RETURN value;
    END IF;
    
    -- Masking based on type
    CASE mask_type
        WHEN 'full' THEN
            RETURN '***MASKED***';
        WHEN 'partial' THEN
            -- Show last 4 characters
            RETURN REPEAT('*', LENGTH(value) - 4) || RIGHT(value, 4);
        WHEN 'email' THEN
            -- Mask email: j***@example.com
            RETURN LEFT(value, 1) || '***@' || SPLIT_PART(value, '@', 2);
        WHEN 'phone' THEN
            -- Mask phone: ***-***-1234
            RETURN '***-***-' || RIGHT(REGEXP_REPLACE(value, '[^0-9]', ''), 4);
        ELSE
            RETURN NULL;
    END CASE;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Apply masking in view
CREATE VIEW gold_customer_360.v_customer_masked AS
SELECT 
    customer_key,
    customer_id,
    customer_name,
    -- Dynamically mask based on current user role
    mask_pii(
        customer_email, 
        (SELECT role_name FROM user_roles WHERE username = CURRENT_USER LIMIT 1),
        'email'
    ) AS customer_email,
    mask_pii(
        customer_phone,
        (SELECT role_name FROM user_roles WHERE username = CURRENT_USER LIMIT 1),
        'phone'
    ) AS customer_phone,
    customer_segment
FROM gold_customer_360.dim_customer
WHERE is_current = TRUE;
```

### Tokenization

**For Highly Sensitive Data**:

```sql
-- Separate token vault (access tightly controlled)
CREATE TABLE security.token_vault (
    token_id VARCHAR(64) PRIMARY KEY,
    original_value TEXT NOT NULL,
    data_type VARCHAR(50),
    created_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    accessed_count INT DEFAULT 0,
    last_accessed_timestamp TIMESTAMP
);

-- Encrypt at rest
ALTER TABLE security.token_vault 
ENABLE ENCRYPTION 
WITH (ENCRYPTION_TYPE = 'AES256', KEY = 'customer_pii_key');

-- Gold layer stores tokens only
CREATE TABLE gold_customer_360.dim_customer (
    customer_key BIGINT PRIMARY KEY,
    customer_id VARCHAR(50),
    customer_name VARCHAR(200),
    customer_ssn_token VARCHAR(64),  -- Token reference, not actual SSN
    customer_email_token VARCHAR(64),
    -- Other attributes
    ...
);

-- Detokenization function (restricted access)
CREATE OR REPLACE FUNCTION detokenize(token VARCHAR(64))
RETURNS TEXT AS $$
DECLARE
    original_value TEXT;
BEGIN
    -- Verify caller has detokenization privilege
    IF NOT HAS_PRIVILEGE(CURRENT_USER, 'security.token_vault', 'SELECT') THEN
        RAISE EXCEPTION 'Insufficient privileges to detokenize';
    END IF;
    
    -- Log access
    UPDATE security.token_vault
    SET accessed_count = accessed_count + 1,
        last_accessed_timestamp = CURRENT_TIMESTAMP
    WHERE token_id = token;
    
    -- Return original value
    SELECT original_value INTO original_value
    FROM security.token_vault
    WHERE token_id = token;
    
    RETURN original_value;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Audit Logging Framework

### Logging Principles

1. **Comprehensive**: Log all data access and modifications
2. **Tamper-Proof**: Logs stored in append-only, immutable storage
3. **Retention**: Maintain logs per compliance requirements (typically 7 years)
4. **Monitoring**: Active monitoring and alerting on suspicious activity
5. **Privacy**: Balance audit needs with user privacy

### Logging Architecture

```
┌─────────────────────────────────────────┐
│  Application Layer                      │
│  - User actions and queries             │
└─────────────────────────────────────────┘
              ↓ (logs emitted)
┌─────────────────────────────────────────┐
│  Database Query Logs                    │
│  - All SELECT, INSERT, UPDATE, DELETE   │
│  - User, timestamp, query text          │
└─────────────────────────────────────────┘
              ↓ (streamed)
┌─────────────────────────────────────────┐
│  Centralized Log Collection             │
│  - Log aggregation service              │
│  - Structured logging (JSON)            │
└─────────────────────────────────────────┘
              ↓ (indexed)
┌─────────────────────────────────────────┐
│  Audit Data Store                       │
│  - Append-only storage                  │
│  - Long-term retention                  │
│  - Compliance-ready                     │
└─────────────────────────────────────────┘
              ↓ (analyzed)
┌─────────────────────────────────────────┐
│  Security Monitoring & Alerting         │
│  - Anomaly detection                    │
│  - Real-time alerts                     │
│  - Compliance reporting                 │
└─────────────────────────────────────────┘
```

### Audit Log Types

#### 1. Data Access Logs

**What to Log**:
- User identity (username, role)
- Timestamp (UTC)
- Action (SELECT, INSERT, UPDATE, DELETE)
- Table/schema accessed
- Row count (for SELECT/UPDATE/DELETE)
- Query execution time
- Source IP address
- Application/tool used
- Session identifier

**Implementation**:
```sql
-- Create audit log table
CREATE TABLE audit.data_access_log (
    log_id BIGSERIAL PRIMARY KEY,
    event_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    username VARCHAR(100) NOT NULL,
    user_role VARCHAR(100),
    session_id VARCHAR(100),
    action_type VARCHAR(20) NOT NULL, -- SELECT, INSERT, UPDATE, DELETE
    schema_name VARCHAR(100) NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    row_count INT,
    query_text TEXT,
    execution_time_ms INT,
    source_ip VARCHAR(50),
    application_name VARCHAR(100),
    success BOOLEAN NOT NULL,
    error_message TEXT
);

-- Index for common queries
CREATE INDEX ix_data_access_log_timestamp ON audit.data_access_log(event_timestamp);
CREATE INDEX ix_data_access_log_username ON audit.data_access_log(username);
CREATE INDEX ix_data_access_log_table ON audit.data_access_log(schema_name, table_name);

-- Trigger for automatic logging
CREATE OR REPLACE FUNCTION audit.log_data_access()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit.data_access_log (
        username,
        user_role,
        action_type,
        schema_name,
        table_name,
        row_count
    ) VALUES (
        CURRENT_USER,
        CURRENT_ROLE,
        TG_OP,
        TG_TABLE_SCHEMA,
        TG_TABLE_NAME,
        CASE 
            WHEN TG_OP = 'DELETE' THEN 1
            WHEN TG_OP = 'INSERT' THEN 1
            WHEN TG_OP = 'UPDATE' THEN 1
            ELSE 0
        END
    );
    
    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Apply to sensitive tables
CREATE TRIGGER audit_customer_access
AFTER INSERT OR UPDATE OR DELETE ON gold_customer_360.dim_customer
FOR EACH ROW EXECUTE FUNCTION audit.log_data_access();
```

#### 2. Schema Change Logs

**What to Log**:
- DDL operations (CREATE, ALTER, DROP)
- Schema/object name
- Change details
- User and timestamp
- Change request/ticket reference

**Implementation**:
```sql
CREATE TABLE audit.schema_change_log (
    log_id BIGSERIAL PRIMARY KEY,
    event_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    username VARCHAR(100) NOT NULL,
    ddl_operation VARCHAR(50) NOT NULL,
    object_type VARCHAR(50) NOT NULL, -- TABLE, VIEW, INDEX, etc.
    schema_name VARCHAR(100),
    object_name VARCHAR(100),
    ddl_statement TEXT,
    change_ticket_id VARCHAR(50),
    approval_status VARCHAR(20),
    approver VARCHAR(100)
);

-- Event trigger for DDL auditing
CREATE OR REPLACE FUNCTION audit.log_schema_change()
RETURNS event_trigger AS $$
DECLARE
    ddl_info RECORD;
BEGIN
    FOR ddl_info IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        INSERT INTO audit.schema_change_log (
            username,
            ddl_operation,
            object_type,
            schema_name,
            object_name,
            ddl_statement
        ) VALUES (
            CURRENT_USER,
            ddl_info.command_tag,
            ddl_info.object_type,
            ddl_info.schema_name,
            ddl_info.object_identity,
            current_query()
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE EVENT TRIGGER schema_change_audit 
ON ddl_command_end
EXECUTE FUNCTION audit.log_schema_change();
```

#### 3. Data Lineage Logs

**What to Log**:
- ETL job execution
- Source to target mapping
- Transformation applied
- Record counts (input/output)
- Data quality results
- Job duration and status

**Implementation**:
```sql
CREATE TABLE audit.etl_execution_log (
    execution_id BIGSERIAL PRIMARY KEY,
    job_name VARCHAR(200) NOT NULL,
    job_type VARCHAR(50), -- FULL_LOAD, INCREMENTAL, AGGREGATION
    start_timestamp TIMESTAMP NOT NULL,
    end_timestamp TIMESTAMP,
    status VARCHAR(20), -- RUNNING, SUCCEEDED, FAILED, CANCELLED
    source_system VARCHAR(100),
    source_table VARCHAR(200),
    target_schema VARCHAR(100),
    target_table VARCHAR(200),
    records_read BIGINT,
    records_written BIGINT,
    records_updated BIGINT,
    records_deleted BIGINT,
    records_rejected BIGINT,
    data_quality_score DECIMAL(5,2),
    error_message TEXT,
    executed_by VARCHAR(100)
);

CREATE TABLE audit.etl_execution_detail (
    detail_id BIGSERIAL PRIMARY KEY,
    execution_id BIGINT REFERENCES audit.etl_execution_log(execution_id),
    step_number INT,
    step_name VARCHAR(200),
    step_start_timestamp TIMESTAMP,
    step_end_timestamp TIMESTAMP,
    step_status VARCHAR(20),
    records_processed BIGINT,
    step_message TEXT
);

-- Log ETL execution
CREATE OR REPLACE PROCEDURE log_etl_execution(
    p_job_name VARCHAR,
    p_source_table VARCHAR,
    p_target_table VARCHAR,
    p_records_written BIGINT,
    p_status VARCHAR
) AS $$
BEGIN
    INSERT INTO audit.etl_execution_log (
        job_name,
        start_timestamp,
        end_timestamp,
        status,
        source_table,
        target_table,
        records_written,
        executed_by
    ) VALUES (
        p_job_name,
        CURRENT_TIMESTAMP - INTERVAL '5 minutes', -- Approximate start
        CURRENT_TIMESTAMP,
        p_status,
        p_source_table,
        p_target_table,
        p_records_written,
        CURRENT_USER
    );
END;
$$ LANGUAGE plpgsql;
```

### Security Event Monitoring

**Alerts and Thresholds**:

```sql
-- Monitor for suspicious access patterns
CREATE VIEW audit.v_suspicious_activity AS
SELECT 
    username,
    DATE_TRUNC('hour', event_timestamp) AS event_hour,
    COUNT(*) AS query_count,
    COUNT(DISTINCT table_name) AS distinct_tables_accessed,
    SUM(row_count) AS total_rows_accessed,
    COUNT(CASE WHEN success = FALSE THEN 1 END) AS failed_attempts
FROM audit.data_access_log
WHERE event_timestamp >= CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY username, DATE_TRUNC('hour', event_timestamp)
HAVING 
    COUNT(*) > 1000  -- More than 1000 queries per hour
    OR COUNT(DISTINCT table_name) > 50  -- Accessing too many tables
    OR SUM(row_count) > 10000000  -- Extracting too much data
    OR COUNT(CASE WHEN success = FALSE THEN 1 END) > 100; -- Many failures

-- Monitor for after-hours access
CREATE VIEW audit.v_after_hours_access AS
SELECT 
    username,
    schema_name,
    table_name,
    event_timestamp,
    action_type,
    row_count
FROM audit.data_access_log
WHERE 
    EXTRACT(HOUR FROM event_timestamp) NOT BETWEEN 7 AND 19  -- Outside 7 AM - 7 PM
    AND schema_name LIKE 'gold_%'
    AND event_timestamp >= CURRENT_DATE - INTERVAL '7 days';

-- Monitor for privilege escalation attempts
CREATE VIEW audit.v_privilege_escalation_attempts AS
SELECT 
    username,
    event_timestamp,
    ddl_statement
FROM audit.schema_change_log
WHERE 
    ddl_statement ILIKE '%GRANT%'
    OR ddl_statement ILIKE '%CREATE ROLE%'
    OR ddl_statement ILIKE '%ALTER USER%'
ORDER BY event_timestamp DESC;
```

## Data Reconciliation Strategy

### Reconciliation Principles

1. **Continuous Validation**: Reconcile throughout migration, not just at end
2. **Layered Approach**: Multiple levels of checks (counts, aggregates, details)
3. **Automated**: Minimize manual reconciliation effort
4. **Exception Management**: Clear process for investigating discrepancies
5. **Documentation**: Record all reconciliation results and resolutions

### Reconciliation Levels

#### Level 1: Record Count Reconciliation

**Purpose**: Quick validation that no data is lost or duplicated

```sql
CREATE TABLE reconciliation.record_count_check (
    check_id BIGSERIAL PRIMARY KEY,
    check_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    source_system VARCHAR(100),
    source_table VARCHAR(200),
    source_count BIGINT,
    target_schema VARCHAR(100),
    target_table VARCHAR(200),
    target_count BIGINT,
    variance BIGINT,
    variance_percent DECIMAL(7,4),
    status VARCHAR(20), -- PASS, FAIL, REVIEW
    notes TEXT
);

-- Reconciliation procedure
CREATE OR REPLACE PROCEDURE reconcile_record_counts(
    p_source_table VARCHAR,
    p_target_table VARCHAR,
    p_tolerance_percent DECIMAL DEFAULT 0.01
) AS $$
DECLARE
    v_source_count BIGINT;
    v_target_count BIGINT;
    v_variance BIGINT;
    v_variance_percent DECIMAL;
    v_status VARCHAR;
BEGIN
    -- Get source count (from SAS-equivalent staging table)
    EXECUTE format('SELECT COUNT(*) FROM %I', p_source_table) 
    INTO v_source_count;
    
    -- Get target count
    EXECUTE format('SELECT COUNT(*) FROM %I', p_target_table) 
    INTO v_target_count;
    
    -- Calculate variance
    v_variance := v_target_count - v_source_count;
    v_variance_percent := ABS(v_variance::DECIMAL / NULLIF(v_source_count, 0)) * 100;
    
    -- Determine status
    IF v_variance_percent <= p_tolerance_percent THEN
        v_status := 'PASS';
    ELSIF v_variance_percent <= p_tolerance_percent * 5 THEN
        v_status := 'REVIEW';
    ELSE
        v_status := 'FAIL';
    END IF;
    
    -- Log result
    INSERT INTO reconciliation.record_count_check (
        source_table,
        source_count,
        target_table,
        target_count,
        variance,
        variance_percent,
        status
    ) VALUES (
        p_source_table,
        v_source_count,
        p_target_table,
        v_target_count,
        v_variance,
        v_variance_percent,
        v_status
    );
    
    -- Raise alert if failed
    IF v_status = 'FAIL' THEN
        RAISE NOTICE 'Reconciliation FAILED for %: variance %', p_target_table, v_variance_percent || '%';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

#### Level 2: Aggregate Reconciliation

**Purpose**: Validate business metrics match between SAS and gold layer

```sql
CREATE TABLE reconciliation.aggregate_check (
    check_id BIGSERIAL PRIMARY KEY,
    check_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    check_name VARCHAR(200),
    source_table VARCHAR(200),
    target_table VARCHAR(200),
    metric_name VARCHAR(100),
    aggregation_type VARCHAR(50), -- SUM, AVG, MIN, MAX, COUNT
    dimension_values TEXT, -- JSON or delimited list of dimension values
    source_value DECIMAL(28,6),
    target_value DECIMAL(28,6),
    variance DECIMAL(28,6),
    variance_percent DECIMAL(7,4),
    status VARCHAR(20),
    notes TEXT
);

-- Example aggregate reconciliation
CREATE OR REPLACE PROCEDURE reconcile_sales_aggregates()
AS $$
DECLARE
    v_source_total DECIMAL(28,6);
    v_target_total DECIMAL(28,6);
BEGIN
    -- Total sales amount
    SELECT SUM(net_amount) INTO v_source_total
    FROM staging.sas_sales_transaction;
    
    SELECT SUM(net_amount) INTO v_target_total
    FROM gold_sales_retail.fact_sales_transaction;
    
    INSERT INTO reconciliation.aggregate_check (
        check_name,
        source_table,
        target_table,
        metric_name,
        aggregation_type,
        source_value,
        target_value,
        variance,
        variance_percent,
        status
    ) VALUES (
        'Total Sales Amount',
        'staging.sas_sales_transaction',
        'gold_sales_retail.fact_sales_transaction',
        'net_amount',
        'SUM',
        v_source_total,
        v_target_total,
        v_target_total - v_source_total,
        ABS((v_target_total - v_source_total) / NULLIF(v_source_total, 0)) * 100,
        CASE 
            WHEN ABS((v_target_total - v_source_total) / NULLIF(v_source_total, 0)) * 100 < 0.01 THEN 'PASS'
            WHEN ABS((v_target_total - v_source_total) / NULLIF(v_source_total, 0)) * 100 < 0.1 THEN 'REVIEW'
            ELSE 'FAIL'
        END
    );
    
    -- Aggregates by dimension (e.g., by product)
    INSERT INTO reconciliation.aggregate_check (
        check_name, source_table, target_table, metric_name, 
        aggregation_type, dimension_values, source_value, target_value,
        variance, variance_percent, status
    )
    SELECT 
        'Sales by Product: ' || COALESCE(s.product_id, t.product_id),
        'staging.sas_sales_transaction',
        'gold_sales_retail.fact_sales_transaction',
        'net_amount',
        'SUM',
        COALESCE(s.product_id, t.product_id),
        COALESCE(s.total_sales, 0),
        COALESCE(t.total_sales, 0),
        COALESCE(t.total_sales, 0) - COALESCE(s.total_sales, 0),
        ABS((COALESCE(t.total_sales, 0) - COALESCE(s.total_sales, 0)) / NULLIF(s.total_sales, 0)) * 100,
        CASE 
            WHEN ABS((COALESCE(t.total_sales, 0) - COALESCE(s.total_sales, 0)) / NULLIF(s.total_sales, 0)) * 100 < 0.01 THEN 'PASS'
            WHEN ABS((COALESCE(t.total_sales, 0) - COALESCE(s.total_sales, 0)) / NULLIF(s.total_sales, 0)) * 100 < 0.1 THEN 'REVIEW'
            ELSE 'FAIL'
        END
    FROM (
        SELECT product_id, SUM(net_amount) AS total_sales
        FROM staging.sas_sales_transaction
        GROUP BY product_id
    ) s
    FULL OUTER JOIN (
        SELECT p.product_id, SUM(f.net_amount) AS total_sales
        FROM gold_sales_retail.fact_sales_transaction f
        JOIN gold_product_catalog.dim_product p ON f.product_key = p.product_key
        GROUP BY p.product_id
    ) t ON s.product_id = t.product_id;
    
END;
$$ LANGUAGE plpgsql;
```

#### Level 3: Detailed Record Reconciliation

**Purpose**: Row-by-row comparison for critical datasets

```sql
CREATE TABLE reconciliation.detail_discrepancy (
    discrepancy_id BIGSERIAL PRIMARY KEY,
    check_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    source_table VARCHAR(200),
    target_table VARCHAR(200),
    business_key VARCHAR(200),
    discrepancy_type VARCHAR(50), -- MISSING_IN_TARGET, MISSING_IN_SOURCE, VALUE_MISMATCH
    column_name VARCHAR(100),
    source_value TEXT,
    target_value TEXT,
    resolution_status VARCHAR(50), -- OPEN, INVESTIGATING, RESOLVED, ACCEPTED
    resolution_notes TEXT,
    resolved_by VARCHAR(100),
    resolved_timestamp TIMESTAMP
);

-- Detailed reconciliation for fact table
CREATE OR REPLACE PROCEDURE reconcile_sales_details()
AS $$
BEGIN
    -- Find missing records in target
    INSERT INTO reconciliation.detail_discrepancy (
        source_table, target_table, business_key, discrepancy_type, resolution_status
    )
    SELECT 
        'staging.sas_sales_transaction',
        'gold_sales_retail.fact_sales_transaction',
        s.transaction_id,
        'MISSING_IN_TARGET',
        'OPEN'
    FROM staging.sas_sales_transaction s
    LEFT JOIN gold_sales_retail.fact_sales_transaction t ON s.transaction_id = t.transaction_id
    WHERE t.transaction_id IS NULL;
    
    -- Find extra records in target
    INSERT INTO reconciliation.detail_discrepancy (
        source_table, target_table, business_key, discrepancy_type, resolution_status
    )
    SELECT 
        'staging.sas_sales_transaction',
        'gold_sales_retail.fact_sales_transaction',
        t.transaction_id,
        'MISSING_IN_SOURCE',
        'OPEN'
    FROM gold_sales_retail.fact_sales_transaction t
    LEFT JOIN staging.sas_sales_transaction s ON s.transaction_id = t.transaction_id
    WHERE s.transaction_id IS NULL;
    
    -- Find value mismatches in key measures
    INSERT INTO reconciliation.detail_discrepancy (
        source_table, target_table, business_key, discrepancy_type, 
        column_name, source_value, target_value, resolution_status
    )
    SELECT 
        'staging.sas_sales_transaction',
        'gold_sales_retail.fact_sales_transaction',
        s.transaction_id,
        'VALUE_MISMATCH',
        'net_amount',
        s.net_amount::TEXT,
        t.net_amount::TEXT,
        'OPEN'
    FROM staging.sas_sales_transaction s
    JOIN gold_sales_retail.fact_sales_transaction t ON s.transaction_id = t.transaction_id
    WHERE ABS(s.net_amount - t.net_amount) > 0.01; -- Tolerance for rounding
    
    -- Similar checks for other critical columns...
END;
$$ LANGUAGE plpgsql;
```

### Continuous Reconciliation

**Ongoing Monitoring**:

```sql
-- Daily reconciliation job
CREATE OR REPLACE PROCEDURE daily_reconciliation_suite()
AS $$
BEGIN
    -- Level 1: Record counts
    CALL reconcile_record_counts('staging.sas_sales_transaction', 'gold_sales_retail.fact_sales_transaction');
    CALL reconcile_record_counts('staging.sas_customer', 'gold_customer_360.dim_customer');
    
    -- Level 2: Aggregates
    CALL reconcile_sales_aggregates();
    
    -- Level 3: Details (sample-based for large tables)
    CALL reconcile_sales_details_sample(0.1); -- 10% sample
    
    -- Generate reconciliation report
    CALL generate_reconciliation_report();
    
    -- Send alerts for failures
    CALL alert_on_reconciliation_failures();
END;
$$ LANGUAGE plpgsql;

-- Schedule daily execution
-- (Using your orchestration tool: Airflow, Azure Data Factory, etc.)
```

### Reconciliation Reporting

**Dashboard Metrics**:

```sql
-- Reconciliation summary view
CREATE VIEW reconciliation.v_daily_summary AS
SELECT 
    CURRENT_DATE AS report_date,
    COUNT(*) AS total_checks,
    COUNT(CASE WHEN status = 'PASS' THEN 1 END) AS passed_checks,
    COUNT(CASE WHEN status = 'REVIEW' THEN 1 END) AS review_checks,
    COUNT(CASE WHEN status = 'FAIL' THEN 1 END) AS failed_checks,
    ROUND(COUNT(CASE WHEN status = 'PASS' THEN 1 END) * 100.0 / COUNT(*), 2) AS pass_rate_percent
FROM (
    SELECT status FROM reconciliation.record_count_check WHERE check_timestamp::DATE = CURRENT_DATE
    UNION ALL
    SELECT status FROM reconciliation.aggregate_check WHERE check_timestamp::DATE = CURRENT_DATE
) checks;

-- Trend analysis
CREATE VIEW reconciliation.v_weekly_trend AS
SELECT 
    DATE_TRUNC('week', check_timestamp)::DATE AS week_start,
    target_table,
    COUNT(*) AS check_count,
    AVG(CASE WHEN status = 'PASS' THEN 1.0 ELSE 0.0 END) AS pass_rate,
    AVG(ABS(variance_percent)) AS avg_variance_percent
FROM reconciliation.aggregate_check
WHERE check_timestamp >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY DATE_TRUNC('week', check_timestamp)::DATE, target_table
ORDER BY week_start DESC, target_table;
```

## Compliance and Regulatory Requirements

### GDPR Compliance

**Right to Access**:
```sql
-- Procedure to extract all data for a customer
CREATE OR REPLACE FUNCTION gdpr_customer_data_export(p_customer_id VARCHAR)
RETURNS TABLE (
    table_name VARCHAR,
    data_json JSON
) AS $$
BEGIN
    RETURN QUERY
    SELECT 'dim_customer'::VARCHAR, row_to_json(c)
    FROM gold_customer_360.dim_customer c
    WHERE customer_id = p_customer_id AND is_current = TRUE;
    
    RETURN QUERY
    SELECT 'fact_sales_transaction'::VARCHAR, row_to_json(f)
    FROM gold_sales_retail.fact_sales_transaction f
    JOIN gold_customer_360.dim_customer c ON f.customer_key = c.customer_key
    WHERE c.customer_id = p_customer_id;
    
    -- Additional tables as needed...
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**Right to Be Forgotten**:
```sql
-- Procedure to anonymize customer data
CREATE OR REPLACE PROCEDURE gdpr_anonymize_customer(p_customer_id VARCHAR)
AS $$
BEGIN
    -- Update dimension with anonymized values
    UPDATE gold_customer_360.dim_customer
    SET 
        customer_name = 'ANONYMIZED',
        customer_email = 'anonymized@example.com',
        customer_phone = '000-000-0000',
        customer_ssn_token = NULL,
        -- Set expiration to mark as historical
        expiration_date = CURRENT_DATE,
        is_current = FALSE
    WHERE customer_id = p_customer_id AND is_current = TRUE;
    
    -- Log anonymization
    INSERT INTO audit.gdpr_anonymization_log (
        customer_id,
        anonymization_timestamp,
        anonymized_by,
        reason
    ) VALUES (
        p_customer_id,
        CURRENT_TIMESTAMP,
        CURRENT_USER,
        'GDPR Right to Be Forgotten request'
    );
END;
$$ LANGUAGE plpgsql;
```

### SOX Compliance

**Change Control**:
- All production changes require approval (logged in schema_change_log)
- Separation of duties: developers cannot deploy to production
- Audit trail of all data modifications

**Access Reviews**:
```sql
-- Quarterly access review report
CREATE VIEW audit.v_user_access_review AS
SELECT 
    u.username,
    u.email,
    r.role_name,
    r.granted_date,
    r.approved_by,
    COUNT(DISTINCT dal.table_name) AS tables_accessed_last_90_days,
    MAX(dal.event_timestamp) AS last_access_date
FROM security.user_master u
JOIN security.user_roles r ON u.username = r.username
LEFT JOIN audit.data_access_log dal ON u.username = dal.username 
    AND dal.event_timestamp >= CURRENT_DATE - INTERVAL '90 days'
WHERE r.role_name LIKE 'gold_%'
GROUP BY u.username, u.email, r.role_name, r.granted_date, r.approved_by
ORDER BY u.username, r.role_name;
```

## Operational Procedures

### Security Incident Response

**Procedure for Suspected Breach**:

1. **Detect**: Automated alerts or manual report
2. **Contain**: Immediately revoke suspicious access
3. **Investigate**: Review audit logs for extent of breach
4. **Remediate**: Fix vulnerability, reset credentials
5. **Report**: Notify stakeholders and compliance team
6. **Post-Mortem**: Document lessons learned

**Incident Log**:
```sql
CREATE TABLE security.incident_log (
    incident_id BIGSERIAL PRIMARY KEY,
    incident_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    incident_type VARCHAR(100),
    severity VARCHAR(20), -- CRITICAL, HIGH, MEDIUM, LOW
    description TEXT,
    affected_systems TEXT,
    affected_users TEXT,
    detection_method VARCHAR(100),
    response_actions TEXT,
    status VARCHAR(50), -- OPEN, INVESTIGATING, CONTAINED, RESOLVED
    resolved_timestamp TIMESTAMP,
    root_cause TEXT,
    preventive_measures TEXT
);
```

### Regular Security Tasks

**Weekly**:
- Review failed login attempts and access denials
- Check for unusual access patterns
- Verify backup completion and integrity

**Monthly**:
- Review and certify user access
- Analyze security event trends
- Test disaster recovery procedures

**Quarterly**:
- Conduct access recertification
- Review and update security policies
- Perform vulnerability assessment
- Security awareness training

**Annually**:
- Comprehensive security audit
- Penetration testing
- Review and update incident response plan
- Disaster recovery drill

## Key Performance Indicators

### Security Metrics

```sql
CREATE VIEW security.v_kpi_dashboard AS
SELECT 
    -- Access Control
    (SELECT COUNT(*) FROM security.user_roles WHERE role_name LIKE 'gold_%') AS total_gold_layer_users,
    (SELECT COUNT(DISTINCT username) FROM audit.data_access_log 
     WHERE event_timestamp >= CURRENT_DATE - INTERVAL '30 days') AS active_users_last_30_days,
    
    -- Audit Logging
    (SELECT COUNT(*) FROM audit.data_access_log 
     WHERE event_timestamp >= CURRENT_DATE) AS queries_logged_today,
    (SELECT AVG(execution_time_ms) FROM audit.data_access_log 
     WHERE event_timestamp >= CURRENT_DATE) AS avg_query_time_ms_today,
    
    -- Reconciliation
    (SELECT COUNT(*) FROM reconciliation.record_count_check 
     WHERE check_timestamp::DATE = CURRENT_DATE AND status = 'PASS') AS reconciliation_checks_passed_today,
    (SELECT COUNT(*) FROM reconciliation.record_count_check 
     WHERE check_timestamp::DATE = CURRENT_DATE AND status = 'FAIL') AS reconciliation_checks_failed_today,
    
    -- Security Events
    (SELECT COUNT(*) FROM audit.v_suspicious_activity 
     WHERE event_hour >= CURRENT_TIMESTAMP - INTERVAL '24 hours') AS suspicious_activities_last_24h,
    (SELECT COUNT(*) FROM audit.data_access_log 
     WHERE success = FALSE AND event_timestamp >= CURRENT_DATE) AS failed_access_attempts_today,
    
    -- Compliance
    (SELECT COUNT(*) FROM audit.gdpr_request_log 
     WHERE request_date >= CURRENT_DATE - INTERVAL '7 days') AS gdpr_requests_last_7_days,
    (SELECT AVG(EXTRACT(EPOCH FROM (completed_timestamp - request_timestamp))/86400) 
     FROM audit.gdpr_request_log 
     WHERE request_date >= CURRENT_DATE - INTERVAL '30 days') AS avg_gdpr_response_time_days;
```

## Checklist: Security Implementation

Before promoting table to production:

- [ ] Data classification documented
- [ ] Row-level security policies created (if needed)
- [ ] Column-level security configured (if needed)
- [ ] Data masking implemented for PII/sensitive data
- [ ] Appropriate roles and grants configured
- [ ] Audit logging enabled (triggers or platform-level)
- [ ] Encryption at rest enabled
- [ ] Encryption in transit enforced (TLS 1.3)
- [ ] Reconciliation procedures implemented
- [ ] Level 1 (record count) reconciliation passing
- [ ] Level 2 (aggregate) reconciliation passing
- [ ] Discrepancies investigated and resolved
- [ ] Backup and recovery tested
- [ ] Data retention policy configured
- [ ] Compliance requirements validated (GDPR, SOX, etc.)
- [ ] Security documentation completed
- [ ] Stakeholder sign-off obtained

---

**Document Control**

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | January 2026 |
| Owner | Data Architecture Team |
| Review Date | April 2026 |
