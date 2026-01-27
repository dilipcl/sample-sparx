# Gold Layer Modeling Approach and Architectural Principles

## Overview

This document defines the data modeling methodology, architectural patterns, and design principles that guide gold layer development in the SAS migration project on **Databricks Lakehouse Platform**. The gold layer employs dimensional modeling techniques optimized for analytical workloads while leveraging Databricks-specific capabilities including Delta Lake, Unity Catalog, and Photon engine for enhanced performance and governance.

## Core Modeling Philosophy

### Dimensional Modeling Foundation

The gold layer employs **Kimball dimensional modeling** as its primary methodology, which provides:

- **Business User Focus**: Intuitive structure aligned with how business users think about data
- **Query Performance**: Optimized for typical analytical query patterns
- **Flexibility**: Supports ad-hoc analysis and changing business requirements
- **Consistency**: Conformed dimensions enable cross-process analysis

### Complementary Approaches

While dimensional modeling is primary, the gold layer incorporates complementary techniques optimized for Databricks:

- **Delta Lake features** for ACID transactions, time travel, and schema evolution
- **Data Vault patterns** for complex many-to-many relationships and deep historization
- **One Big Table (OBT)** for specific high-performance use cases (Databricks excels at wide tables)
- **Metric stores** for pre-calculated KPIs and complex business rules
- **Medallion architecture native patterns** aligned with Delta Lake best practices
- **Liquid Clustering** for automated data organization (replacing traditional partitioning)
- **Materialized views and streaming tables** for real-time analytics

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

### 7. Delta Lake and Databricks-Specific Features

**Principle**: Leverage Databricks platform capabilities for enhanced performance, reliability, and governance.

**Delta Lake Implementation**:

```sql
-- All gold layer tables use Delta Lake format
CREATE TABLE gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT,
    transaction_id STRING NOT NULL,
    order_date DATE NOT NULL,
    customer_key BIGINT NOT NULL,
    product_key BIGINT NOT NULL,
    quantity DECIMAL(18,4),
    net_amount DECIMAL(18,4),
    created_timestamp TIMESTAMP
)
USING DELTA
TBLPROPERTIES (
    'delta.enableChangeDataFeed' = 'true',
    'delta.deletedFileRetentionDuration' = 'interval 30 days',
    'delta.logRetentionDuration' = 'interval 90 days'
)
CLUSTER BY (order_date, customer_key);  -- Liquid clustering
```

**Key Delta Lake Features**:

**1. ACID Transactions & Upserts**:
```sql
-- Atomic merge operations for incremental loads
MERGE INTO gold_sales_retail.fact_sales_transaction AS target
USING staging.sales_updates AS source
ON target.transaction_id = source.transaction_id
WHEN MATCHED AND source.modified_timestamp > target.modified_timestamp
  THEN UPDATE SET *
WHEN NOT MATCHED
  THEN INSERT *;
```

**2. Time Travel for Auditing**:
```sql
-- Query historical versions
SELECT * FROM gold_sales_retail.fact_sales_transaction 
VERSION AS OF 10;

SELECT * FROM gold_sales_retail.fact_sales_transaction 
TIMESTAMP AS OF '2024-01-15 10:00:00';

-- Restore to previous version if needed
RESTORE TABLE gold_sales_retail.fact_sales_transaction 
TO VERSION AS OF 5;

-- Reconciliation using time travel
CREATE OR REPLACE VIEW reconciliation.v_daily_comparison AS
SELECT 
    'Today' AS period,
    COUNT(*) AS row_count,
    SUM(net_amount) AS total_amount
FROM gold_sales_retail.fact_sales_transaction
UNION ALL
SELECT 
    'Yesterday' AS period,
    COUNT(*) AS row_count,
    SUM(net_amount) AS total_amount
FROM gold_sales_retail.fact_sales_transaction 
TIMESTAMP AS OF CURRENT_TIMESTAMP() - INTERVAL 1 DAY;
```

**3. Change Data Feed (CDF)**:
```sql
-- Enable CDF for efficient incremental processing
ALTER TABLE gold_sales_retail.fact_sales_transaction 
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);

-- Read only changes since last load
CREATE OR REPLACE TEMP VIEW sales_changes AS
SELECT * 
FROM table_changes('gold_sales_retail.fact_sales_transaction', 2, 10)
WHERE _change_type IN ('insert', 'update_postimage');

-- Use CDF for downstream aggregates
MERGE INTO gold_sales_retail.agg_sales_customer_monthly AS target
USING (
  SELECT 
    DATE_TRUNC('month', order_date) AS month_date,
    customer_key,
    SUM(net_amount) AS total_amount
  FROM table_changes('gold_sales_retail.fact_sales_transaction', 
       (SELECT MAX(last_processed_version) FROM audit.cdf_checkpoints))
  WHERE _change_type IN ('insert', 'update_postimage')
  GROUP BY DATE_TRUNC('month', order_date), customer_key
) AS source
ON target.month_date = source.month_date 
   AND target.customer_key = source.customer_key
WHEN MATCHED THEN UPDATE SET total_amount = target.total_amount + source.total_amount
WHEN NOT MATCHED THEN INSERT *;
```

**4. Liquid Clustering (Recommended over Partitioning)**:
```sql
-- Replaces traditional PARTITION BY with automatic clustering
CREATE TABLE gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT,
    order_date DATE NOT NULL,
    customer_key BIGINT NOT NULL,
    product_key BIGINT NOT NULL,
    net_amount DECIMAL(18,4)
)
USING DELTA
CLUSTER BY (order_date, customer_key, product_key);

-- Benefits:
-- ✓ Automatic maintenance as data changes
-- ✓ Multi-dimensional optimization
-- ✓ No small file problems
-- ✓ Supports high cardinality columns
-- ✓ No need to specify at write time

-- Periodically optimize clustering
OPTIMIZE gold_sales_retail.fact_sales_transaction;

-- For existing tables, convert from partitioning to clustering
ALTER TABLE gold_sales_retail.fact_sales_transaction 
CLUSTER BY (order_date, customer_key);
```

**When to Use Traditional Partitioning vs Liquid Clustering**:

```
USE LIQUID CLUSTERING when:
✓ Databricks Runtime 13.3+
✓ Multi-dimensional query patterns
✓ Need flexibility in clustering columns
✓ Want automatic maintenance (recommended default)

USE TRADITIONAL PARTITIONING when:
✓ Must use older Databricks runtime
✓ Strict data lifecycle requirements (e.g., delete partitions)
✓ Downstream consumers expect partitioned structure
✓ Single dimension dominates all queries (rare)

Example Traditional Partitioning (if needed):
CREATE TABLE fact_sales_transaction (
    ...
)
USING DELTA
PARTITIONED BY (order_date);
```

**5. Unity Catalog Integration**:
```sql
-- Three-level namespace for organized governance
CREATE CATALOG IF NOT EXISTS production;
USE CATALOG production;

CREATE SCHEMA IF NOT EXISTS production.gold_sales_retail
COMMENT 'Retail sales gold layer'
WITH DBPROPERTIES (
    'owner' = 'sales_team',
    'data_classification' = 'confidential',
    'business_domain' = 'sales'
);

-- Create table with comprehensive metadata
CREATE TABLE production.gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT COMMENT 'Surrogate key',
    transaction_id STRING NOT NULL COMMENT 'Business key from source',
    customer_key BIGINT COMMENT 'FK to customer dimension'
)
USING DELTA
COMMENT 'Daily retail sales transactions from all channels'
TBLPROPERTIES (
    'delta.enableChangeDataFeed' = 'true',
    'quality_score' = '0.99',
    'refresh_frequency' = 'daily',
    'sas_source' = 'sales_daily_load.sas'
)
CLUSTER BY (order_date, customer_key);

-- Tag sensitive columns for data governance
ALTER TABLE production.gold_sales_retail.fact_sales_transaction 
ALTER COLUMN customer_key SET TAGS ('pii_category' = 'identifier');
```

**6. Databricks-Optimized SCD Type 2**:
```python
# Efficient SCD Type 2 using Delta Lake MERGE
from delta.tables import DeltaTable
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Source data
source_df = spark.table("silver.customer_updates")

# Calculate row hash for change detection
source_df = source_df.withColumn(
    "row_hash",
    F.sha2(F.concat_ws("||", *[F.col(c) for c in source_df.columns]), 256)
)

# Target dimension
target = DeltaTable.forName(spark, "gold_customer_360.dim_customer")
target_df = target.toDF().filter("is_current = true")

# Identify changes
changes_df = (
    source_df.alias("source")
    .join(target_df.alias("target"), "customer_id", "left_outer")
    .where(
        (F.col("target.customer_key").isNull()) |  # New records
        (F.col("source.row_hash") != F.col("target.row_hash"))  # Changed records
    )
    .select("source.*")
)

# Expire old versions (SCD Type 2 closure)
target.alias("target").merge(
    changes_df.alias("source"),
    """target.customer_id = source.customer_id 
       AND target.is_current = true"""
).whenMatchedUpdate(
    set = {
        "is_current": "false",
        "expiration_date": "current_date()",
        "modified_timestamp": "current_timestamp()"
    }
).execute()

# Insert new versions
new_versions = (
    changes_df
    .withColumn("customer_key", F.expr("uuid()"))  # Generate surrogate key
    .withColumn("effective_date", F.current_date())
    .withColumn("expiration_date", F.lit(None).cast("date"))
    .withColumn("is_current", F.lit(True))
    .withColumn("created_timestamp", F.current_timestamp())
    .withColumn("batch_id", F.lit(spark.conf.get("spark.databricks.job.id")))
)

# Use MERGE for idempotency
target.alias("target").merge(
    new_versions.alias("source"),
    """target.customer_id = source.customer_id 
       AND target.effective_date = source.effective_date 
       AND target.is_current = true"""
).whenNotMatchedInsert(
    values = {col: f"source.{col}" for col in new_versions.columns}
).execute()

# Optimize after SCD operations
spark.sql("OPTIMIZE gold_customer_360.dim_customer")
```

**7. Schema Evolution**:
```sql
-- Delta Lake supports automatic schema evolution
SET spark.databricks.delta.schema.autoMerge.enabled = true;

-- Add new columns without rewriting entire table
ALTER TABLE gold_sales_retail.fact_sales_transaction 
ADD COLUMNS (
    promotion_id STRING COMMENT 'Promotion applied',
    channel_type STRING COMMENT 'Sales channel'
);

-- Merge will automatically handle new columns
MERGE INTO gold_sales_retail.fact_sales_transaction AS target
USING staging.sales_with_new_schema AS source
ON target.transaction_id = source.transaction_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

**8. Performance Optimization**:
```sql
-- Z-Order optimization for frequently co-filtered columns
OPTIMIZE gold_sales_retail.fact_sales_transaction
ZORDER BY (customer_key, product_key);

-- Vacuum to remove old files (beyond retention)
VACUUM gold_sales_retail.fact_sales_transaction RETAIN 168 HOURS;  -- 7 days

-- Analyze table for query optimization
ANALYZE TABLE gold_sales_retail.fact_sales_transaction 
COMPUTE STATISTICS FOR ALL COLUMNS;

-- Enable Photon acceleration (cluster setting)
-- Photon automatically accelerates:
-- - Delta and Parquet scans
-- - Joins and aggregations
-- - Window functions
-- - Complex expressions
```

**9. Streaming Tables for Real-Time Gold Layer**:
```sql
-- Create streaming table for near real-time processing
CREATE OR REFRESH STREAMING TABLE gold_sales_retail.fact_sales_transaction_stream
AS SELECT 
    uuid() AS transaction_key,
    transaction_id,
    order_date,
    customer_key,
    product_key,
    net_amount,
    current_timestamp() AS created_timestamp
FROM STREAM(silver.sales_transactions);

-- Materialized view for pre-aggregated metrics
CREATE OR REFRESH MATERIALIZED VIEW gold_sales_retail.mv_sales_daily_summary
AS SELECT 
    order_date,
    customer_key,
    COUNT(*) AS transaction_count,
    SUM(net_amount) AS total_amount,
    AVG(net_amount) AS avg_amount
FROM gold_sales_retail.fact_sales_transaction
GROUP BY order_date, customer_key;
```

### 8. Junk Dimensions

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

### 9. Bridge Tables for Many-to-Many

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

### Databricks Liquid Clustering (Recommended)

**Liquid Clustering** is the recommended approach for Databricks Runtime 13.3+:

```sql
-- Use Liquid Clustering for automatic optimization
CREATE TABLE gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT,
    transaction_id STRING,
    order_date DATE,
    customer_key BIGINT,
    product_key BIGINT,
    store_key INT,
    net_amount DECIMAL(18,4),
    created_timestamp TIMESTAMP
)
USING DELTA
CLUSTER BY (order_date, customer_key, product_key);

-- Advantages over traditional partitioning:
-- ✓ Automatically maintains clustering as data changes
-- ✓ Optimizes for multiple query patterns simultaneously
-- ✓ No small file problems
-- ✓ Supports high cardinality columns
-- ✓ No partition specification needed at write time
-- ✓ Incremental clustering on INSERT/MERGE/UPDATE

-- Periodically optimize (can be scheduled)
OPTIMIZE gold_sales_retail.fact_sales_transaction;
```

**Clustering Column Selection Guidelines**:
```
Priority 1: Date/timestamp columns (most common filter)
Priority 2: High-cardinality foreign keys (customer, product)
Priority 3: Frequently filtered dimension keys
Priority 4: JOIN keys for common query patterns

Best Practices:
- Limit to 3-4 clustering columns
- Order by query frequency (most common first)
- Supports up to unlimited cardinality (unlike ZORDER)
- Can be changed without rewriting data: ALTER TABLE ... CLUSTER BY (...)
```

**Example Clustering Patterns by Table Type**:

```sql
-- Fact table: Date + key dimensions
CREATE TABLE fact_sales_transaction (...)
USING DELTA
CLUSTER BY (order_date, customer_key, product_key);

-- Large dimension: Business key + filter attributes
CREATE TABLE dim_customer (...)
USING DELTA
CLUSTER BY (customer_id, customer_segment, geography_key);

-- Periodic snapshot: Date + entity
CREATE TABLE fact_inventory_daily (...)
USING DELTA
CLUSTER BY (snapshot_date, warehouse_key, product_key);

-- Bridge table: Both relationship ends
CREATE TABLE bridge_product_hierarchy (...)
USING DELTA
CLUSTER BY (parent_product_key, child_product_key);
```

### Traditional Partitioning (Alternative Approach)

Use traditional partitioning only when:
- Using Databricks Runtime < 13.3
- Strict data lifecycle requirements (e.g., DROP PARTITION for GDPR)
- Downstream systems expect partitioned structure
- Single dimension completely dominates all queries

```sql
-- Traditional date partitioning
CREATE TABLE gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT,
    transaction_id STRING,
    order_date DATE,
    customer_key BIGINT,
    net_amount DECIMAL(18,4)
)
USING DELTA
PARTITIONED BY (order_date);

-- Hive-style partitioning for compatibility
CREATE TABLE gold_sales_retail.fact_sales_historical (
    transaction_key BIGINT,
    transaction_id STRING,
    customer_key BIGINT,
    net_amount DECIMAL(18,4),
    year INT,
    month INT
)
USING DELTA
PARTITIONED BY (year, month);
```

**Partition Limitations**:
- Can cause small file problems with high cardinality
- Requires partition values at write time
- Less flexible for evolving query patterns
- Partition pruning only benefits single column

### Z-ORDER Optimization (Legacy)

For tables created before Liquid Clustering, use Z-ORDER:

```sql
-- Z-ORDER clusters data by multiple columns
OPTIMIZE gold_sales_retail.fact_sales_transaction
ZORDER BY (customer_key, product_key, order_date);

-- Limitation: Max 4 columns recommended
-- Limitation: Requires periodic manual execution
-- Limitation: More expensive than Liquid Clustering

-- Migrate to Liquid Clustering:
ALTER TABLE gold_sales_retail.fact_sales_transaction
CLUSTER BY (order_date, customer_key, product_key);
```

### Migration Path: Partitioned → Liquid Clustered

```sql
-- Step 1: Create new table with Liquid Clustering
CREATE TABLE gold_sales_retail.fact_sales_transaction_v2 (
    transaction_key BIGINT,
    transaction_id STRING,
    order_date DATE,
    customer_key BIGINT,
    net_amount DECIMAL(18,4)
)
USING DELTA
CLUSTER BY (order_date, customer_key);

-- Step 2: Copy data (maintains clustering)
INSERT INTO gold_sales_retail.fact_sales_transaction_v2
SELECT * FROM gold_sales_retail.fact_sales_transaction;

-- Step 3: Swap tables (use Unity Catalog ALTER TABLE RENAME)
ALTER TABLE gold_sales_retail.fact_sales_transaction 
RENAME TO gold_sales_retail.fact_sales_transaction_old;

ALTER TABLE gold_sales_retail.fact_sales_transaction_v2 
RENAME TO gold_sales_retail.fact_sales_transaction;

-- Step 4: Update downstream references and drop old table
DROP TABLE gold_sales_retail.fact_sales_transaction_old;
```

### Optimization Scheduling

```python
# Schedule regular OPTIMIZE jobs for Liquid Clustered tables
from pyspark.sql import SparkSession

def optimize_gold_tables():
    """Optimize all gold layer tables with Liquid Clustering"""
    tables = [
        "gold_sales_retail.fact_sales_transaction",
        "gold_customer_360.dim_customer",
        "gold_sales_retail.agg_sales_customer_monthly"
    ]
    
    for table in tables:
        print(f"Optimizing {table}...")
        spark.sql(f"OPTIMIZE {table}")
        
        # Optional: VACUUM to remove old files
        spark.sql(f"VACUUM {table} RETAIN 168 HOURS")  # 7 days
        
    print("Optimization complete")

# Run daily via Databricks workflow/job
```

**Optimization Best Practices**:
- Run OPTIMIZE after large data loads
- Schedule regular optimizations (daily/weekly based on write volume)
- Use VACUUM to reclaim storage from old file versions
- Monitor file sizes: aim for 128MB-1GB per file
- Leverage auto-optimize for write-heavy tables:
  ```sql
  ALTER TABLE fact_sales_transaction 
  SET TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true');
  ```

## Data Quality and Constraints

### Primary Key Enforcement

**Note**: Databricks Delta Lake supports PRIMARY KEY and FOREIGN KEY as metadata for documentation and query optimization, but **does not enforce** them at runtime (as of DBR 13.x).

```sql
-- Define primary key (informational, not enforced)
CREATE TABLE gold_customer_360.dim_customer (
    customer_key BIGINT NOT NULL,
    customer_id STRING NOT NULL,
    customer_name STRING,
    effective_date DATE,
    is_current BOOLEAN,
    PRIMARY KEY (customer_key)
);

-- Composite key for fact tables
CREATE TABLE gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT NOT NULL,
    transaction_id STRING NOT NULL,
    order_date DATE,
    PRIMARY KEY (transaction_key, transaction_id)
);
```

**Enforce Uniqueness with Delta Constraints**:
```sql
-- Use CHECK constraints for actual enforcement
ALTER TABLE gold_customer_360.dim_customer
ADD CONSTRAINT unique_customer_key 
CHECK (customer_key IS NOT NULL);

-- For uniqueness, use MERGE logic to prevent duplicates
MERGE INTO gold_customer_360.dim_customer AS target
USING source_data AS source
ON target.customer_key = source.customer_key
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- Or use deduplication in INSERT
INSERT INTO gold_customer_360.dim_customer
SELECT DISTINCT customer_key, customer_id, customer_name, ...
FROM staging.customer_data
WHERE customer_key NOT IN (SELECT customer_key FROM gold_customer_360.dim_customer);
```

### Foreign Key Relationships

```sql
-- Define foreign keys for documentation (not enforced)
CREATE TABLE gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT PRIMARY KEY,
    customer_key BIGINT,
    product_key BIGINT,
    FOREIGN KEY (customer_key) REFERENCES gold_customer_360.dim_customer(customer_key),
    FOREIGN KEY (product_key) REFERENCES gold_product_catalog.dim_product(product_key)
);

-- Alternative: Document in table properties
CREATE TABLE gold_sales_retail.fact_sales_transaction (...)
TBLPROPERTIES (
    'foreign_keys' = 'customer_key->dim_customer, product_key->dim_product'
);
```

**Implement Referential Integrity Validation**:
```python
# Validate foreign keys in ETL pipeline
from pyspark.sql import functions as F

# Check for orphaned records
fact_df = spark.table("gold_sales_retail.fact_sales_transaction")
dim_customer = spark.table("gold_customer_360.dim_customer")

orphaned_customers = (
    fact_df
    .select("customer_key")
    .distinct()
    .join(dim_customer, "customer_key", "left_anti")
)

orphan_count = orphaned_customers.count()
if orphan_count > 0:
    print(f"WARNING: {orphan_count} orphaned customer keys found")
    # Log to data quality table or raise exception
    raise ValueError(f"Foreign key violation: {orphan_count} invalid customer_keys")
```

### Check Constraints

Delta Lake **does enforce** CHECK constraints:

```sql
-- Enforce data validity rules
ALTER TABLE gold_sales_retail.fact_sales_transaction 
ADD CONSTRAINT check_quantity_positive 
CHECK (quantity > 0);

ALTER TABLE gold_sales_retail.fact_sales_transaction 
ADD CONSTRAINT check_amount_valid 
CHECK (net_amount >= 0);

ALTER TABLE gold_sales_retail.fact_sales_transaction
ADD CONSTRAINT check_dates_logical
CHECK (ship_date >= order_date);

-- View constraints
SHOW TBLPROPERTIES gold_sales_retail.fact_sales_transaction;

-- Drop constraint if needed
ALTER TABLE gold_sales_retail.fact_sales_transaction 
DROP CONSTRAINT check_quantity_positive;
```

### Not Null Constraints

```sql
-- NOT NULL is enforced in Delta Lake
CREATE TABLE gold_customer_360.dim_customer (
    customer_key BIGINT NOT NULL,
    customer_id STRING NOT NULL,
    customer_name STRING NOT NULL,
    email STRING,  -- Nullable
    effective_date DATE NOT NULL
)
USING DELTA;

-- Add NOT NULL to existing column (requires rewrite)
ALTER TABLE gold_customer_360.dim_customer
ALTER COLUMN customer_name SET NOT NULL;
```

### Data Quality Framework

Since Delta doesn't enforce FK constraints, implement a comprehensive DQ framework:

```python
# Data Quality validation framework
from pyspark.sql import functions as F
from delta.tables import DeltaTable

def validate_gold_layer_quality(table_name, validations):
    """
    Run data quality checks on gold layer table
    
    Args:
        table_name: Full table name (catalog.schema.table)
        validations: Dict of validation rules
    """
    df = spark.table(table_name)
    results = []
    
    for validation_name, validation_func in validations.items():
        try:
            passed, message = validation_func(df)
            results.append({
                "table": table_name,
                "validation": validation_name,
                "passed": passed,
                "message": message,
                "timestamp": datetime.now()
            })
        except Exception as e:
            results.append({
                "table": table_name,
                "validation": validation_name,
                "passed": False,
                "message": str(e),
                "timestamp": datetime.now()
            })
    
    # Log results
    results_df = spark.createDataFrame(results)
    results_df.write.format("delta").mode("append").saveAsTable("audit.data_quality_checks")
    
    # Raise exception if critical validations failed
    critical_failures = [r for r in results if not r["passed"] and "CRITICAL" in r["validation"]]
    if critical_failures:
        raise ValueError(f"Critical data quality checks failed: {critical_failures}")
    
    return results

# Example validations for fact table
fact_sales_validations = {
    "CRITICAL_no_null_keys": lambda df: (
        df.filter(F.col("customer_key").isNull() | F.col("product_key").isNull()).count() == 0,
        "All foreign keys must be populated"
    ),
    "CRITICAL_valid_foreign_keys": lambda df: (
        df.join(
            spark.table("gold_customer_360.dim_customer"),
            "customer_key",
            "left_anti"
        ).count() == 0,
        "All customer_keys must exist in dimension"
    ),
    "WARNING_reasonable_amounts": lambda df: (
        df.filter((F.col("net_amount") < 0) | (F.col("net_amount") > 1000000)).count() < df.count() * 0.01,
        "Less than 1% of amounts outside reasonable range"
    ),
    "INFO_completeness": lambda df: (
        True,
        f"Row count: {df.count()}, Null customer_name: {df.filter(F.col('customer_name').isNull()).count()}"
    )
}

# Run validations
validate_gold_layer_quality("gold_sales_retail.fact_sales_transaction", fact_sales_validations)
```

### Databricks Expectations (Alternative)

```sql
-- Use Delta Lake Expectations for automatic validation
ALTER TABLE gold_sales_retail.fact_sales_transaction
ADD CONSTRAINT valid_quantity 
EXPECT (quantity > 0) ON VIOLATION DROP ROW;

-- Or FAIL the entire transaction
ALTER TABLE gold_sales_retail.fact_sales_transaction
ADD CONSTRAINT valid_dates 
EXPECT (ship_date >= order_date) ON VIOLATION FAIL UPDATE;

-- Track violations without blocking
ALTER TABLE gold_sales_retail.fact_sales_transaction
ADD CONSTRAINT reasonable_amount 
EXPECT (net_amount BETWEEN 0 AND 1000000) ON VIOLATION LOG ROW;
``` 
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

### 1. Delta Lake Optimization

**Regular OPTIMIZE Command**:
```sql
-- Compact small files and optimize file layout
OPTIMIZE gold_sales_retail.fact_sales_transaction;

-- With Z-ORDER for legacy tables (pre-Liquid Clustering)
OPTIMIZE gold_sales_retail.fact_sales_transaction
ZORDER BY (customer_key, product_key, order_date);

-- Check file statistics
DESCRIBE DETAIL gold_sales_retail.fact_sales_transaction;
```

**Auto-Optimize for Write-Heavy Tables**:
```sql
-- Enable auto-optimize for automatic compaction
ALTER TABLE gold_sales_retail.fact_sales_transaction 
SET TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',  -- Optimize during writes
    'delta.autoOptimize.autoCompact' = 'true'     -- Auto-compact after writes
);

-- Recommended for:
-- - Tables with frequent small writes
-- - Streaming writes
-- - Tables with many concurrent writers
```

**VACUUM for Storage Reclamation**:
```sql
-- Remove old file versions beyond retention period
VACUUM gold_sales_retail.fact_sales_transaction RETAIN 168 HOURS;  -- 7 days

-- Check space to be reclaimed (DRY RUN)
VACUUM gold_sales_retail.fact_sales_transaction DRY RUN;

-- Configure default retention
ALTER TABLE gold_sales_retail.fact_sales_transaction
SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = 'interval 7 days');
```

### 2. Photon Engine Acceleration

**Enable at Cluster Level** (no code changes required):
- Cluster Configuration → Runtime: Select Photon-enabled runtime
- Photon automatically accelerates: Scans, Joins, Aggregations, Window Functions

**Optimize for Photon**:
```python
# Write data with optimal file sizes for Photon (128MB-1GB)
df.write \
    .format("delta") \
    .option("maxRecordsPerFile", 2000000) \  # Adjust based on row size
    .mode("append") \
    .saveAsTable("gold_sales_retail.fact_sales_transaction")

# Partition writing for large datasets
df.repartition(100).write.format("delta").saveAsTable("...")
```

### 3. Materialized Views and Streaming Tables

**Materialized Views** (Auto-refresh):
```sql
-- Self-maintaining aggregated view
CREATE OR REFRESH MATERIALIZED VIEW gold_sales_retail.mv_customer_sales_summary
AS SELECT 
    c.customer_key,
    c.customer_name,
    c.customer_segment,
    COUNT(DISTINCT s.transaction_id) AS transaction_count,
    SUM(s.net_amount) AS total_revenue,
    AVG(s.net_amount) AS avg_transaction_value,
    MAX(s.order_date) AS last_order_date
FROM gold_customer_360.dim_customer c
LEFT JOIN gold_sales_retail.fact_sales_transaction s ON c.customer_key = s.customer_key
GROUP BY c.customer_key, c.customer_name, c.customer_segment;

-- Materialized view refreshes automatically on underlying data changes
-- No manual refresh needed

-- Query materialized view (much faster than base query)
SELECT * FROM gold_sales_retail.mv_customer_sales_summary
WHERE customer_segment = 'Enterprise';
```

**Streaming Tables** for Real-Time:
```sql
-- Create streaming table for continuous processing
CREATE OR REFRESH STREAMING TABLE gold_sales_retail.fact_sales_realtime
AS SELECT 
    transaction_id,
    customer_key,
    product_key,
    order_date,
    net_amount,
    current_timestamp() AS processed_timestamp
FROM STREAM(silver.sales_cdc);

-- Automatically processes new data as it arrives
```

### 4. Caching Strategies

**Delta Caching (Automatic)**:
```python
# Delta cache automatically caches frequently accessed data on SSD
# Enable at cluster level: "Use Delta Lake Caching"

# Explicitly cache hot tables
spark.sql("CACHE SELECT * FROM gold_sales_retail.fact_sales_transaction WHERE order_date >= current_date() - 7")

# Check cache status
spark.sql("SHOW TABLES IN gold_sales_retail").show()
```

**Spark Caching for Iterative Queries**:
```python
# Cache DataFrame for reuse in session
df = spark.table("gold_sales_retail.fact_sales_transaction")
df.cache()

# Use cached df multiple times
daily_summary = df.groupBy("order_date").sum("net_amount")
customer_summary = df.groupBy("customer_key").count()

# Uncache when done
df.unpersist()
```

### 5. Query Performance Best Practices

**Predicate Pushdown**:
```sql
-- Good: Filter pushed to file scan level
SELECT * FROM gold_sales_retail.fact_sales_transaction
WHERE order_date >= '2024-01-01'  -- Uses Liquid Clustering
  AND customer_key IN (1001, 1002, 1003);

-- Avoid: Complex WHERE clauses that prevent pushdown
SELECT * FROM gold_sales_retail.fact_sales_transaction
WHERE YEAR(order_date) = 2024;  -- Use: WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'
```

**Column Pruning**:
```sql
-- Good: Select only needed columns
SELECT customer_key, net_amount, order_date
FROM gold_sales_retail.fact_sales_transaction;

-- Avoid: SELECT * reads all columns from parquet files
```

**Broadcast Joins for Small Dimensions**:
```python
from pyspark.sql import functions as F

# Explicitly broadcast small dimension (<10GB)
fact_df = spark.table("gold_sales_retail.fact_sales_transaction")
dim_customer = spark.table("gold_customer_360.dim_customer")

result = fact_df.join(
    F.broadcast(dim_customer),
    "customer_key"
)

# Databricks Auto-Broadcast: Tables <100MB automatically broadcast
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 104857600)  # 100MB
```

**Avoid Shuffle-Heavy Operations**:
```python
# Use Liquid Clustering to co-locate frequently joined data
# Reduces shuffle when joining on clustered columns

# Example: Both tables clustered by customer_key
# CREATE TABLE fact_sales (...) CLUSTER BY (customer_key, ...)
# CREATE TABLE dim_customer (...) CLUSTER BY (customer_key, ...)

# Join on customer_key benefits from co-location
result = spark.sql("""
    SELECT f.*, c.customer_name
    FROM gold_sales_retail.fact_sales_transaction f
    JOIN gold_customer_360.dim_customer c ON f.customer_key = c.customer_key
""")
```

### 6. Statistics Collection

```sql
-- Collect table statistics for query optimization
ANALYZE TABLE gold_sales_retail.fact_sales_transaction 
COMPUTE STATISTICS FOR ALL COLUMNS;

-- For specific columns
ANALYZE TABLE gold_customer_360.dim_customer
COMPUTE STATISTICS FOR COLUMNS customer_key, customer_segment;

-- Enable automatic statistics collection
SET spark.sql.statistics.auto.collection.enabled = true;

-- View statistics
DESCRIBE EXTENDED gold_sales_retail.fact_sales_transaction;
```

### 7. Databricks SQL Warehouse Optimization

**For BI/Reporting Workloads**:
```sql
-- Use SQL Warehouse (Serverless) for optimal performance
-- - Automatic scaling
-- - Instant cluster start
-- - Photon acceleration included
-- - Result caching
-- - Query history and profiling

-- Configure warehouse size based on concurrency:
-- X-Small: 1-5 concurrent users
-- Small: 5-10 concurrent users  
-- Medium: 10-20 concurrent users
-- Large: 20-50 concurrent users
```

### 8. Predictive Optimization (Auto)

```sql
-- Enable Predictive Optimization for automatic maintenance
ALTER TABLE gold_sales_retail.fact_sales_transaction
SET TBLPROPERTIES ('delta.autoOptimize.autoCompact' = 'true');

-- Databricks automatically:
-- ✓ Runs OPTIMIZE during low usage
-- ✓ Collects statistics
-- ✓ Compacts small files
-- ✓ Maintains Liquid Clustering
-- ✓ No manual scheduling needed

-- Check optimization history
DESCRIBE HISTORY gold_sales_retail.fact_sales_transaction;
```

### 9. Performance Monitoring

```sql
-- Query profile in Databricks SQL
-- Automatically shows: scan times, shuffle sizes, spills, etc.

-- Use query tags for tracking
SET spark.databricks.queryWatchdog.queryTag = 'daily_sales_report';

-- Monitor table performance
SELECT 
    table_name,
    num_files,
    size_in_bytes,
    num_files / NULLIF(size_in_bytes/1024/1024/1024, 0) AS files_per_gb
FROM (
    DESCRIBE DETAIL gold_sales_retail.fact_sales_transaction
)
WHERE num_files / NULLIF(size_in_bytes/1024/1024/1024, 0) > 100  -- Alert if > 100 files per GB
```

### 10. Performance Checklist

Before production deployment:

- [ ] Liquid Clustering enabled with appropriate columns
- [ ] Auto-optimize configured for write-heavy tables
- [ ] VACUUM scheduled for storage management
- [ ] Photon enabled on compute clusters
- [ ] Materialized views created for common aggregations
- [ ] Statistics collected on all tables
- [ ] Query performance tested with realistic data volumes
- [ ] Broadcast joins configured for small dimensions
- [ ] File sizes optimized (128MB-1GB range)
- [ ] Monitoring and alerting configured

## Documentation Standards

### Table Documentation Template

```sql
-- Unity Catalog supports comprehensive metadata
CREATE TABLE production.gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT COMMENT 'Surrogate key - unique identifier',
    transaction_id STRING COMMENT 'Business key from source POS system',
    order_date DATE COMMENT 'Date order was placed',
    customer_key BIGINT COMMENT 'Foreign key to dim_customer',
    product_key BIGINT COMMENT 'Foreign key to dim_product',
    store_key INT COMMENT 'Foreign key to dim_store',
    quantity DECIMAL(18,4) COMMENT 'Quantity sold',
    net_amount DECIMAL(18,4) COMMENT 'Net sales amount after discounts and taxes'
)
USING DELTA
COMMENT 'Daily retail sales transactions from all channels. Grain: One row per line item on each sales order.'
TBLPROPERTIES (
    'business_owner' = 'jane.smith@company.com',
    'data_steward' = 'sales-analytics@company.com',
    'source_systems' = 'POS, E-commerce, B2B Portal',
    'load_frequency' = 'Near real-time (15-minute intervals)',
    'retention_period' = '7 years',
    'sas_migration_source' = 'daily_sales_fact.sas, sales_detail.sas',
    'data_classification' = 'Confidential',
    'last_modified' = '2024-01-15',
    'quality_score' = '0.99',
    'delta.enableChangeDataFeed' = 'true',
    'delta.autoOptimize.autoCompact' = 'true'
)
CLUSTER BY (order_date, customer_key, product_key);

-- Tag sensitive columns for governance
ALTER TABLE production.gold_sales_retail.fact_sales_transaction
ALTER COLUMN customer_key SET TAGS ('pii_category' = 'identifier', 'sensitivity' = 'high');

-- View all metadata
DESCRIBE EXTENDED production.gold_sales_retail.fact_sales_transaction;

-- View tags
SELECT * FROM system.information_schema.column_tags
WHERE table_catalog = 'production'
  AND table_schema = 'gold_sales_retail'
  AND table_name = 'fact_sales_transaction';
```

### Column-Level Documentation

```sql
-- Add detailed column comments
COMMENT ON COLUMN production.gold_sales_retail.fact_sales_transaction.net_amount IS
'Net sales amount calculated as: extended_price - discount_amount + tax_amount.
Currency: USD. Excludes returns and refunds (tracked separately).
Business Rule: Must be >= 0 for valid transactions.
Source: POS.transaction_total, Ecommerce.order_amount';

-- Update table properties
ALTER TABLE production.gold_sales_retail.fact_sales_transaction
SET TBLPROPERTIES (
    'related_tables' = 'dim_customer, dim_product, dim_store, dim_date',
    'downstream_consumers' = 'PowerBI Sales Dashboard, Tableau Executive Reports',
    'sla_data_freshness' = '30 minutes'
);
```

## Quality Checklist

Before promoting to production, verify:

### Data Modeling
- [ ] Grain explicitly defined and enforced
- [ ] Primary key defined (metadata, uniqueness validated in ETL)
- [ ] Foreign keys documented and validated in data quality checks
- [ ] Naming conventions followed (tables, columns, Unity Catalog structure)
- [ ] SCD strategy implemented correctly (with Delta MERGE)

### Delta Lake Configuration
- [ ] Table format is DELTA (USING DELTA)
- [ ] Unity Catalog three-level namespace used (catalog.schema.table)
- [ ] Change Data Feed enabled if needed (delta.enableChangeDataFeed)
- [ ] File retention configured (delta.deletedFileRetentionDuration)
- [ ] Log retention configured (delta.logRetentionDuration)
- [ ] Auto-optimize configured for write-heavy tables

### Performance Optimization
- [ ] Liquid Clustering configured with appropriate columns
- [ ] OPTIMIZE run and scheduled for ongoing maintenance
- [ ] VACUUM scheduled for storage management
- [ ] Table statistics collected (ANALYZE TABLE)
- [ ] File sizes optimized (128MB-1GB range verified)
- [ ] Photon-enabled clusters used for queries
- [ ] Materialized views created for common aggregations
- [ ] Performance tested with realistic data volumes

### Documentation & Metadata
- [ ] Table comment describes purpose and grain
- [ ] Column comments added for key columns
- [ ] TBLPROPERTIES populated (owner, classification, source, etc.)
- [ ] Sensitive columns tagged in Unity Catalog
- [ ] Data lineage documented and validated
- [ ] SAS source programs referenced in metadata
- [ ] Related tables documented

### Security & Governance
- [ ] Data classification assigned
- [ ] Unity Catalog grants configured
- [ ] Row-level security policies created (if applicable)
- [ ] Column-level security configured (if applicable)
- [ ] Sensitive data masked or tokenized
- [ ] Audit columns populated (created_timestamp, batch_id, etc.)

### Data Quality
- [ ] CHECK constraints defined for business rules
- [ ] NOT NULL constraints on required columns
- [ ] Data quality validations implemented in ETL
- [ ] Reconciliation against SAS validated (all 3 levels)
- [ ] Discrepancies investigated and resolved
- [ ] Expected data patterns validated (volume, distribution)

### Operational Readiness
- [ ] Load patterns tested (incremental, full refresh)
- [ ] Time travel tested (VERSION AS OF, TIMESTAMP AS OF)
- [ ] Recovery procedures tested (RESTORE TABLE)
- [ ] Monitoring and alerting configured
- [ ] ETL job logging and error handling implemented
- [ ] Downstream consumers identified and validated
- [ ] Rollback plan documented and tested

### Compliance
- [ ] Retention policy configured and enforced
- [ ] GDPR/privacy requirements addressed
- [ ] SOX controls implemented (if applicable)
- [ ] Access review process established
- [ ] Change control procedures followed

---

**Document Control**

| Attribute | Value |
|-----------|-------|
| Version | 2.0 (Databricks Edition) |
| Last Updated | January 2026 |
| Owner | Data Architecture Team |
| Target Platform | Databricks Lakehouse |
| Review Date | April 2026 |
