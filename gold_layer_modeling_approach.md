# Gold Layer Modeling Approach and Architectural Principles

## Overview

This document defines the data modeling methodology, architectural patterns, and design principles that guide gold layer development in the SAS migration project. The gold layer employs dimensional modeling techniques optimized for analytical workloads while maintaining flexibility for evolving business requirements.

## Core Modeling Philosophy

### Dimensional Modeling Foundation

The gold layer employs **Kimball dimensional modeling** as its primary methodology, which provides:

- **Business User Focus**: Intuitive structure aligned with how business users think about data
- **Query Performance**: Optimized for typical analytical query patterns
- **Flexibility**: Supports ad-hoc analysis and changing business requirements
- **Consistency**: Conformed dimensions enable cross-process analysis

### Complementary Approaches

While dimensional modeling is primary, the gold layer incorporates complementary techniques:

- **Data Vault patterns** for complex many-to-many relationships and deep historization
- **One Big Table (OBT)** for specific high-performance use cases
- **Metric stores** for pre-calculated KPIs and complex business rules
- **Graph patterns** for network and hierarchical analysis

## Architectural Principles

### 1. Business-Driven Design

**Principle**: Model data according to business processes and terminology, not source system structure.

**Application**:
- Organize by business subject areas (sales, finance, customer, product)
- Use business terminology in table and column names
- Structure around business questions and analytics needs
- Decouple from source system structures

**Example**:
```
✓ GOOD: fact_sales_transaction, dim_customer, dim_product
✗ AVOID: erp_table_456, system_a_customer, src_prod_mstr
```

### 2. Conformed Dimensions

**Principle**: Shared dimensions have consistent structure and grain across all fact tables.

**Application**:
- Single customer dimension used by all business processes
- Consistent date dimension across the enterprise
- Standardized product dimension regardless of source
- Common attributes named and defined identically

**Benefits**:
- Enables drill-across analysis between fact tables
- Ensures consistency in reporting and analysis
- Reduces confusion and errors
- Simplifies BI tool configuration

**Example**:
```sql
-- Customer dimension used by sales, service, marketing facts
-- Always has same grain: one row per customer
dim_customer (customer_key, customer_id, customer_name, ...)

-- Used consistently in all fact tables
fact_sales_transaction (customer_key, ...)
fact_service_ticket (customer_key, ...)
fact_marketing_campaign_response (customer_key, ...)
```

### 3. Grain Clarity

**Principle**: Every fact table has a precisely defined grain - the business meaning of one row.

**Application**:
- Document grain explicitly in table metadata
- All facts and dimensions must be true to the grain
- Cannot mix granularities in a single fact table
- When grain changes, create separate fact table

**Examples**:

**Transaction Grain**:
```
fact_sales_transaction
Grain: One row per line item on each sales order
```

**Periodic Snapshot Grain**:
```
fact_account_balance_daily
Grain: One row per account per day
```

**Accumulating Snapshot Grain**:
```
fact_order_fulfillment
Grain: One row per order, updated as order progresses through lifecycle
```

### 4. Dimensional Role-Playing

**Principle**: A single dimension can be used multiple times in a fact table with different meanings.

**Application**:
- Use foreign key naming to indicate role
- Create views with role-specific names if needed
- Document each role clearly

**Example**:
```sql
fact_sales_transaction (
    transaction_id,
    order_date_key,           -- Role: When order was placed
    ship_date_key,            -- Role: When order was shipped
    required_date_key,        -- Role: Customer requested date
    invoice_date_key,         -- Role: When invoice was generated
    bill_customer_key,        -- Role: Customer being billed
    ship_customer_key,        -- Role: Customer receiving shipment
    ...
)
```

### 5. Slowly Changing Dimensions

**Principle**: Track historical changes in dimension attributes appropriately for business needs.

**SCD Type Selection**:

**Type 0 - Retain Original**: Never changes
```
dim_customer.customer_birth_date
dim_product.product_launch_date
```

**Type 1 - Overwrite**: Current value only, no history
```
dim_customer.email_address
dim_customer.phone_number
dim_product.product_description
```

**Type 2 - Track History**: Full change history with effective dating
```
dim_customer.customer_credit_tier
dim_customer.billing_address
dim_product.product_category
dim_product.unit_cost
```

**Type 2 Implementation**:
```sql
CREATE TABLE dim_customer (
    customer_key BIGINT PRIMARY KEY,        -- Surrogate key
    customer_id VARCHAR(50) NOT NULL,        -- Business key
    customer_name VARCHAR(200),
    customer_segment VARCHAR(50),
    customer_credit_tier VARCHAR(20),
    -- SCD2 tracking columns
    effective_date DATE NOT NULL,
    expiration_date DATE,                    -- NULL for current
    is_current BOOLEAN DEFAULT TRUE,
    row_hash VARCHAR(64),                    -- For change detection
    created_timestamp TIMESTAMP,
    modified_timestamp TIMESTAMP,
    source_system VARCHAR(50),
    batch_id VARCHAR(100)
);

-- Support query for current records
CREATE INDEX ix_dim_customer_current 
ON dim_customer(customer_id) WHERE is_current = TRUE;
```

**Type 3 - Track Limited History**: Store previous value
```sql
dim_customer (
    customer_key,
    current_customer_segment,
    prior_customer_segment,
    segment_change_date
)
```

### 6. Fact Table Patterns

#### Transaction Fact Tables

**Characteristics**:
- One row per business event
- Most granular level of detail
- Sparse - only records when events occur
- Additive measures preferred

**Example**:
```sql
CREATE TABLE fact_sales_transaction (
    transaction_key BIGINT PRIMARY KEY,
    transaction_id VARCHAR(50) NOT NULL,
    order_date_key INT NOT NULL,
    customer_key BIGINT NOT NULL,
    product_key BIGINT NOT NULL,
    store_key INT NOT NULL,
    -- Additive measures
    quantity DECIMAL(18,4),
    unit_price DECIMAL(18,4),
    extended_price DECIMAL(18,4),
    discount_amount DECIMAL(18,4),
    tax_amount DECIMAL(18,4),
    net_amount DECIMAL(18,4),
    -- Semi-additive measures
    unit_cost DECIMAL(18,4),
    -- Non-additive measures (stored for reference)
    profit_margin_percent DECIMAL(5,2),
    -- Degenerate dimensions
    order_number VARCHAR(50),
    invoice_number VARCHAR(50),
    -- Audit
    batch_id VARCHAR(100),
    created_timestamp TIMESTAMP
);
```

#### Periodic Snapshot Fact Tables

**Characteristics**:
- One row per entity per period
- Dense - records exist even with no activity
- Captures state at regular intervals
- Semi-additive measures (additive across dimensions but not time)

**Example**:
```sql
CREATE TABLE fact_inventory_level_daily (
    snapshot_date_key INT NOT NULL,
    product_key BIGINT NOT NULL,
    warehouse_key INT NOT NULL,
    -- Semi-additive measures (point-in-time balances)
    on_hand_quantity DECIMAL(18,4),
    available_quantity DECIMAL(18,4),
    reserved_quantity DECIMAL(18,4),
    on_order_quantity DECIMAL(18,4),
    -- Additive measures (activity during day)
    receipts_quantity DECIMAL(18,4),
    shipments_quantity DECIMAL(18,4),
    adjustments_quantity DECIMAL(18,4),
    -- Calculated metrics
    days_supply INT,
    inventory_value DECIMAL(18,4),
    -- Audit
    batch_id VARCHAR(100),
    created_timestamp TIMESTAMP,
    PRIMARY KEY (snapshot_date_key, product_key, warehouse_key)
);
```

#### Accumulating Snapshot Fact Tables

**Characteristics**:
- One row per entity tracked through a process
- Multiple date stamps capturing milestone events
- Updated as entity progresses through workflow
- Includes elapsed time between milestones

**Example**:
```sql
CREATE TABLE fact_order_fulfillment (
    order_key BIGINT PRIMARY KEY,
    order_id VARCHAR(50) NOT NULL,
    customer_key BIGINT NOT NULL,
    -- Multiple date foreign keys for milestones
    order_date_key INT,
    payment_date_key INT,
    pick_date_key INT,
    pack_date_key INT,
    ship_date_key INT,
    delivery_date_key INT,
    -- Measures
    order_amount DECIMAL(18,4),
    shipping_cost DECIMAL(18,4),
    -- Lag measures (days between milestones)
    order_to_payment_days INT,
    order_to_ship_days INT,
    ship_to_delivery_days INT,
    order_to_delivery_days INT,
    -- Status tracking
    current_status VARCHAR(50),
    -- Audit
    created_timestamp TIMESTAMP,
    modified_timestamp TIMESTAMP,
    batch_id VARCHAR(100)
);
```

### 7. Junk Dimensions

**Principle**: Consolidate low-cardinality transaction flags and indicators into a single dimension.

**Application**:
- Reduces dimension table proliferation
- Improves fact table efficiency
- Pre-populate with common combinations

**Example**:
```sql
CREATE TABLE dim_transaction_flags (
    transaction_flags_key INT PRIMARY KEY,
    is_promotional_sale BOOLEAN,
    is_online_order BOOLEAN,
    is_gift_wrapped BOOLEAN,
    is_express_shipping BOOLEAN,
    has_discount_applied BOOLEAN,
    payment_type VARCHAR(20)
);

-- Fact table reference
fact_sales_transaction (
    ...,
    transaction_flags_key INT,  -- Reference to junk dimension
    ...
);
```

### 8. Bridge Tables for Many-to-Many

**Principle**: Use bridge tables to resolve many-to-many relationships between facts and dimensions.

**Example - Product Bundles**:
```sql
-- Dimension of individual products
CREATE TABLE dim_product (
    product_key BIGINT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(200),
    ...
);

-- Bridge table for product bundles
CREATE TABLE bridge_product_bundle (
    bundle_key BIGINT,              -- Foreign key to dimension
    component_product_key BIGINT,   -- Foreign key to dimension
    quantity DECIMAL(18,4),         -- How many of component in bundle
    allocation_percent DECIMAL(5,4), -- For revenue allocation
    PRIMARY KEY (bundle_key, component_product_key)
);

-- Fact table can reference either individual products or bundles
fact_sales_transaction (
    product_key BIGINT,  -- Could be individual or bundle
    ...
);
```

## Design Patterns for Common Scenarios

### Pattern 1: Customer 360 View

**Objective**: Comprehensive customer view integrating data across business processes.

**Approach**:
```sql
-- Core customer dimension
dim_customer (
    customer_key,
    customer_id,
    customer_name,
    customer_segment,
    customer_tier,
    ...
)

-- Customer metrics/KPIs in metric table
metric_customer_lifetime_value (
    customer_key,
    calculation_date_key,
    lifetime_revenue,
    lifetime_profit,
    predicted_lifetime_value,
    churn_probability,
    customer_health_score,
    ...
)

-- Integration across fact tables
fact_sales_transaction (customer_key, ...)
fact_service_interaction (customer_key, ...)
fact_marketing_campaign_response (customer_key, ...)

-- 360 view via query
SELECT 
    c.customer_name,
    clv.lifetime_revenue,
    sales.total_orders,
    service.total_tickets,
    marketing.campaign_responses
FROM dim_customer c
LEFT JOIN metric_customer_lifetime_value clv ...
LEFT JOIN (SELECT customer_key, COUNT(*) ...) sales ...
LEFT JOIN (SELECT customer_key, COUNT(*) ...) service ...
LEFT JOIN (SELECT customer_key, COUNT(*) ...) marketing ...
```

### Pattern 2: Product Hierarchy

**Objective**: Handle multi-level product categorization.

**Approach**:
```sql
-- Product dimension with flattened hierarchy
CREATE TABLE dim_product (
    product_key BIGINT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(200),
    -- Level 1
    division_code VARCHAR(20),
    division_name VARCHAR(100),
    -- Level 2
    department_code VARCHAR(20),
    department_name VARCHAR(100),
    -- Level 3
    category_code VARCHAR(20),
    category_name VARCHAR(100),
    -- Level 4
    subcategory_code VARCHAR(20),
    subcategory_name VARCHAR(100),
    -- Product attributes
    brand VARCHAR(100),
    manufacturer VARCHAR(100),
    unit_of_measure VARCHAR(20),
    ...
);

-- Alternative: Separate hierarchy dimension for flexibility
CREATE TABLE dim_product_hierarchy (
    hierarchy_key INT PRIMARY KEY,
    division_code VARCHAR(20),
    division_name VARCHAR(100),
    department_code VARCHAR(20),
    department_name VARCHAR(100),
    category_code VARCHAR(20),
    category_name VARCHAR(100),
    subcategory_code VARCHAR(20),
    subcategory_name VARCHAR(100)
);

CREATE TABLE dim_product (
    product_key BIGINT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(200),
    hierarchy_key INT,  -- Reference to hierarchy
    ...
);
```

### Pattern 3: Time Series Analysis

**Objective**: Support trending, seasonality, and comparative analysis.

**Approach**:
```sql
-- Comprehensive date dimension
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,          -- YYYYMMDD format
    full_date DATE NOT NULL,
    -- Calendar attributes
    day_of_week INT,
    day_of_week_name VARCHAR(20),
    day_of_month INT,
    day_of_year INT,
    week_of_year INT,
    month_number INT,
    month_name VARCHAR(20),
    quarter_number INT,
    quarter_name VARCHAR(10),
    year_number INT,
    -- Fiscal attributes
    fiscal_year INT,
    fiscal_quarter INT,
    fiscal_month INT,
    fiscal_week INT,
    -- Flags
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(100),
    is_business_day BOOLEAN,
    -- Prior period references
    prior_day_date_key INT,
    prior_week_date_key INT,
    prior_month_date_key INT,
    prior_year_date_key INT,
    -- Relative attributes
    days_from_today INT,               -- For relative queries
    is_current_week BOOLEAN,
    is_current_month BOOLEAN,
    is_current_quarter BOOLEAN,
    is_current_year BOOLEAN
);
```

### Pattern 4: Event/Activity Schema

**Objective**: Track diverse business events with flexible attributes.

**Approach**:
```sql
-- Core event fact
CREATE TABLE fact_business_event (
    event_key BIGINT PRIMARY KEY,
    event_id VARCHAR(50),
    event_date_key INT NOT NULL,
    event_timestamp TIMESTAMP NOT NULL,
    event_type_key INT NOT NULL,
    customer_key BIGINT,
    product_key BIGINT,
    -- Standard measures
    event_value DECIMAL(18,4),
    event_quantity DECIMAL(18,4),
    -- JSON for flexible attributes
    event_attributes JSON,
    -- Audit
    source_system VARCHAR(50),
    batch_id VARCHAR(100),
    created_timestamp TIMESTAMP
);

-- Event type dimension
CREATE TABLE dim_event_type (
    event_type_key INT PRIMARY KEY,
    event_type_code VARCHAR(50),
    event_type_name VARCHAR(200),
    event_category VARCHAR(100),
    is_revenue_event BOOLEAN,
    is_engagement_event BOOLEAN,
    event_weight DECIMAL(5,2)
);
```

## Aggregation Strategy

### When to Aggregate

Create aggregate tables when:
1. **Query patterns are predictable**: Most queries at specific grain (monthly, by region)
2. **Base fact is large**: Billions of rows making queries slow
3. **Calculations are expensive**: Complex business rules computed repeatedly
4. **BI tools need optimization**: Supporting hundreds of concurrent users

### Aggregate Design

**Naming**: `agg_<fact>_<grain>_<timeperiod>`

**Example**:
```sql
-- Daily transaction fact (base grain)
fact_sales_transaction (100M+ rows)

-- Monthly aggregate by customer and product
CREATE TABLE agg_sales_customer_product_monthly (
    month_date_key INT NOT NULL,
    customer_key BIGINT NOT NULL,
    product_key BIGINT NOT NULL,
    -- Aggregated measures
    total_quantity DECIMAL(18,4),
    total_revenue DECIMAL(18,4),
    total_cost DECIMAL(18,4),
    total_profit DECIMAL(18,4),
    transaction_count INT,
    average_order_value DECIMAL(18,4),
    -- Audit
    aggregation_timestamp TIMESTAMP,
    batch_id VARCHAR(100),
    PRIMARY KEY (month_date_key, customer_key, product_key)
);

-- Quarterly aggregate by region
CREATE TABLE agg_sales_region_quarterly (
    quarter_date_key INT NOT NULL,
    region_key INT NOT NULL,
    -- Aggregated measures
    total_revenue DECIMAL(18,4),
    total_profit DECIMAL(18,4),
    customer_count INT,
    average_customer_revenue DECIMAL(18,4),
    PRIMARY KEY (quarter_date_key, region_key)
);
```

### Aggregate Refresh Strategy

1. **Full rebuild**: Simple but resource-intensive
2. **Incremental append**: For append-only facts
3. **Sliding window**: Keep last N periods, drop older
4. **Triggered updates**: Refresh when base table changes

## Partitioning and Clustering Strategy

### Partitioning Principles

**Date Partitioning** (Primary Strategy):
```sql
-- Partition by day for recent, detailed data
CREATE TABLE fact_sales_transaction (
    transaction_date DATE,
    ...
)
PARTITION BY RANGE (transaction_date);

-- Partition by month for historical data
CREATE TABLE fact_sales_transaction_historical (
    transaction_date DATE,
    ...
)
PARTITION BY RANGE (DATE_TRUNC('month', transaction_date));
```

**Benefits**:
- Partition pruning for date-filtered queries
- Efficient data lifecycle management
- Parallel processing

### Clustering Strategy

**Cluster on frequently filtered dimensions**:
```sql
-- Cluster by customer and product for drill-down queries
CREATE TABLE fact_sales_transaction (
    ...
)
PARTITION BY RANGE (transaction_date)
CLUSTER BY (customer_key, product_key);
```

**Considerations**:
- Cluster keys should have high cardinality
- Order by query frequency (most filtered first)
- Limit to 3-4 cluster columns
- Monitor and adjust based on query patterns

## Data Quality and Constraints

### Primary Key Enforcement

```sql
-- Every table must have primary key
ALTER TABLE dim_customer ADD CONSTRAINT pk_dim_customer 
PRIMARY KEY (customer_key);

-- Composite keys for fact tables
ALTER TABLE fact_sales_transaction ADD CONSTRAINT pk_fact_sales 
PRIMARY KEY (transaction_key);
```

### Foreign Key Relationships

```sql
-- Enforce referential integrity where appropriate
ALTER TABLE fact_sales_transaction 
ADD CONSTRAINT fk_sales_customer 
FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key);

-- Consider NOT ENFORCED for performance in high-volume loads
ALTER TABLE fact_sales_transaction 
ADD CONSTRAINT fk_sales_product 
FOREIGN KEY (product_key) REFERENCES dim_product(product_key)
NOT ENFORCED;
```

### Check Constraints

```sql
-- Ensure data validity
ALTER TABLE fact_sales_transaction 
ADD CONSTRAINT chk_quantity_positive 
CHECK (quantity > 0);

ALTER TABLE fact_sales_transaction 
ADD CONSTRAINT chk_amount_valid 
CHECK (net_amount = extended_price - discount_amount + tax_amount);
```

### Not Null Constraints

```sql
-- Enforce required fields
ALTER TABLE dim_customer ALTER COLUMN customer_id SET NOT NULL;
ALTER TABLE fact_sales_transaction ALTER COLUMN order_date_key SET NOT NULL;
ALTER TABLE fact_sales_transaction ALTER COLUMN customer_key SET NOT NULL;
```

## Denormalization Guidelines

### When to Denormalize

**Justified Denormalization**:
1. **Dimension attributes in facts**: Small, stable attributes frequently queried
2. **Hierarchy flattening**: Product/geographic hierarchies in dimension
3. **Computed values**: Pre-calculated metrics avoiding complex joins
4. **One Big Table**: Specific departmental or reporting needs

**Example**:
```sql
-- Denormalize stable product attributes into fact
CREATE TABLE fact_sales_transaction (
    transaction_key BIGINT,
    product_key BIGINT,           -- Full dimension available
    product_category VARCHAR(50),  -- Denormalized for filtering
    product_brand VARCHAR(50),     -- Denormalized for grouping
    ...
);
```

### Controlled Denormalization

- Document denormalized attributes clearly
- Maintain synchronization with dimension
- Use for read-optimization only
- Keep source of truth in dimension

## Schema Evolution and Versioning

### Backward Compatible Changes

**Safe Operations**:
- Adding new columns (with defaults)
- Adding new tables
- Creating new indexes
- Extending column length
- Making required columns optional

```sql
-- Add new column (backward compatible)
ALTER TABLE dim_customer 
ADD COLUMN customer_lifetime_status VARCHAR(20) DEFAULT 'Active';
```

### Breaking Changes

**Require Versioning**:
- Removing columns
- Renaming columns
- Changing data types
- Changing business logic

**Versioning Strategy**:
```sql
-- Create new version
CREATE TABLE dim_customer_v2 AS 
SELECT 
    customer_key,
    customer_id,
    customer_full_name,  -- Renamed from customer_name
    ... 
FROM dim_customer;

-- Maintain compatibility view
CREATE VIEW dim_customer AS 
SELECT 
    customer_key,
    customer_id,
    customer_full_name AS customer_name,  -- Backward compatibility
    ...
FROM dim_customer_v2;
```

## Performance Optimization Techniques

### Materialized Views

```sql
CREATE MATERIALIZED VIEW mv_customer_sales_summary AS
SELECT 
    c.customer_key,
    c.customer_name,
    c.customer_segment,
    COUNT(DISTINCT s.order_id) AS order_count,
    SUM(s.net_amount) AS total_revenue,
    MAX(s.order_date) AS last_order_date
FROM dim_customer c
LEFT JOIN fact_sales_transaction s ON c.customer_key = s.customer_key
GROUP BY c.customer_key, c.customer_name, c.customer_segment;

-- Refresh strategy
REFRESH MATERIALIZED VIEW mv_customer_sales_summary;
```

### Covering Indexes

```sql
-- Index includes all columns needed by common query
CREATE INDEX ix_sales_customer_product_covering 
ON fact_sales_transaction (customer_key, product_key)
INCLUDE (order_date, quantity, net_amount);
```

### Statistics Maintenance

```sql
-- Update statistics regularly for query optimization
ANALYZE dim_customer;
ANALYZE fact_sales_transaction;

-- Automatic statistics collection
-- Configure database to auto-update statistics on significant changes
```

## Documentation Standards

### Table Documentation Template

```sql
COMMENT ON TABLE fact_sales_transaction IS 
'PURPOSE: Records individual sales transactions from all channels (retail, online, wholesale).
GRAIN: One row per line item on each sales order.
LOAD FREQUENCY: Near real-time (15-minute intervals).
RETENTION: 7 years of detailed transactions.
SOURCE SYSTEMS: POS (retail), E-commerce platform, B2B portal.
SAS MIGRATION: Replaces daily_sales_fact.sas and sales_detail.sas.
BUSINESS OWNER: VP of Sales Operations (jane.smith@company.com).
DATA STEWARD: Sales Analytics Team (sales-analytics@company.com).
LAST MODIFIED: 2024-01-15 by Data Engineering Team.
RELATED TABLES: dim_customer, dim_product, dim_store, dim_date.';
```

## Quality Checklist

Before promoting to production, verify:

- [ ] Grain explicitly defined and enforced
- [ ] Primary key defined and unique
- [ ] Foreign keys validated (enforced or checked)
- [ ] Naming conventions followed
- [ ] SCD strategy implemented correctly
- [ ] Audit columns populated
- [ ] Partitioning/clustering configured
- [ ] Indexes created for common queries
- [ ] Constraints defined appropriately
- [ ] Documentation complete (table and column comments)
- [ ] Data lineage documented
- [ ] Security classifications applied
- [ ] Reconciliation against SAS validated
- [ ] Performance tested with realistic data volumes
- [ ] Backup and recovery verified

---

**Document Control**

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | January 2026 |
| Owner | Data Architecture Team |
| Review Date | April 2026 |
