# Databricks Gold Layer Naming Conventions

## Overview

This document establishes comprehensive naming standards for all gold layer objects in the Databricks Lakehouse Platform. Consistent naming is critical for discoverability, maintainability, and Unity Catalog governance.

**Platform Context**: All standards are optimized for Databricks Unity Catalog's three-level namespace (catalog.schema.table) and Delta Lake table management.

**SAS Migration Context**: These conventions support the SAS decommissioning program by providing clear, business-centric names that map cleanly from SAS datasets while establishing modern data platform standards.

## Guiding Principles

1. **Business-Centric**: Names reflect business concepts, not technical implementation
2. **Unity Catalog Aligned**: Leverage three-level namespace for organization
3. **Self-Documenting**: Names convey purpose without extensive documentation
4. **Consistent**: Predictable patterns across all objects
5. **Future-Proof**: Avoid time-bound or technology-specific references
6. **Lowercase with Underscores**: snake_case for all database objects
7. **SAS Traceability**: Document SAS source in metadata, not names

## Unity Catalog Namespace Structure

### Three-Level Hierarchy

```
catalog.schema.table
   │      │      │
   │      │      └─ Table name (this document)
   │      └─ Schema name (business domain)
   └─ Catalog name (environment)
```

### Catalog Naming

**Format**: `<environment>` or `<org>_<environment>`

**Standard Catalogs**:
```
production              # Production gold layer
staging                 # Pre-production validation
development             # Developer sandboxes
analytics_production    # Analytics-specific catalog
```

**Usage**:
```sql
-- Always specify catalog in production code
SELECT * FROM production.gold_sales_retail.fact_sales_transaction;

-- Set default catalog in session
USE CATALOG production;
SELECT * FROM gold_sales_retail.fact_sales_transaction;
```

### Schema Naming

**Format**: `gold_<domain>_<subject_area>`

**Components**:
- **gold**: Prefix identifying medallion layer
- **domain**: Business domain or functional area
- **subject_area**: Specific subject within domain (optional for simple domains)

**Examples**:
```sql
-- Finance domain
gold_finance_accounting
gold_finance_treasury
gold_finance_payables
gold_finance_receivables

-- Sales domain
gold_sales_retail
gold_sales_wholesale
gold_sales_ecommerce

-- Customer domain
gold_customer_360
gold_customer_analytics
gold_customer_service

-- Operations domain
gold_operations_manufacturing
gold_operations_logistics
gold_operations_quality

-- Human Resources
gold_hr_compensation
gold_hr_talent
gold_hr_workforce

-- Product domain
gold_product_catalog
gold_product_pricing

-- Common/Shared
gold_common_reference        # Shared reference data
gold_common_dimensions       # Conformed dimensions
```

**Schema Creation with Metadata**:
```sql
CREATE SCHEMA IF NOT EXISTS production.gold_sales_retail
COMMENT 'Retail sales business-ready data including transactions, customers, and products'
WITH DBPROPERTIES (
    'owner' = 'sales_operations_team',
    'business_domain' = 'sales',
    'data_classification' = 'confidential',
    'sas_migration_wave' = 'wave_2',
    'created_date' = '2024-01-15'
);
```

## Table Naming Standards

### General Pattern

**Format**: `<entity_type>_<business_entity>_<qualifier>`

**Components**:
- **entity_type**: Table type prefix (dim_, fact_, agg_, etc.)
- **business_entity**: Core business concept
- **qualifier**: Additional specificity (optional)

### Entity Type Prefixes

| Prefix | Type | Purpose | Grain |
|--------|------|---------|-------|
| `dim_` | Dimension | Master/reference data | One row per entity version |
| `fact_` | Fact | Transactions or events | One row per business event |
| `agg_` | Aggregate | Pre-aggregated facts | One row per aggregation level |
| `bridge_` | Bridge | M:N relationships | One row per relationship pair |
| `snapshot_` | Snapshot | Point-in-time state | One row per entity per period |
| `ref_` | Reference | Static lookup data | One row per code value |
| `metric_` | Metric | Calculated KPIs | One row per metric instance |

### Dimension Table Naming

**Format**: `dim_<entity>_<type_qualifier>`

**Standard Dimensions**:
```sql
-- Core dimensions
dim_customer                   # Customer master
dim_customer_scd2              # With Type 2 history (if needed to distinguish)
dim_product                    # Product master
dim_employee                   # Employee information
dim_store                      # Store locations
dim_supplier                   # Supplier master

-- Date dimensions
dim_date                       # Calendar date dimension
dim_date_fiscal                # Fiscal calendar
dim_time                       # Time of day dimension

-- Hierarchical dimensions
dim_geography                  # Geographic hierarchy
dim_account                    # Chart of accounts
dim_organization               # Org structure

-- Classification dimensions
dim_product_category           # Product categories
dim_customer_segment           # Customer segments
dim_channel                    # Sales channels
```

**SCD Type Indicators** (optional suffix):
```sql
dim_customer                   # Type 1 (current state only)
dim_customer_scd2              # Type 2 (full history) - only if both types exist
dim_price_scd3                 # Type 3 (current + prior)
```

**Creation Example**:
```sql
CREATE TABLE production.gold_customer_360.dim_customer (
    customer_key BIGINT NOT NULL COMMENT 'Surrogate key',
    customer_id STRING NOT NULL COMMENT 'Business key from source',
    customer_name STRING COMMENT 'Customer full name',
    customer_segment STRING COMMENT 'Customer segmentation',
    effective_date DATE NOT NULL COMMENT 'SCD2 effective date',
    expiration_date DATE COMMENT 'SCD2 expiration date - NULL for current',
    is_current BOOLEAN COMMENT 'Current version flag',
    row_hash STRING COMMENT 'Hash for change detection',
    created_timestamp TIMESTAMP COMMENT 'Record creation time',
    source_system STRING COMMENT 'Source system identifier',
    PRIMARY KEY (customer_key)
)
USING DELTA
COMMENT 'Customer master dimension with Type 2 slowly changing dimension logic'
TBLPROPERTIES (
    'delta.enableChangeDataFeed' = 'true',
    'sas_source_program' = 'customer_master_load.sas',
    'sas_source_dataset' = 'crm.customer_master',
    'business_owner' = 'crm-team@company.com'
)
CLUSTER BY (customer_id, customer_segment);
```

### Fact Table Naming

**Format**: `fact_<business_process>_<granularity>`

**Transaction Facts**:
```sql
fact_sales_transaction         # Individual sales transactions
fact_order_line                # Order line items
fact_payment                   # Payment transactions
fact_shipment                  # Shipment events
fact_inventory_movement        # Inventory transactions
fact_gl_transaction            # General ledger entries
fact_service_ticket            # Service interactions
fact_web_clickstream           # Website events
fact_call_detail               # Call center details
```

**Periodic Snapshot Facts**:
```sql
fact_account_balance_daily     # Daily account balances
fact_inventory_level_daily     # Daily inventory positions
fact_employee_headcount_monthly # Monthly headcount
fact_pipeline_weekly           # Weekly sales pipeline
fact_portfolio_quarterly       # Quarterly portfolio positions
```

**Accumulating Snapshot Facts**:
```sql
fact_order_fulfillment         # Order lifecycle
fact_claim_processing          # Insurance claim lifecycle
fact_loan_origination          # Loan application to funding
fact_project_lifecycle         # Project start to completion
fact_employee_hiring           # Requisition to onboarding
```

**Creation Example**:
```sql
CREATE TABLE production.gold_sales_retail.fact_sales_transaction (
    transaction_key BIGINT NOT NULL COMMENT 'Surrogate key',
    transaction_id STRING NOT NULL COMMENT 'Business key',
    order_date_key INT NOT NULL COMMENT 'FK to dim_date',
    customer_key BIGINT NOT NULL COMMENT 'FK to dim_customer',
    product_key BIGINT NOT NULL COMMENT 'FK to dim_product',
    store_key INT NOT NULL COMMENT 'FK to dim_store',
    -- Measures
    quantity DECIMAL(18,4) COMMENT 'Quantity sold',
    unit_price DECIMAL(18,4) COMMENT 'Unit selling price',
    extended_price DECIMAL(18,4) COMMENT 'Extended price before discounts',
    discount_amount DECIMAL(18,4) COMMENT 'Total discounts applied',
    tax_amount DECIMAL(18,4) COMMENT 'Sales tax amount',
    net_amount DECIMAL(18,4) COMMENT 'Net sales amount',
    unit_cost DECIMAL(18,4) COMMENT 'Unit cost',
    -- Degenerate dimensions
    order_number STRING COMMENT 'Order identifier',
    invoice_number STRING COMMENT 'Invoice identifier',
    -- Audit
    created_timestamp TIMESTAMP COMMENT 'ETL load timestamp',
    batch_id STRING COMMENT 'ETL batch identifier',
    PRIMARY KEY (transaction_key)
)
USING DELTA
COMMENT 'Daily retail sales transactions. Grain: One row per line item on each sales order.'
TBLPROPERTIES (
    'delta.enableChangeDataFeed' = 'true',
    'sas_source_program' = 'daily_sales_load.sas',
    'sas_source_dataset' = 'sales.daily_transactions'
)
CLUSTER BY (order_date_key, customer_key, product_key);
```

### Aggregate Table Naming

**Format**: `agg_<fact_name>_<aggregation_level>_<time_grain>`

**Examples**:
```sql
agg_sales_customer_daily       # Sales by customer by day
agg_sales_customer_monthly     # Sales by customer by month
agg_sales_product_weekly       # Sales by product by week
agg_revenue_region_quarterly   # Revenue by region by quarter
agg_inventory_warehouse_daily  # Inventory by warehouse by day
agg_orders_channel_yearly      # Orders by channel by year
agg_claims_status_monthly      # Claims by status by month
```

**Multiple Dimensions**:
```sql
agg_sales_customer_product_monthly      # Customer + Product + Month
agg_revenue_region_channel_quarterly    # Region + Channel + Quarter
```

**Creation Example**:
```sql
CREATE TABLE production.gold_sales_retail.agg_sales_customer_monthly (
    month_date_key INT NOT NULL COMMENT 'First day of month',
    customer_key BIGINT NOT NULL COMMENT 'FK to dim_customer',
    -- Aggregated measures
    transaction_count INT COMMENT 'Number of transactions',
    order_count INT COMMENT 'Number of distinct orders',
    quantity_sum DECIMAL(18,4) COMMENT 'Total quantity',
    revenue_sum DECIMAL(18,4) COMMENT 'Total revenue',
    cost_sum DECIMAL(18,4) COMMENT 'Total cost',
    profit_sum DECIMAL(18,4) COMMENT 'Total profit',
    avg_transaction_value DECIMAL(18,4) COMMENT 'Average transaction amount',
    -- Metadata
    aggregation_timestamp TIMESTAMP COMMENT 'When aggregation was computed',
    source_fact_table STRING COMMENT 'Source fact table name',
    PRIMARY KEY (month_date_key, customer_key)
)
USING DELTA
COMMENT 'Monthly sales aggregated by customer. Grain: One row per customer per month.'
TBLPROPERTIES (
    'source_table' = 'fact_sales_transaction',
    'refresh_frequency' = 'daily'
)
CLUSTER BY (month_date_key, customer_key);
```

### Bridge Table Naming

**Format**: `bridge_<relationship_description>`

**Examples**:
```sql
bridge_account_hierarchy       # Account parent-child relationships
bridge_employee_manager        # Employee-manager hierarchy
bridge_product_category        # Product multi-level categories
bridge_customer_household      # Customer household groupings
bridge_store_region            # Store to region mapping
bridge_product_bundle          # Product bundle components
```

**Creation Example**:
```sql
CREATE TABLE production.gold_product_catalog.bridge_product_bundle (
    bundle_product_key BIGINT NOT NULL COMMENT 'FK to dim_product (bundle)',
    component_product_key BIGINT NOT NULL COMMENT 'FK to dim_product (component)',
    quantity DECIMAL(18,4) COMMENT 'Quantity of component in bundle',
    allocation_percent DECIMAL(5,4) COMMENT 'Revenue allocation percentage',
    effective_date DATE COMMENT 'When bundle became effective',
    expiration_date DATE COMMENT 'When bundle expires',
    PRIMARY KEY (bundle_product_key, component_product_key, effective_date)
)
USING DELTA
COMMENT 'Product bundle composition for revenue allocation. Grain: One row per bundle-component pair per effective period.'
CLUSTER BY (bundle_product_key, effective_date);
```

### Reference Table Naming

**Format**: `ref_<reference_type>`

**Examples**:
```sql
ref_country_code               # ISO country codes
ref_currency_code              # Currency codes and conversion
ref_status_code                # Status code lookups
ref_transaction_type           # Transaction type classifications
ref_industry_code              # Industry classification (SIC, NAICS)
ref_payment_method             # Payment method types
ref_promotion_type             # Promotion type codes
```

**Creation Example**:
```sql
CREATE TABLE production.gold_common_reference.ref_country_code (
    country_code STRING NOT NULL COMMENT 'ISO 3166-1 alpha-2 code',
    country_name STRING COMMENT 'Official country name',
    country_name_long STRING COMMENT 'Full country name',
    iso3_code STRING COMMENT 'ISO 3166-1 alpha-3 code',
    numeric_code STRING COMMENT 'ISO 3166-1 numeric code',
    region STRING COMMENT 'Geographic region',
    sub_region STRING COMMENT 'Geographic sub-region',
    is_active BOOLEAN COMMENT 'Active flag',
    PRIMARY KEY (country_code)
)
USING DELTA
COMMENT 'ISO country code reference data'
TBLPROPERTIES (
    'data_source' = 'ISO 3166-1 standard',
    'update_frequency' = 'as needed'
);
```

### Metric Table Naming

**Format**: `metric_<metric_category>_<specificity>`

**Examples**:
```sql
metric_customer_lifetime_value # CLV calculations
metric_customer_churn          # Churn predictions
metric_product_margin          # Product profitability
metric_sales_target            # Sales goals and targets
metric_inventory_health        # Inventory KPIs
metric_employee_performance    # Performance scores
metric_service_quality         # Service level metrics
```

**Creation Example**:
```sql
CREATE TABLE production.gold_customer_analytics.metric_customer_lifetime_value (
    customer_key BIGINT NOT NULL COMMENT 'FK to dim_customer',
    calculation_date DATE NOT NULL COMMENT 'Date of calculation',
    -- Metrics
    historical_revenue DECIMAL(18,4) COMMENT 'Total historical revenue',
    historical_profit DECIMAL(18,4) COMMENT 'Total historical profit',
    predicted_ltv_12m DECIMAL(18,4) COMMENT 'Predicted 12-month LTV',
    predicted_ltv_24m DECIMAL(18,4) COMMENT 'Predicted 24-month LTV',
    predicted_ltv_36m DECIMAL(18,4) COMMENT 'Predicted 36-month LTV',
    churn_probability DECIMAL(5,4) COMMENT 'Probability of churn',
    customer_health_score INT COMMENT 'Health score (0-100)',
    -- Metadata
    model_version STRING COMMENT 'ML model version used',
    calculation_timestamp TIMESTAMP COMMENT 'When calculated',
    PRIMARY KEY (customer_key, calculation_date)
)
USING DELTA
COMMENT 'Customer lifetime value metrics and predictions. Grain: One row per customer per calculation date.'
CLUSTER BY (calculation_date, customer_key);
```

## Column Naming Standards

### General Pattern

**Format**: `<entity>_<attribute>_<qualifier>`

**Examples**:
```sql
customer_id                    # Customer identifier
customer_name                  # Customer name
customer_email_address         # Customer email
customer_registration_date     # Registration date
product_category_code          # Product category
product_unit_price             # Unit price
order_total_amount             # Order total
transaction_timestamp          # Transaction time
```

### Primary Keys

**Format**: `<entity>_key` (surrogate) or `<entity>_id` (natural)

**Examples**:
```sql
customer_key                   # Surrogate key (auto-generated)
product_key                    # Surrogate key
transaction_id                 # Natural business key
order_id                       # Natural business key
```

**Convention**:
- `_key`: Surrogate keys (system-generated, typically BIGINT)
- `_id`: Natural business keys (from source systems, typically STRING)

### Foreign Keys

**Format**: `<referenced_entity>_key` or `<role>_<referenced_entity>_key`

**Standard Foreign Keys**:
```sql
customer_key                   # FK to dim_customer
product_key                    # FK to dim_product
order_date_key                 # FK to dim_date
store_key                      # FK to dim_store
```

**Role-Playing Foreign Keys**:
```sql
-- Multiple references to same dimension
order_date_key                 # Order date
ship_date_key                  # Ship date
required_date_key              # Required date
invoice_date_key               # Invoice date

-- Multiple customer roles
bill_customer_key              # Billing customer
ship_customer_key              # Shipping customer
sold_customer_key              # Sold-to customer
```

### Date and Time Columns

**Timestamps** (`_timestamp` suffix):
```sql
created_timestamp              # When record created
modified_timestamp             # Last modification time
transaction_timestamp          # Transaction occurrence time
processed_timestamp            # ETL processing time
effective_timestamp            # Effective time
```

**Dates** (`_date` suffix):
```sql
order_date                     # Order date
ship_date                      # Shipment date
invoice_date                   # Invoice date
birth_date                     # Date of birth
hire_date                      # Hire date
effective_date                 # SCD2 effective date
expiration_date                # SCD2 expiration date
```

**Date Keys** (`_date_key` suffix):
```sql
order_date_key                 # FK to dim_date (YYYYMMDD format INT)
ship_date_key                  # FK to dim_date
transaction_date_key           # FK to dim_date
```

**Time Components** (`_time` suffix):
```sql
transaction_time               # Time portion only
appointment_time               # Scheduled time
open_time                      # Opening time
close_time                     # Closing time
```

### Amounts and Quantities

**Monetary Amounts** (`_amount` suffix):
```sql
order_total_amount             # Order total
tax_amount                     # Tax amount
discount_amount                # Discount amount
net_amount                     # Net amount
gross_amount                   # Gross amount
fee_amount                     # Fee amount
refund_amount                  # Refund amount
```

**Quantities** (`_quantity` or `_count` suffix):
```sql
order_quantity                 # Quantity ordered
ship_quantity                  # Quantity shipped
inventory_quantity             # Inventory quantity
line_item_count                # Number of line items
customer_count                 # Number of customers
transaction_count              # Number of transactions
```

**Rates and Percentages** (`_rate` or `_percent` suffix):
```sql
interest_rate                  # Interest rate (decimal)
discount_rate                  # Discount rate
tax_rate                       # Tax rate
conversion_rate                # Conversion rate
growth_percent                 # Growth percentage
margin_percent                 # Margin percentage
```

### Codes and Descriptions

**Pattern**: `<attribute>_code` and `<attribute>_description`/`<attribute>_name`

**Examples**:
```sql
status_code                    # Status code value
status_description             # Status description
country_code                   # Country ISO code
country_name                   # Country name
product_category_code          # Category code
product_category_name          # Category name
transaction_type_code          # Transaction type
transaction_type_description   # Transaction type description
```

### Boolean/Flag Columns

**Format**: `is_<condition>` or `has_<attribute>`

**Examples**:
```sql
is_active                      # Active flag
is_deleted                     # Soft delete flag
is_current                     # Current record (SCD2)
is_primary                     # Primary flag
is_taxable                     # Taxable flag
has_discount                   # Discount applied
has_subscription               # Has subscription
has_warranty                   # Has warranty
```

### Calculated/Derived Columns

Optional prefix `calc_` or `derived_` for clarity:

```sql
calc_profit_margin             # Calculated margin
calc_year_over_year_growth     # YoY growth
derived_customer_segment       # Derived segmentation
calc_days_to_ship              # Calculated days
derived_age_years              # Calculated age
```

### SCD2 Standard Columns

**Required for Type 2 Dimensions**:
```sql
effective_date                 # Start date for version
expiration_date                # End date (NULL = current)
is_current                     # Current version flag
row_hash                       # Hash for change detection
version_number                 # Optional version counter
```

### Audit and Metadata Columns

**Standard Audit Columns** (include in every table):
```sql
created_timestamp              # Record creation time
created_by                     # User/process that created
modified_timestamp             # Last modification time
modified_by                    # User/process that modified
source_system                  # Source system identifier
source_record_id               # Source system record ID
batch_id                       # ETL batch/job ID
record_hash                    # Full record hash
etl_insert_date                # Date loaded into table
```

**Delta Lake Metadata** (handled by platform):
```sql
_commit_version                # Delta Lake internal
_commit_timestamp              # Delta Lake internal
```

## View and Materialized View Naming

### View Naming

**Format**: `v_<purpose>_<entity>`

**Examples**:
```sql
v_current_customer             # Current customer records only
v_active_product               # Active products only
v_sales_summary                # Summarized sales view
v_customer_360                 # Comprehensive customer view
v_employee_hierarchy           # Employee org chart
v_product_with_category        # Product with category details
```

**Creation Example**:
```sql
CREATE OR REPLACE VIEW production.gold_customer_360.v_current_customer
AS
SELECT 
    customer_key,
    customer_id,
    customer_name,
    customer_email_address,
    customer_segment,
    effective_date
FROM production.gold_customer_360.dim_customer
WHERE is_current = TRUE;
```

### Materialized View Naming

**Format**: `mv_<purpose>_<entity>`

**Examples**:
```sql
mv_sales_ytd                   # YTD sales metrics
mv_inventory_snapshot          # Current inventory snapshot
mv_customer_metrics            # Customer KPIs
mv_product_performance         # Product performance metrics
mv_monthly_revenue             # Monthly revenue rollup
```

**Creation Example**:
```sql
CREATE OR REFRESH MATERIALIZED VIEW production.gold_sales_retail.mv_sales_ytd
AS
SELECT 
    customer_key,
    product_key,
    YEAR(order_date) AS year,
    SUM(net_amount) AS ytd_revenue,
    COUNT(DISTINCT transaction_id) AS ytd_transaction_count,
    current_timestamp() AS refreshed_timestamp
FROM production.gold_sales_retail.fact_sales_transaction
WHERE order_date >= DATE_TRUNC('year', CURRENT_DATE)
GROUP BY customer_key, product_key, YEAR(order_date);
```

## Databricks-Specific Naming

### Delta Table Properties

**Format**: `<category>.<property_name>`

**Common Properties**:
```sql
-- Delta Lake core properties
'delta.enableChangeDataFeed' = 'true'
'delta.deletedFileRetentionDuration' = 'interval 30 days'
'delta.logRetentionDuration' = 'interval 90 days'
'delta.autoOptimize.optimizeWrite' = 'true'
'delta.autoOptimize.autoCompact' = 'true'

-- Custom business metadata
'business_owner' = 'team@company.com'
'data_classification' = 'confidential'
'refresh_frequency' = 'daily'
'quality_score' = '0.99'

-- SAS migration metadata
'sas_source_program' = 'program_name.sas'
'sas_source_dataset' = 'library.dataset'
'sas_migration_wave' = 'wave_2'
'sas_migration_date' = '2024-01-15'
```

### Unity Catalog Tags

**Format**: `<tag_key>` = `<tag_value>`

**Security Tags**:
```sql
ALTER TABLE production.gold_customer_360.dim_customer
ALTER COLUMN customer_ssn SET TAGS (
    'pii_category' = 'identifier',
    'sensitivity' = 'high',
    'gdpr_relevant' = 'true',
    'encryption_required' = 'true'
);

ALTER TABLE production.gold_finance_accounting.fact_gl_transaction
ALTER COLUMN account_balance SET TAGS (
    'financial_data' = 'true',
    'sox_relevant' = 'true',
    'sensitivity' = 'high'
);
```

**Data Quality Tags**:
```sql
ALTER TABLE production.gold_sales_retail.fact_sales_transaction
ALTER COLUMN net_amount SET TAGS (
    'data_quality_rule' = 'must_be_positive',
    'reconciliation_key' = 'true'
);
```

### Cluster Key Naming

Liquid Clustering uses actual column names (not separate objects), but document the rationale:

```sql
CREATE TABLE fact_sales_transaction (...)
USING DELTA
COMMENT 'Clustered by order_date (temporal queries), customer_key (drill-down), product_key (analysis)'
CLUSTER BY (order_date, customer_key, product_key);
```

## SAS Migration Naming Conventions

### SAS Source Documentation

**Always document SAS source in table properties, not in table names**:

```sql
CREATE TABLE production.gold_sales_retail.fact_sales_transaction (...)
TBLPROPERTIES (
    'sas_source_program' = 'daily_sales_load.sas',
    'sas_source_dataset' = 'sales.daily_trans',
    'sas_source_library' = 'sales',
    'sas_logic_document' = 'confluence.company.com/sas-sales-logic',
    'sas_migration_date' = '2024-01-15',
    'sas_migration_wave' = 'wave_2',
    'sas_validated_by' = 'jane.smith@company.com'
);
```

### Temporary Migration Tables

During parallel validation, use temporary naming:

**Format**: `stg_sas_<original_name>` (in bronze or silver layer)

**Examples**:
```sql
-- Bronze layer (raw SAS data)
bronze.sas_source.stg_sas_daily_sales
bronze.sas_source.stg_sas_customer_master

-- Silver layer (cleansed SAS data for comparison)
silver.sas_validation.stg_sas_sales_clean
```

**Do not use temporary prefixes in gold layer** - use proper dimensional names:
```
❌ gold_sales_retail.temp_sales_transaction
❌ gold_sales_retail.fact_sales_transaction_v1
✅ gold_sales_retail.fact_sales_transaction
```

### Version Suffixes (Temporary Only)

During migration, temporary version suffixes may be used in development:

```sql
-- Development only (remove before production)
development.gold_sales_retail.fact_sales_transaction_v1
development.gold_sales_retail.fact_sales_transaction_v2

-- Production: No version suffixes
production.gold_sales_retail.fact_sales_transaction
```

Use Unity Catalog schemas or separate catalogs for versioning instead:
```sql
development.gold_sales_retail.fact_sales_transaction
staging.gold_sales_retail.fact_sales_transaction
production.gold_sales_retail.fact_sales_transaction
```

## Anti-Patterns to Avoid

### Don't Use

❌ **Abbreviations without business context**:
```
dim_cust                       # Use: dim_customer
fact_ord                       # Use: fact_order
```

❌ **Hungarian notation**:
```
tbl_customer                   # Use: dim_customer
vw_sales                       # Use: v_sales or mv_sales
```

❌ **Mixed case**:
```
DimCustomer                    # Use: dim_customer
FactSalesTransaction           # Use: fact_sales_transaction
```

❌ **Reserved words** (Spark SQL keywords):
```
SELECT, INSERT, UPDATE, DELETE, FROM, WHERE, JOIN, ORDER, GROUP, HAVING,
CREATE, ALTER, DROP, TABLE, VIEW, INDEX, PRIMARY, FOREIGN, KEY, DATE, TIME,
TIMESTAMP, USER, CURRENT, SESSION, SYSTEM, PARTITION, BUCKET, CLUSTER
```

If business term matches reserved word, add qualifier:
```
order → sales_order or customer_order
date → transaction_date or order_date
```

❌ **Special characters** (except underscore):
```
customer-name                  # Use: customer_name
product.category               # Use: product_category
sales$total                    # Use: sales_total
```

❌ **Dates or versions in names**:
```
customer_2024                  # Use: partitioning or date columns
sales_jan                      # Use: date filters
product_v3                     # Use: Unity Catalog versioning
```

❌ **Overly generic names**:
```
data                           # Be specific
table1                         # Use descriptive name
temp                           # Use purpose-specific name
fact_data                      # Which fact?
```

## Databricks Reserved Words

Avoid these Spark SQL reserved words as object names:

```
SELECT, FROM, WHERE, JOIN, ON, AS, AND, OR, NOT, IN, EXISTS, BETWEEN, LIKE,
IS, NULL, TRUE, FALSE, CASE, WHEN, THEN, ELSE, END, DISTINCT, ALL, UNION,
INTERSECT, EXCEPT, GROUP, HAVING, ORDER, LIMIT, OFFSET, WITH, CREATE, ALTER,
DROP, TABLE, VIEW, INDEX, DATABASE, SCHEMA, CATALOG, PARTITION, CLUSTER, BUCKET,
INSERT, UPDATE, DELETE, MERGE, TRUNCATE, DESCRIBE, EXPLAIN, SHOW, USE,
GRANT, REVOKE, SET, RESET, ADD, RENAME, REPLACE
```

Full list: https://spark.apache.org/docs/latest/sql-ref-ansi-compliance.html

## Documentation Requirements

### Table-Level Documentation

Every gold layer table must include:

```sql
CREATE TABLE production.gold_sales_retail.fact_sales_transaction (
    -- Column definitions...
)
USING DELTA
COMMENT 'Comprehensive business description of table purpose, grain, and usage.
Grain: One row per line item on each sales order.
Update Frequency: Near real-time (15-minute refresh).
Data Retention: 7 years per financial policy.
Business Owner: Sales Operations Team.
Primary Use Cases: Daily sales reporting, trend analysis, customer analytics.'
TBLPROPERTIES (
    -- Business metadata
    'business_owner' = 'sales-ops@company.com',
    'data_steward' = 'jane.smith@company.com',
    'data_classification' = 'confidential',
    'compliance_tags' = 'SOX,GDPR',
    
    -- Technical metadata  
    'source_systems' = 'SAP,Salesforce,Shopify',
    'refresh_frequency' = '15 minutes',
    'sla_freshness' = '30 minutes',
    'retention_period' = '7 years',
    
    -- SAS migration metadata
    'sas_source_program' = 'daily_sales_etl.sas',
    'sas_source_dataset' = 'sales.transactions',
    'sas_migration_date' = '2024-01-15',
    'sas_validated_by' = 'john.doe@company.com',
    
    -- Delta Lake configuration
    'delta.enableChangeDataFeed' = 'true',
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.deletedFileRetentionDuration' = 'interval 30 days'
);
```

### Column-Level Documentation

```sql
-- Add comments to all key columns
ALTER TABLE production.gold_sales_retail.fact_sales_transaction
ALTER COLUMN net_amount COMMENT 'Net sales amount = extended_price - discount_amount + tax_amount. Always in USD. Excludes returns (tracked separately in fact_sales_return). Business rule: Must be >= 0 for valid transactions.';

ALTER TABLE production.gold_sales_retail.fact_sales_transaction
ALTER COLUMN customer_key COMMENT 'Foreign key to dim_customer. Links to current customer record if SCD Type 2 is used. Source: CRM system customer ID.';
```

### Unity Catalog Tags

```sql
-- Tag sensitive columns
ALTER TABLE production.gold_sales_retail.fact_sales_transaction
ALTER COLUMN customer_key SET TAGS (
    'pii_category' = 'identifier',
    'sensitivity' = 'high',
    'gdpr_relevant' = 'true'
);

-- Tag reconciliation keys
ALTER TABLE production.gold_sales_retail.fact_sales_transaction
ALTER COLUMN transaction_id SET TAGS (
    'business_key' = 'true',
    'reconciliation_key' = 'true',
    'uniqueness_required' = 'true'
);
```

## Naming Governance

### Approval Process

**Standard Names** (following conventions):
- No approval required
- Use documented patterns
- Document in design review

**Exceptions** (deviating from standards):
- Data Architecture Team approval required
- Document justification
- Add to exception register
- Time-bound with removal plan

**Exception Example**:
```sql
-- Exception documented in table properties
CREATE TABLE production.gold_finance_accounting.fact_legacy_gl (...)
TBLPROPERTIES (
    'naming_exception' = 'true',
    'exception_reason' = 'Maintains compatibility with 20+ existing reports during migration',
    'exception_approved_by' = 'data-architecture-team',
    'exception_approved_date' = '2024-01-15',
    'exception_removal_date' = '2024-06-30',
    'exception_ticket' = 'JIRA-1234'
);
```

### Validation Tools

**Pre-Deployment Checks**:
```python
# Example validation script
def validate_table_name(table_name: str) -> bool:
    """Validate table name follows conventions"""
    
    # Check lowercase
    if table_name != table_name.lower():
        return False
    
    # Check valid prefix
    valid_prefixes = ['dim_', 'fact_', 'agg_', 'bridge_', 'snapshot_', 'ref_', 'metric_']
    if not any(table_name.startswith(p) for p in valid_prefixes):
        return False
    
    # Check no special characters (except underscore)
    if not table_name.replace('_', '').isalnum():
        return False
    
    # Check not a reserved word
    if table_name.upper() in SPARK_RESERVED_WORDS:
        return False
    
    return True
```

**Unity Catalog Query for Compliance**:
```sql
-- Find tables not following naming conventions
SELECT 
    table_catalog,
    table_schema,
    table_name,
    table_type
FROM system.information_schema.tables
WHERE table_schema LIKE 'gold_%'
AND table_name NOT RLIKE '^(dim_|fact_|agg_|bridge_|snapshot_|ref_|metric_)[a-z0-9_]+$'
ORDER BY table_schema, table_name;
```

## Quick Reference Guide

### Schema Naming
```
Pattern: gold_<domain>_<subject_area>
Example: gold_sales_retail
```

### Table Naming by Type
```
Dimension:   dim_<entity>              (e.g., dim_customer)
Fact:        fact_<process>_<grain>    (e.g., fact_sales_transaction)
Aggregate:   agg_<fact>_<level>_<time> (e.g., agg_sales_customer_monthly)
Bridge:      bridge_<relationship>     (e.g., bridge_product_category)
Reference:   ref_<type>                (e.g., ref_country_code)
Metric:      metric_<category>         (e.g., metric_customer_ltv)
```

### Column Naming
```
Primary Key:    <entity>_key  or  <entity>_id
Foreign Key:    <entity>_key  or  <role>_<entity>_key
Date:           <purpose>_date
Timestamp:      <purpose>_timestamp
Date Key:       <purpose>_date_key
Amount:         <description>_amount
Quantity:       <description>_quantity
Boolean:        is_<condition>  or  has_<attribute>
Code:           <attribute>_code
Description:    <attribute>_description  or  <attribute>_name
```

### SCD2 Columns
```
effective_date, expiration_date, is_current, row_hash
```

### Standard Audit Columns
```
created_timestamp, created_by, modified_timestamp, modified_by,
source_system, source_record_id, batch_id, record_hash
```

### Unity Catalog Full Path
```
<catalog>.<schema>.<table>
Example: production.gold_sales_retail.fact_sales_transaction
```

---

**Document Control**

| Attribute | Value |
|-----------|-------|
| Version | 1.0 (Databricks Edition) |
| Last Updated | January 2026 |
| Owner | Data Architecture Team |
| Target Platform | Databricks Unity Catalog |
| Review Date | April 2026 |
| Related Docs | Databricks Gold Layer Master, Modeling Approach |
