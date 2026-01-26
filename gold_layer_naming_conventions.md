# Gold Layer Naming Conventions

## Overview

Consistent and meaningful naming conventions are critical for the gold layer's usability, maintainability, and self-documentation. This document establishes comprehensive naming standards for all gold layer objects to ensure clarity, discoverability, and alignment with business terminology.

## Guiding Principles

1. **Business-Centric**: Names should reflect business concepts and terminology, not technical implementation details
2. **Self-Documenting**: Names should be descriptive enough to convey purpose without requiring extensive documentation
3. **Consistent**: Follow patterns consistently across all objects to build intuitive understanding
4. **Concise**: Balance descriptiveness with brevity to avoid unwieldy names
5. **Future-Proof**: Avoid time-bound or technology-specific references that may become obsolete
6. **Case Sensitivity**: Use lowercase with underscores for database objects (snake_case)

## Schema Naming Standards

### Schema Naming Pattern

**Format**: `gold_<domain>_<subject_area>`

**Components**:
- **gold**: Prefix identifying the medallion layer
- **domain**: Business domain or functional area
- **subject_area**: Specific subject within the domain (optional for simple domains)

### Schema Examples

```
gold_finance_accounting        # Financial accounting data
gold_finance_treasury          # Treasury and cash management
gold_sales_retail              # Retail sales transactions
gold_sales_wholesale           # Wholesale and B2B sales
gold_hr_compensation           # Human resources compensation
gold_hr_talent                 # Talent management and recruiting
gold_operations_manufacturing  # Manufacturing operations
gold_operations_logistics      # Supply chain and logistics
gold_customer_360              # Integrated customer view
gold_product_catalog           # Product master data
gold_common_reference          # Shared reference data (codes, lookups)
```

### Reserved Schema Names

The following schema names are reserved for specific purposes:

- `gold_common`: Cross-domain common dimensions and reference data
- `gold_metrics`: Enterprise-wide KPIs and calculated metrics
- `gold_audit`: Audit and compliance tracking tables
- `gold_archive`: Historical snapshots and archived datasets
- `gold_sandbox`: Experimental or development datasets (temporary)

## Table Naming Standards

### General Table Naming Pattern

**Format**: `<entity_type>_<business_entity>_<qualifier>`

**Components**:
- **entity_type**: Indicates the type of table (dim, fact, agg, bridge, etc.)
- **business_entity**: Core business concept or entity
- **qualifier**: Additional context or specificity (optional)

### Entity Type Prefixes

| Prefix | Type | Purpose | Example |
|--------|------|---------|---------|
| `dim_` | Dimension | Master data, reference data, slowly changing dimensions | `dim_customer` |
| `fact_` | Fact | Transaction or event data with measures | `fact_sales_transaction` |
| `agg_` | Aggregate | Pre-aggregated fact tables | `agg_sales_monthly` |
| `bridge_` | Bridge | Many-to-many relationship resolution | `bridge_account_hierarchy` |
| `snapshot_` | Snapshot | Point-in-time snapshots of dimensions or facts | `snapshot_inventory_daily` |
| `ref_` | Reference | Static reference data and lookup tables | `ref_country_code` |
| `metric_` | Metric | Calculated KPIs and business metrics | `metric_customer_lifetime_value` |

### Dimension Table Naming

**Format**: `dim_<entity>_<type_qualifier>`

**Examples**:
```
dim_customer                   # Customer master dimension
dim_customer_scd2              # Customer with Type 2 history
dim_product                    # Product master dimension
dim_date                       # Date dimension (calendar)
dim_date_fiscal                # Fiscal calendar dimension
dim_employee                   # Employee dimension
dim_account                    # Chart of accounts dimension
dim_geography                  # Geographic hierarchy
dim_channel                    # Sales channel dimension
dim_promotion                  # Marketing promotion dimension
```

**SCD Type Suffix**:
- No suffix: Current state only (Type 0 or Type 1)
- `_scd2`: Slowly changing dimension Type 2 (full history)
- `_scd3`: Slowly changing dimension Type 3 (limited history)

### Fact Table Naming

**Format**: `fact_<business_process>_<granularity>`

**Examples**:
```
fact_sales_transaction         # Individual sales transactions
fact_order_line                # Order line items
fact_payment                   # Payment events
fact_inventory_movement        # Inventory transactions
fact_shipment                  # Shipment events
fact_gl_transaction            # General ledger transactions
fact_web_clickstream           # Website clickstream events
fact_service_ticket            # Customer service tickets
```

**Periodic Snapshot Facts**:
```
fact_account_balance_daily     # Daily account balances
fact_inventory_level_monthly   # Monthly inventory levels
fact_pipeline_weekly           # Weekly sales pipeline snapshot
```

**Accumulating Snapshot Facts**:
```
fact_order_fulfillment         # Order lifecycle from creation to delivery
fact_claim_processing          # Insurance claim lifecycle
fact_loan_origination          # Loan application lifecycle
```

### Aggregate Table Naming

**Format**: `agg_<fact_name>_<aggregation_level>_<time_grain>`

**Examples**:
```
agg_sales_customer_monthly     # Sales aggregated by customer and month
agg_sales_product_daily        # Sales aggregated by product and day
agg_revenue_region_quarterly   # Revenue by region and quarter
agg_inventory_warehouse_weekly # Inventory levels by warehouse and week
agg_orders_channel_yearly      # Order counts by channel and year
```

### Bridge Table Naming

**Format**: `bridge_<relationship_description>`

**Examples**:
```
bridge_account_hierarchy       # Account parent-child relationships
bridge_employee_manager        # Employee organizational hierarchy
bridge_product_category        # Product multi-level categorization
bridge_customer_household      # Customer household groupings
```

### Reference Table Naming

**Format**: `ref_<reference_type>`

**Examples**:
```
ref_country_code               # Country code lookup
ref_currency_code              # Currency code lookup
ref_status_code                # Status code descriptions
ref_transaction_type           # Transaction type classifications
ref_industry_classification    # Industry classification codes (SIC, NAICS)
```

### Metric Table Naming

**Format**: `metric_<metric_category>_<specificity>`

**Examples**:
```
metric_customer_lifetime_value # CLV calculations
metric_customer_churn          # Churn metrics and predictions
metric_product_margin          # Product profitability metrics
metric_sales_target            # Sales targets and goals
metric_inventory_health        # Inventory health indicators
```

## Column Naming Standards

### General Column Naming Pattern

**Format**: `<entity>_<attribute>_<qualifier>`

**Examples**:
```
customer_id                    # Customer identifier
customer_name                  # Customer name
customer_email_address         # Customer email
customer_registration_date     # Date customer registered
product_category_code          # Product category code
product_unit_price             # Product unit price
order_total_amount             # Order total amount
transaction_timestamp          # Transaction timestamp
```

### Standard Column Names

#### Primary Keys

**Format**: `<table_entity>_key` or `<table_entity>_id`

**Examples**:
```
customer_key                   # Surrogate key for dim_customer
product_key                    # Surrogate key for dim_product
order_id                       # Natural business key for orders
transaction_id                 # Natural business key for transactions
```

**Best Practice**: Use `_key` suffix for surrogate keys (auto-generated integers), `_id` suffix for natural business keys

#### Foreign Keys

**Format**: `<referenced_table>_<column_name>`

**Examples**:
```
customer_key                   # Foreign key to dim_customer.customer_key
product_key                    # Foreign key to dim_product.product_key
ship_date_key                  # Foreign key to dim_date.date_key
bill_customer_key              # Foreign key to customer dimension (billing)
ship_customer_key              # Foreign key to customer dimension (shipping)
```

**Role-Playing Dimensions**: When a dimension is referenced multiple times, prefix with the role

#### Date and Time Columns

**Timestamp Columns**: Use `_timestamp` suffix
```
created_timestamp
modified_timestamp
transaction_timestamp
processed_timestamp
```

**Date Columns**: Use `_date` suffix
```
order_date
ship_date
invoice_date
effective_date
expiration_date
```

**Time Columns**: Use `_time` suffix
```
transaction_time
appointment_time
```

#### Amount and Quantity Columns

**Monetary Amounts**: Use `_amount` suffix
```
order_total_amount
tax_amount
discount_amount
net_amount
```

**Quantities**: Use `_quantity` or `_count` suffix
```
order_quantity
inventory_quantity
line_item_count
customer_count
```

**Rates and Percentages**: Use `_rate` or `_percent` suffix
```
interest_rate
discount_rate
tax_rate
growth_percent
```

#### Code and Description Columns

**Format**: 
- Code: `<attribute>_code`
- Description: `<attribute>_description` or `<attribute>_desc`

**Examples**:
```
status_code                    # Status code value
status_description             # Status code description
country_code                   # Country ISO code
country_name                   # Country full name
product_category_code          # Category code
product_category_description   # Category description
```

#### Boolean/Flag Columns

**Format**: `is_<condition>` or `has_<attribute>`

**Examples**:
```
is_active                      # Active flag
is_deleted                     # Soft delete flag
is_current                     # Current record indicator (SCD2)
has_discount                   # Discount applied flag
has_subscription               # Subscription flag
is_primary                     # Primary record flag
```

#### Calculated and Derived Columns

Prefix with `calc_` or `derived_` when it's important to distinguish from source data:

```
calc_profit_margin             # Calculated profit margin
calc_year_over_year_growth     # Calculated YoY growth
derived_customer_segment       # Derived customer segmentation
```

### SCD2 Standard Columns

For Type 2 slowly changing dimensions, include these standard columns:

```
effective_date                 # Start date for this version
expiration_date                # End date for this version (NULL for current)
is_current                     # Flag indicating current version (TRUE/FALSE)
row_hash                       # Hash of business key + attributes for change detection
```

### Audit and Metadata Columns

Every gold layer table should include these standard audit columns:

```
created_timestamp              # When record was created
created_by                     # User/process that created record
modified_timestamp             # When record was last modified
modified_by                    # User/process that last modified record
source_system                  # Source system identifier
source_record_id               # Source system record identifier
batch_id                       # Batch/job identifier for data lineage
record_hash                    # Full record hash for change detection
```

## View Naming Standards

### View Naming Pattern

**Format**: `v_<purpose>_<entity>`

**Examples**:
```
v_current_customer             # View of current customer records (SCD2)
v_active_product               # View of active products only
v_sales_summary                # Summarized sales view
v_customer_360                 # Comprehensive customer view
v_employee_hierarchy           # Employee organizational hierarchy
```

### Materialized View Naming

**Format**: `mv_<purpose>_<entity>`

**Examples**:
```
mv_sales_ytd                   # Materialized view of YTD sales
mv_inventory_snapshot          # Materialized inventory snapshot
mv_customer_metrics            # Materialized customer metrics
```

## Stored Procedure and Function Naming

### Stored Procedure Naming

**Format**: `sp_<action>_<entity>`

**Examples**:
```
sp_load_dim_customer           # Load customer dimension
sp_merge_fact_sales            # Merge sales fact table
sp_calculate_customer_metrics  # Calculate customer metrics
sp_reconcile_inventory         # Reconcile inventory data
```

### Function Naming

**Format**: `fn_<purpose>` or `udf_<purpose>`

**Examples**:
```
fn_calculate_age               # Calculate age from birth date
fn_fiscal_period               # Convert date to fiscal period
fn_distance_between_points     # Calculate geographic distance
udf_parse_json_field           # User-defined function to parse JSON
```

## Index and Constraint Naming

### Primary Key Constraint

**Format**: `pk_<table_name>`

**Examples**:
```
pk_dim_customer                # Primary key on dim_customer
pk_fact_sales_transaction      # Primary key on fact_sales_transaction
```

### Foreign Key Constraint

**Format**: `fk_<table_name>_<referenced_table>`

**Examples**:
```
fk_fact_sales_customer         # Foreign key to customer dimension
fk_fact_sales_product          # Foreign key to product dimension
fk_fact_sales_date             # Foreign key to date dimension
```

### Index Naming

**Format**: `ix_<table_name>_<column(s)>`

**Examples**:
```
ix_dim_customer_email          # Index on email address
ix_fact_sales_date_product     # Composite index on date and product
ix_fact_sales_timestamp        # Index on transaction timestamp
```

### Unique Constraint

**Format**: `uq_<table_name>_<column(s)>`

**Examples**:
```
uq_dim_customer_email          # Unique constraint on email
uq_dim_product_sku             # Unique constraint on product SKU
```

## Naming Conventions for SAS Migration Objects

### SAS Source Reference

When migrating from SAS, maintain traceability to source:

**Documentation Field**: `sas_source_program`
```
-- Document in table comments
sas_source_program: sales_monthly_rollup.sas
sas_dataset: work.monthly_sales
```

**Temporary Migration Tables**: `stg_sas_<original_name>`
```
stg_sas_monthly_sales          # Staging table from SAS monthly_sales dataset
```

### Version Suffixes (Temporary Use Only)

During migration, version suffixes may be used temporarily:

```
dim_customer_v1                # Version 1 (legacy SAS logic)
dim_customer_v2                # Version 2 (redesigned)
```

**Important**: Remove version suffixes once migration is complete and validated

## Anti-Patterns to Avoid

### Don't Use

❌ **Abbreviations without business alignment**:
```
dim_cust                       # Use: dim_customer
fact_ord                       # Use: fact_order
```

❌ **Hungarian notation**:
```
tbl_customer                   # Use: dim_customer
str_name                       # Use: name
int_count                      # Use: count
```

❌ **Camel case or pascal case** for database objects:
```
DimCustomer                    # Use: dim_customer
FactSalesTransaction           # Use: fact_sales_transaction
```

❌ **Reserved words** as table or column names:
```
user                           # Use: user_profile or username
date                           # Use: transaction_date or event_date
order                          # Use: sales_order or order_header
```

❌ **Overly generic names**:
```
data                           # Be specific
table1                         # Use descriptive name
temp                           # Use purpose-specific name
```

❌ **Special characters** (except underscore):
```
customer-name                  # Use: customer_name
product.category               # Use: product_category
sales$total                    # Use: sales_total
```

❌ **Dates or versions in table names**:
```
customer_2024                  # Use partitioning or effective dates
sales_jan                      # Use date column and filtering
product_v3                     # Use proper versioning in metadata
```

## Reserved Words and Keywords

Avoid using database reserved words as object names. Common reserved words include:

```
SELECT, INSERT, UPDATE, DELETE, FROM, WHERE, JOIN, GROUP, ORDER, HAVING,
CREATE, ALTER, DROP, TABLE, VIEW, INDEX, CONSTRAINT, PRIMARY, FOREIGN,
KEY, REFERENCES, UNIQUE, NOT, NULL, DEFAULT, CHECK, DISTINCT, UNION,
INTERSECT, EXCEPT, CASE, WHEN, THEN, ELSE, END, AS, IN, EXISTS, LIKE,
BETWEEN, AND, OR, NOT, IS, TRUE, FALSE, DATE, TIME, TIMESTAMP, YEAR,
MONTH, DAY, HOUR, MINUTE, SECOND, USER, CURRENT, SESSION, SYSTEM
```

If you must use a reserved word (due to business terminology), enclose it in quotes or delimiters as appropriate for your database platform.

## Documentation Requirements

### Table-Level Documentation

Every gold layer table must include:

1. **Table Comment**: Business purpose and description
2. **Owner Information**: Data owner and steward
3. **Source Mapping**: Source systems and upstream tables
4. **SAS Lineage**: Original SAS program/dataset (if applicable)
5. **Refresh Frequency**: How often data is updated
6. **Data Retention**: How long data is retained

**Example**:
```sql
COMMENT ON TABLE gold_sales_retail.fact_sales_transaction IS 
'Daily retail sales transactions including all point-of-sale data.
Owner: Sales Operations (john.doe@company.com)
Source: silver_pos.transactions, silver_crm.customer
SAS Source: daily_sales_load.sas
Refresh: Daily at 6 AM EST
Retention: 7 years';
```

### Column-Level Documentation

Critical columns should include:

1. **Column Comment**: Business definition
2. **Data Type Justification**: Why this data type was chosen
3. **Valid Values**: For code columns, reference to valid values
4. **Calculation Logic**: For derived columns

**Example**:
```sql
COMMENT ON COLUMN gold_sales_retail.fact_sales_transaction.customer_segment IS
'Customer segmentation based on RFM analysis (Recency, Frequency, Monetary).
Valid values: Premium, Standard, Occasional, Lapsed
Calculated by: sp_calculate_customer_segment
Business Rule: Defined in Customer Segmentation Framework v2.1';
```

## Naming Governance

### Approval Process

New naming patterns or exceptions require approval from:
1. Data Architecture team (mandatory)
2. Data Governance committee (for enterprise-wide patterns)
3. Domain data steward (for domain-specific patterns)

### Exception Handling

Exceptions to naming standards must be:
1. **Documented**: Reason for exception clearly stated
2. **Approved**: By data architecture team
3. **Time-Boxed**: For temporary exceptions, include removal date
4. **Tracked**: In exception registry

**Example Exception**:
```
Table: dim_customer_legacy
Reason: Maintains compatibility with existing BI reports during migration
Approved By: Data Architecture Team (2024-01-15)
Removal Date: 2024-06-30 (post migration completion)
Tracking ID: EXCEPTION-2024-001
```

## Tools and Automation

### Naming Validation

Use automated tools to validate naming conventions:

1. **Pre-commit hooks**: Check naming before code commit
2. **CI/CD pipeline checks**: Validate during deployment
3. **Data catalog integration**: Flag non-compliant objects
4. **Periodic audits**: Monthly review of naming compliance

### Naming Templates

Use templates for common object types:

```sql
-- Dimension table template
CREATE TABLE gold_<domain>.dim_<entity> (
    <entity>_key BIGINT PRIMARY KEY,
    <entity>_id VARCHAR(50) NOT NULL,
    -- Business attributes
    effective_date DATE NOT NULL,
    expiration_date DATE,
    is_current BOOLEAN DEFAULT TRUE,
    created_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100),
    modified_timestamp TIMESTAMP,
    modified_by VARCHAR(100),
    source_system VARCHAR(50),
    batch_id VARCHAR(100),
    row_hash VARCHAR(64)
);
```

## Quick Reference Guide

### Schema Naming
- Pattern: `gold_<domain>_<subject_area>`
- Example: `gold_sales_retail`

### Table Naming
- Dimension: `dim_<entity>` (e.g., `dim_customer`)
- Fact: `fact_<process>_<grain>` (e.g., `fact_sales_transaction`)
- Aggregate: `agg_<fact>_<level>_<grain>` (e.g., `agg_sales_customer_monthly`)

### Column Naming
- Primary key: `<entity>_key` or `<entity>_id`
- Foreign key: `<referenced_entity>_key`
- Date: `<purpose>_date`
- Timestamp: `<purpose>_timestamp`
- Amount: `<description>_amount`
- Boolean: `is_<condition>` or `has_<attribute>`

### Standard Audit Columns
```
created_timestamp, created_by, modified_timestamp, modified_by,
source_system, source_record_id, batch_id, record_hash
```

### SCD2 Columns
```
effective_date, expiration_date, is_current, row_hash
```

---

**Document Control**

| Attribute | Value |
|-----------|-------|
| Version | 1.0 |
| Last Updated | January 2026 |
| Owner | Data Architecture Team |
| Review Date | April 2026 |
