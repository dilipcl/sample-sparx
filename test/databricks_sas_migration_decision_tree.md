# SAS to Databricks Gold Layer: Migration Decision Tree

## Overview

This decision tree provides a structured framework for migrating SAS programs and datasets to Databricks gold layer. It guides teams through critical decisions about migration patterns, technology choices, and implementation approaches optimized for the Databricks Lakehouse Platform.

**Platform Focus**: All decisions are tailored for Databricks Delta Lake, Unity Catalog, and associated technologies.

**Objective**: Enable teams to make informed, consistent decisions that result in successful SAS decommissioning while building a modern, scalable gold layer.

## How to Use This Decision Tree

1. **Start with Asset Analysis**: Understand what you're migrating (SAS program characteristics)
2. **Follow Decision Points Sequentially**: Each decision builds on previous ones
3. **Document All Decisions**: Use the decision record template at the end
4. **Validate with Architecture Team**: Review complex migrations before starting
5. **Iterate as Needed**: Reassess if new information emerges

## Decision Framework Overview

```
SAS Asset → Complexity Assessment → Migration Pattern Selection
              ↓                        ↓
        Load Strategy ← Technology Choices → Testing Strategy
              ↓                              ↓
        Implementation ← Validation ← Cutover Strategy
```

## Phase 1: Asset Analysis and Classification

### Decision Point 1.1: What Are We Migrating?

**Question**: What type of SAS asset is in scope?

```
SAS PROGRAM TYPES:

├─ DATA Step
│  └─ Assessment: Transformation logic - translate to PySpark/Spark SQL
│  └─ Complexity: Moderate (depends on complexity of logic)
│  └─ Pattern: Incremental transformation development
│
├─ PROC SQL
│  └─ Assessment: SQL-based - direct translation to Spark SQL
│  └─ Complexity: Low to Moderate
│  └─ Pattern: SQL translation with syntax adjustments
│
├─ PROC MEANS / PROC SUMMARY
│  └─ Assessment: Aggregation logic - map to GROUP BY queries
│  └─ Complexity: Low
│  └─ Pattern: Direct aggregation translation
│
├─ PROC TRANSPOSE
│  └─ Assessment: Pivoting logic - use PIVOT/UNPIVOT
│  └─ Complexity: Moderate
│  └─ Pattern: Reshape transformation
│
├─ PROC FREQ
│  └─ Assessment: Frequency/counting - GROUP BY with COUNT
│  └─ Complexity: Low
│  └─ Pattern: Simple aggregation
│
├─ Macro Programs
│  └─ Assessment: Parameterized logic - use Databricks widgets/parameters
│  └─ Complexity: Moderate to High
│  └─ Pattern: Decompose and convert to parameterized notebooks
│
└─ Mixed/Complex Programs
   └─ Assessment: Multiple PROC steps and data manipulation
   └─ Complexity: High
   └─ Pattern: Detailed analysis and decomposition required

SAS DATASET TYPES:

├─ Permanent Datasets (Libraries)
│  └─ Assessment: Gold layer candidates
│  └─ Action: Map to dimensional model (dim/fact tables)
│
├─ Work Datasets (Temporary)
│  └─ Assessment: Intermediate processing artifacts
│  └─ Action: May not need gold layer equivalent (handle in pipeline)
│
└─ SAS Views
   └─ Assessment: Derived/calculated data
   └─ Action: Evaluate if needed as materialized view or regular view
```

**Output**: Asset type classification and initial migration approach

### Decision Point 1.2: Complexity Scoring

**Question**: How complex is this SAS asset?

Use this scoring matrix to assess complexity:

| Factor | Low (1 pt) | Moderate (2 pts) | High (3 pts) |
|--------|------------|------------------|--------------|
| **Lines of Code** | < 100 lines | 100-500 lines | > 500 lines |
| **Input Sources** | 1-2 sources | 3-5 sources | 6+ sources |
| **Transformations** | Simple filters/joins | Aggregations, calculations | Complex business rules, nested logic |
| **Macros Used** | No macros | Simple macros (< 3) | Nested/dynamic macros (3+) |
| **Dependencies** | Standalone | Few dependencies (2-4) | Highly interdependent (5+) |
| **Output Datasets** | 1 dataset | 2-3 datasets | 4+ datasets |
| **Business Logic** | Straightforward | Documented, moderate complexity | Undocumented, domain-specific |
| **Data Volume** | < 1M rows | 1M-100M rows | > 100M rows |
| **SAS Features** | Basic SQL, DATA steps | PROC usage, formats | Advanced: arrays, DO loops, RETAIN |

**Complexity Scoring**:
```
Total Score = Sum of all factors

IF Score <= 9:
    Classification = LOW COMPLEXITY
    → Estimated Duration: 2-5 days
    → Resources: 1 developer
    → Migration Pattern: Direct Translation
    
ELSE IF Score 10-18:
    Classification = MODERATE COMPLEXITY  
    → Estimated Duration: 1-2 weeks
    → Resources: 1-2 developers + code reviewer
    → Migration Pattern: Standard Migration with Testing
    
ELSE IF Score >= 19:
    Classification = HIGH COMPLEXITY
    → Estimated Duration: 2-4 weeks
    → Resources: 2+ developers + architect + SAS SME
    → Migration Pattern: Phased Migration with Extensive Validation
```

**Example Scoring**:
```
SAS Program: monthly_sales_summary.sas
- Lines of Code: 350 lines → 2 pts
- Input Sources: 4 tables → 2 pts
- Transformations: Aggregations + calculations → 2 pts
- Macros: 2 simple macros → 2 pts
- Dependencies: Reads from 3 upstream programs → 2 pts
- Outputs: 2 datasets → 2 pts
- Business Logic: Documented, moderate → 2 pts
- Data Volume: 25M rows → 2 pts
- SAS Features: PROC SQL + basic DATA step → 1 pt

Total Score = 17 → MODERATE COMPLEXITY
```

**Output**: Complexity score, resource estimate, timeline estimate

### Decision Point 1.3: Business Criticality

**Question**: What is the business criticality and risk profile?

```
CRITICALITY ASSESSMENT:

CRITICAL (Highest Priority):
├─ Supports regulatory reporting (SOX, SEC, FDA)
├─ C-level dashboards and executive reporting
├─ Financial close processes
├─ Customer-facing data or systems
├─ Revenue recognition or billing
├─ No acceptable downtime (< 1 hour)
└─ Legal or compliance requirements

→ Requirements:
  ✓ Extended parallel run (4+ weeks)
  ✓ 100% reconciliation (all 3 levels)
  ✓ Executive stakeholder sign-off
  ✓ Detailed rollback plan with tested procedures
  ✓ 24x7 hypercare support post-cutover
  ✓ Independent validation by audit/compliance team

HIGH (Important):
├─ Departmental KPIs and scorecards
├─ Operational dashboards (daily use)
├─ Core business processes
├─ Weekly/monthly reporting cycles
└─ Short downtime acceptable (< 4 hours)

→ Requirements:
  ✓ Standard parallel run (2 weeks)
  ✓ Full reconciliation (levels 1-2, sample level 3)
  ✓ Business owner sign-off
  ✓ Rollback plan documented
  ✓ Business hours hypercare support

MEDIUM (Standard):
├─ Ad-hoc analysis datasets
├─ Historical research data
├─ Supporting analytics
└─ Longer downtime acceptable (< 1 day)

→ Requirements:
  ✓ Short parallel run (1 week)
  ✓ Aggregate reconciliation (levels 1-2)
  ✓ Technical team validation
  ✓ Standard support

LOW (Opportunistic):
├─ Exploratory analysis
├─ Rarely used reports
├─ Deprecated but not yet retired
└─ Extended downtime acceptable (days)

→ Requirements:
  ✓ Basic validation
  ✓ Smoke testing
  ✓ As-needed support
```

**Output**: Criticality level, testing requirements, support model

## Phase 2: Migration Pattern Selection

### Decision Point 2.1: Translation Approach

**Question**: Should we do direct translation or re-architecture?

```
DECISION FACTORS:

Factor 1: SAS Code Quality
├─ Well-documented, tested, optimized
│  → Lean toward DIRECT TRANSLATION
├─ Functional but has performance issues
│  → Consider HYBRID approach
└─ Poorly documented, inefficient, technical debt
   → Lean toward RE-ARCHITECTURE

Factor 2: Business Logic Changes
├─ Logic is still current and valid
│  → DIRECT TRANSLATION acceptable
├─ Minor updates needed
│  → HYBRID with targeted improvements
└─ Significant business logic changes required
   → RE-ARCHITECTURE opportunity

Factor 3: Data Model Alignment
├─ SAS dataset structure aligns with dimensional model
│  → DIRECT TRANSLATION viable
├─ Some denormalization or restructuring needed
│  → HYBRID approach
└─ Significant structural mismatch
   → RE-ARCHITECTURE required

Factor 4: Timeline Pressure
├─ Aggressive deadline, limited resources
│  → DIRECT TRANSLATION (lower risk)
├─ Standard timeline
│  → HYBRID (balanced approach)
└─ Flexible timeline, focus on optimization
   → RE-ARCHITECTURE (long-term investment)

Factor 5: Databricks Optimization Opportunities
├─ SAS logic maps well to Spark SQL patterns
│  → DIRECT TRANSLATION works
├─ Could benefit from Delta Lake features (MERGE, CDF)
│  → HYBRID to leverage platform
└─ Significant performance gains from re-design
   → RE-ARCHITECTURE recommended
```

**Migration Patterns**:

**A. DIRECT TRANSLATION** (Lift and Shift):
```
When to use:
✓ Low to moderate complexity
✓ Well-documented SAS code
✓ Tight timeline
✓ Minimal business logic changes
✓ SAS logic works well

Approach:
1. Translate SAS PROC SQL → Spark SQL (syntax changes only)
2. Translate DATA steps → PySpark DataFrame operations
3. Maintain similar processing logic and flow
4. Preserve data structures where possible
5. Focus on functional equivalence

Databricks Implementation:
- Use Spark SQL for PROC SQL equivalents
- Use PySpark for DATA step equivalents
- Store as Delta tables with basic configuration
- Implement standard Liquid Clustering
- Use Delta MERGE for updates

Example:
SAS: PROC SQL; CREATE TABLE output AS SELECT ...
→ Databricks: spark.sql("CREATE TABLE output USING DELTA AS SELECT ...")
```

**B. RE-ARCHITECTURE** (Transform and Optimize):
```
When to use:
✓ High complexity with technical debt
✓ Poor SAS code quality or performance
✓ Significant business logic changes needed
✓ Flexible timeline
✓ Opportunity for major improvements

Approach:
1. Analyze business requirements (not just SAS code)
2. Design optimal dimensional model for Databricks
3. Implement using Delta Lake best practices
4. Leverage Liquid Clustering for performance
5. Build in data quality and governance from start

Databricks Implementation:
- Design star schema or data vault model
- Use SCD Type 2 with Delta MERGE for dimensions
- Implement Change Data Feed for incremental processing
- Optimize with Liquid Clustering and Photon
- Add Unity Catalog governance controls
- Create materialized views for aggregations

Example:
SAS: Complex macro with multiple DATA steps
→ Databricks: Dimensional model with fact/dim tables, 
              Delta Live Tables pipeline, ML-powered data quality
```

**C. HYBRID APPROACH** (Pragmatic Balance):
```
When to use:
✓ Moderate complexity
✓ Good SAS logic with some inefficiencies  
✓ Standard timeline
✓ Balance of speed and optimization

Approach:
1. Translate core logic directly
2. Re-architect specific pain points
3. Add Databricks optimizations selectively
4. Maintain overall SAS structure but enhance

Databricks Implementation:
- Translate SAS logic to Spark SQL/PySpark
- Refactor inefficient sections
- Add Delta Lake features (MERGE, time travel)
- Implement Liquid Clustering
- Enhance with data quality checks

Example:
SAS: Efficient aggregation + inefficient joins
→ Databricks: Keep aggregation logic, redesign join strategy,
              add broadcast hints, use Liquid Clustering
```

**Decision Matrix**:

| SAS Code Quality | Business Changes | Timeline | → Recommendation |
|-----------------|------------------|----------|------------------|
| Good | None | Tight | Direct Translation |
| Good | Minor | Standard | Hybrid |
| Good | Major | Flexible | Re-Architecture |
| Poor | Any | Tight | Direct (plan re-arch later) |
| Poor | Minor | Standard | Hybrid |
| Poor | Major | Flexible | Re-Architecture |

**Output**: Selected migration pattern with justification

### Decision Point 2.2: Databricks Technology Choices

**Question**: Which Databricks features should we use?

```
DECISION: DELTA LAKE MERGE vs INSERT
├─ Data Characteristics:
│  ├─ Updates/Deletes Required?
│  │  ├─ YES → Use MERGE
│  │  └─ NO → Use INSERT (append-only)
│  │
│  ├─ Deduplication Needed?
│  │  ├─ YES → Use MERGE
│  │  └─ NO → Use INSERT
│  │
│  └─ SCD Type 2 Required?
│     ├─ YES → Use MERGE
│     └─ NO → Use INSERT
│
└─ Recommendation:
   ├─ MERGE for dimensions (especially with SCD)
   ├─ MERGE for facts with updates (accumulating snapshots)
   └─ INSERT for append-only facts (transaction facts)

DECISION: LIQUID CLUSTERING vs TRADITIONAL PARTITIONING
├─ Databricks Runtime Version:
│  ├─ DBR 13.3+ → Use LIQUID CLUSTERING (recommended)
│  └─ DBR < 13.3 → Use traditional partitioning or ZORDER
│
├─ Query Patterns:
│  ├─ Multiple filter dimensions → LIQUID CLUSTERING
│  ├─ Single dominant filter dimension → Either works
│  └─ Date-only filtering → Either works
│
├─ Data Lifecycle:
│  ├─ Need DROP PARTITION for GDPR → Traditional partitioning
│  └─ Standard retention → LIQUID CLUSTERING
│
└─ Recommendation:
   └─ DEFAULT to LIQUID CLUSTERING for new tables

DECISION: BATCH vs STREAMING
├─ Latency Requirements:
│  ├─ Real-time (< 5 minutes) → STREAMING (Delta Live Tables)
│  ├─ Near real-time (< 30 minutes) → STREAMING or frequent batch
│  └─ Batch (hourly, daily) → BATCH (Databricks Workflows)
│
├─ Data Arrival Pattern:
│  ├─ Continuous stream → STREAMING
│  ├─ Micro-batches → STREAMING
│  └─ Scheduled batches → BATCH
│
└─ Complexity:
   ├─ Simple transformation → STREAMING (Delta Live Tables)
   └─ Complex business logic → BATCH (easier to develop/test)

DECISION: MATERIALIZED VIEW vs AGGREGATE TABLE
├─ Update Pattern:
│  ├─ Need automatic updates → MATERIALIZED VIEW
│  ├─ Complex aggregation logic → AGGREGATE TABLE (more control)
│  └─ Custom incremental logic → AGGREGATE TABLE
│
├─ Query Performance:
│  ├─ Simple aggregations → MATERIALIZED VIEW
│  └─ Complex multi-fact aggregations → AGGREGATE TABLE
│
└─ Recommendation:
   ├─ Start with MATERIALIZED VIEW (easier)
   └─ Move to AGGREGATE TABLE if need custom logic

DECISION: PYTHON vs SQL
├─ Team Skills:
│  ├─ Strong SQL, limited Python → Spark SQL
│  ├─ Strong Python → PySpark
│  └─ Mixed → Spark SQL for queries, Python for complex logic
│
├─ Complexity:
│  ├─ Simple transformations → Spark SQL
│  ├─ Complex iterations → PySpark
│  └─ ML integration → PySpark
│
└─ SAS Source:
   ├─ PROC SQL → Spark SQL (direct translation)
   └─ DATA Step → PySpark (more similar paradigm)
```

**Technology Selection Template**:
```
For table: fact_sales_transaction

Load Strategy: MERGE (need to handle late-arriving data)
Organization: LIQUID CLUSTERING on (order_date, customer_key)
Processing: BATCH (daily refresh acceptable)
Language: Spark SQL (team strength, simple logic)
Aggregates: MATERIALIZED VIEW for daily/monthly rollups
CDC: Enable Change Data Feed (for downstream consumers)
```

**Output**: Technology stack for implementation

## Phase 3: Load Strategy Decisions

### Decision Point 3.1: Incremental vs Full Load

**Question**: What load pattern should we use?

```
FACTORS TO CONSIDER:

Data Volume Analysis:
├─ < 1M rows
│  └─ FULL LOAD typically acceptable (fast enough)
│
├─ 1M - 100M rows
│  ├─ Change rate < 10% → INCREMENTAL preferred
│  └─ Change rate > 50% → FULL LOAD may be simpler
│
└─ > 100M rows
   └─ INCREMENTAL required (full load too slow)

Change Tracking Capability:
├─ Has timestamp/version column?
│  ├─ YES → INCREMENTAL easy to implement
│  └─ NO → Consider full load or add tracking
│
├─ Has CDC available?
│  ├─ YES → INCREMENTAL with CDC
│  └─ NO → Evaluate alternatives
│
└─ SAS used incremental logic?
   ├─ YES → Preserve pattern
   └─ NO → Evaluate best approach for Databricks

SAS Processing Pattern:
├─ SAS did full refresh → Start with full, optimize to incremental
├─ SAS did incremental → Preserve incremental pattern
└─ SAS was inconsistent → Redesign for clarity

Business Requirements:
├─ Need point-in-time snapshots?
│  └─ May need periodic full loads for snapshot tables
│
├─ Data corrections common?
│  └─ Consider full load with validations
│
└─ Regulatory requirements?
   └─ May need specific load patterns for audit
```

**Load Pattern Decision Matrix**:

| Volume | Change Rate | Tracking | → Recommendation |
|--------|-------------|----------|------------------|
| Small | Any | Any | Full Load |
| Medium | < 10% | Yes | Incremental |
| Medium | < 10% | No | Full Load (simpler) |
| Medium | > 50% | Yes | Full Load or Incremental |
| Large | < 25% | Yes | Incremental (required) |
| Large | Any | No | Add tracking, then Incremental |

**Implementation Patterns**:

**A. FULL LOAD**:
```sql
-- Simple full refresh pattern
CREATE OR REPLACE TABLE production.gold_sales_retail.dim_product
USING DELTA
AS
SELECT 
    product_key,
    product_id,
    product_name,
    product_category,
    current_timestamp() AS loaded_timestamp
FROM silver.product_master;

-- Advantages:
✓ Simple to implement and test
✓ No complex change tracking logic
✓ Always consistent (no missed updates)
✓ Easy reconciliation

-- Disadvantages:
✗ Can be slow for large tables
✗ Resource intensive
✗ May not meet latency requirements
```

**B. INCREMENTAL LOAD (Timestamp-based)**:
```sql
-- Get last loaded timestamp
DECLARE last_load_time TIMESTAMP;
SET last_load_time = (
    SELECT MAX(modified_timestamp) 
    FROM production.gold_sales_retail.fact_sales_transaction
);

-- Load only new/changed records
MERGE INTO production.gold_sales_retail.fact_sales_transaction AS target
USING (
    SELECT * FROM silver.sales_transactions
    WHERE modified_timestamp > last_load_time
) AS source
ON target.transaction_id = source.transaction_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- Advantages:
✓ Fast for large tables with few changes
✓ Lower resource usage
✓ Can meet tight SLAs

-- Disadvantages:
✗ More complex logic
✗ Requires reliable timestamp column
✗ Risk of missing deletes (need separate handling)
```

**C. INCREMENTAL LOAD (Change Data Feed)**:
```python
# Using Delta Change Data Feed for incremental processing
from delta.tables import DeltaTable

# Read changes from source
source_changes = (
    spark.readStream
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", last_processed_version)
    .table("silver.sales_transactions")
)

# Process changes
def process_changes(batch_df, batch_id):
    # Filter for inserts and updates
    new_data = batch_df.filter(
        batch_df._change_type.isin(['insert', 'update_postimage'])
    )
    
    # Merge into target
    target = DeltaTable.forName(spark, "production.gold_sales_retail.fact_sales_transaction")
    target.alias("target").merge(
        new_data.alias("source"),
        "target.transaction_id = source.transaction_id"
    ).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()

# Write stream
query = (
    source_changes.writeStream
    .foreachBatch(process_changes)
    .option("checkpointLocation", "/checkpoints/sales_incremental")
    .start()
)

-- Advantages:
✓ Automatic change tracking
✓ Handles all change types (insert, update, delete)
✓ Efficient processing
✓ Exactly-once semantics

-- Disadvantages:
✗ Requires Delta CDF enabled on source
✗ More complex setup
✗ Need checkpoint management
```

**D. HYBRID (Full + Incremental)**:
```
Pattern: Daily incremental, weekly full reconciliation

Daily: Incremental load for performance
Weekly: Full load to catch any missed changes

-- Schedule
Daily 6 AM: Incremental load (fast)
Sunday 2 AM: Full load (comprehensive)

-- Use case:
✓ Best of both worlds
✓ Daily speed + weekly accuracy
✓ Catches any edge cases

-- Implementation:
IF day_of_week = 'Sunday' THEN
    CALL full_load_procedure()
ELSE
    CALL incremental_load_procedure()
END IF;
```

**Output**: Load strategy with implementation approach

### Decision Point 3.2: SCD Strategy for Dimensions

**Question**: How do we track dimension history?

```
SCD TYPE DECISION TREE:

For each dimension attribute, ask:

Q1: Does this attribute ever change?
├─ NO → SCD Type 0 (no tracking needed)
│       Examples: birth_date, original_registration_date
│
└─ YES → Continue to Q2

Q2: Do we need historical values for analysis?
├─ NO → SCD Type 1 (overwrite with new value)
│       Examples: email_address, phone_number, description
│       Use: When only current value matters
│
└─ YES → Continue to Q3

Q3: How much history do we need?
├─ Just current + prior value → SCD Type 3
│       Examples: previous_territory, prior_segment
│       Implementation: Add prior_value columns
│
└─ Full audit trail → SCD Type 2
        Examples: customer_segment, product_category, pricing_tier
        Implementation: New row for each change with effective dates
```

**SCD Type Implementation**:

**Type 1 - Overwrite** (Simplest):
```sql
-- Just UPDATE existing records
MERGE INTO production.gold_customer_360.dim_customer AS target
USING silver.customer_updates AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN UPDATE SET
    email_address = source.email_address,
    phone_number = source.phone_number,
    modified_timestamp = current_timestamp()
WHEN NOT MATCHED THEN INSERT *;
```

**Type 2 - Full History** (Most Common for Gold Layer):
```python
# Efficient SCD Type 2 with Delta Lake
from delta.tables import DeltaTable
from pyspark.sql import functions as F

source_df = spark.table("silver.customer_updates")
target = DeltaTable.forName(spark, "production.gold_customer_360.dim_customer")

# Add hash for change detection
source_df = source_df.withColumn(
    "row_hash",
    F.sha2(F.concat_ws("||", *source_df.columns), 256)
)

# Identify changes
target_df = target.toDF().filter("is_current = true")
changes = (
    source_df.alias("source")
    .join(target_df.alias("target"), "customer_id", "left_outer")
    .where(
        F.col("target.customer_key").isNull() |  # New customer
        (F.col("source.row_hash") != F.col("target.row_hash"))  # Changed
    )
    .select("source.*")
)

# Step 1: Expire old versions
target.alias("target").merge(
    changes.alias("source"),
    "target.customer_id = source.customer_id AND target.is_current = true"
).whenMatchedUpdate(
    set = {
        "is_current": "false",
        "expiration_date": "current_date()",
        "modified_timestamp": "current_timestamp()"
    }
).execute()

# Step 2: Insert new versions
new_versions = (
    changes
    .withColumn("customer_key", F.expr("uuid()"))
    .withColumn("effective_date", F.current_date())
    .withColumn("expiration_date", F.lit(None).cast("date"))
    .withColumn("is_current", F.lit(True))
)

new_versions.write.format("delta").mode("append").saveAsTable(
    "production.gold_customer_360.dim_customer"
)
```

**Type 3 - Current + Prior** (Rare):
```sql
-- Add columns for prior value
ALTER TABLE dim_customer
ADD COLUMNS (
    prior_segment STRING,
    segment_change_date DATE
);

-- Update with current and prior
UPDATE dim_customer AS target
SET 
    prior_segment = target.current_segment,
    segment_change_date = current_date(),
    current_segment = source.segment
FROM customer_updates AS source
WHERE target.customer_id = source.customer_id
AND target.current_segment != source.segment;
```

**Mixed SCD Approach** (Recommended):
```
Different attributes in same dimension can use different SCD types:

dim_customer:
├─ customer_id (Type 0 - never changes)
├─ email_address (Type 1 - overwrite OK)
├─ phone_number (Type 1 - overwrite OK)
├─ customer_segment (Type 2 - track history)
├─ credit_tier (Type 2 - track history)
└─ last_purchase_date (Type 1 - overwrite OK)

Implementation: Use Type 2 for dimension, but only trigger
new version when Type 2 attributes change
```

**SAS to Databricks SCD Migration**:

```
IF SAS maintained history:
├─ With effective dates → Migrate to Type 2
│  └─ Preserve existing history
│  └─ Continue pattern with Delta MERGE
│
├─ Append-only pattern → Convert to Type 2
│  └─ Add proper effective/expiration dates
│  └─ Add is_current flag
│
└─ No clear pattern → Design Type 2 from scratch
   └─ Define which attributes need history
   └─ Implement properly in Databricks

IF SAS did NOT maintain history:
└─ Evaluate business need:
   ├─ Need going forward → Implement Type 2
   └─ Not needed → Use Type 1
```

**Output**: SCD strategy per dimension with implementation pattern

## Phase 4: Testing and Validation Strategy

### Decision Point 4.1: Reconciliation Depth

**Question**: What level of reconciliation is required?

```
THREE-LEVEL RECONCILIATION FRAMEWORK:

LEVEL 1 - RECORD COUNT VALIDATION (Always Required)
├─ Purpose: Ensure no data loss or duplication
├─ Method: Compare row counts between SAS and Databricks
├─ Tolerance: 0% variance for critical tables, < 0.01% for others
└─ Implementation:
    SELECT 
        'SAS' AS source,
        COUNT(*) AS record_count
    FROM bronze.sas_source.dataset
    UNION ALL
    SELECT 
        'Databricks' AS source,
        COUNT(*) AS record_count
    FROM production.gold_sales_retail.fact_sales_transaction;

LEVEL 2 - AGGREGATE VALIDATION (Standard for All)
├─ Purpose: Validate business metrics match
├─ Method: Compare sums, averages, min/max across key dimensions
├─ Tolerance: < 0.01% variance (accounting for rounding)
└─ Implementation:
    -- Compare by customer
    SELECT 
        customer_id,
        SUM(net_amount) AS total_amount,
        COUNT(*) AS transaction_count,
        AVG(net_amount) AS avg_amount
    FROM [source]
    GROUP BY customer_id
    -- Compare results between SAS and Databricks

LEVEL 3 - DETAILED ROW-BY-ROW VALIDATION (For Critical Tables)
├─ Purpose: Identify exact discrepancies
├─ Method: Row-by-row comparison of all columns
├─ Tolerance: 0% for business keys, < $0.01 for amounts
└─ Implementation:
    -- Find mismatches
    SELECT 
        s.transaction_id,
        s.net_amount AS sas_amount,
        d.net_amount AS databricks_amount,
        s.net_amount - d.net_amount AS variance
    FROM sas_output s
    FULL OUTER JOIN databricks_output d
        ON s.transaction_id = d.transaction_id
    WHERE ABS(s.net_amount - d.net_amount) > 0.01
    OR s.transaction_id IS NULL
    OR d.transaction_id IS NULL;
```

**Reconciliation Requirements by Criticality**:

| Criticality | Level 1 | Level 2 | Level 3 | Sample Size |
|-------------|---------|---------|---------|-------------|
| Critical | Required | Required | Required | 100% |
| High | Required | Required | Sample | 50% stratified |
| Medium | Required | Required | Optional | 10% random |
| Low | Required | Sample | No | 5% random |

**Databricks Reconciliation Tools**:

```python
# Reconciliation framework using Delta Lake time travel
def reconcile_tables(sas_table, databricks_table, business_key, measures):
    """
    Comprehensive reconciliation between SAS and Databricks
    """
    # Level 1: Record counts
    sas_count = spark.table(sas_table).count()
    databricks_count = spark.table(databricks_table).count()
    
    count_variance_pct = abs(sas_count - databricks_count) / sas_count * 100
    
    print(f"Level 1 - Record Count:")
    print(f"  SAS: {sas_count:,}")
    print(f"  Databricks: {databricks_count:,}")
    print(f"  Variance: {count_variance_pct:.4f}%")
    
    # Level 2: Aggregate validation
    for measure in measures:
        sas_sum = spark.table(sas_table).selectExpr(f"SUM({measure})").collect()[0][0]
        db_sum = spark.table(databricks_table).selectExpr(f"SUM({measure})").collect()[0][0]
        
        agg_variance_pct = abs(sas_sum - db_sum) / sas_sum * 100
        
        print(f"Level 2 - {measure} Sum:")
        print(f"  SAS: {sas_sum:,.2f}")
        print(f"  Databricks: {db_sum:,.2f}")
        print(f"  Variance: {agg_variance_pct:.4f}%")
    
    # Level 3: Detailed comparison
    comparison = (
        spark.table(sas_table).alias("sas")
        .join(
            spark.table(databricks_table).alias("db"),
            business_key,
            "full_outer"
        )
    )
    
    for measure in measures:
        comparison = comparison.withColumn(
            f"{measure}_variance",
            F.abs(F.col(f"sas.{measure}") - F.col(f"db.{measure}"))
        )
    
    # Find discrepancies
    discrepancies = comparison.filter(
        " OR ".join([f"{measure}_variance > 0.01" for measure in measures])
    )
    
    discrepancy_count = discrepancies.count()
    print(f"Level 3 - Detailed Discrepancies: {discrepancy_count:,}")
    
    if discrepancy_count > 0:
        discrepancies.show(100, truncate=False)
        
    return {
        "count_variance_pct": count_variance_pct,
        "discrepancy_count": discrepancy_count,
        "passed": count_variance_pct < 0.01 and discrepancy_count == 0
    }
```

**Output**: Reconciliation plan with acceptance criteria

### Decision Point 4.2: Testing Approach

**Question**: What testing is required?

```
TESTING PYRAMID FOR SAS MIGRATION:

┌─────────────────────────────┐
│   UAT (Business Testing)    │  ← 1-4 weeks based on criticality
├─────────────────────────────┤
│  Integration Testing        │  ← Test full pipeline
├─────────────────────────────┤
│  Reconciliation Testing     │  ← Compare SAS vs Databricks
├─────────────────────────────┤
│  Unit Testing               │  ← Test individual transformations
└─────────────────────────────┘

TESTING BY COMPLEXITY:

Low Complexity:
├─ Unit Tests: Basic SQL validation
├─ Integration: End-to-end pipeline test
├─ Reconciliation: Level 1 + Level 2 (sample)
└─ UAT: 1 week, key scenarios

Moderate Complexity:
├─ Unit Tests: Comprehensive coverage of logic branches
├─ Integration: Full pipeline with dependencies
├─ Reconciliation: Level 1 + Level 2 (full) + Level 3 (sample)
├─ Performance: Load testing with realistic volumes
└─ UAT: 2 weeks, all user scenarios

High Complexity:
├─ Unit Tests: 100% branch coverage
├─ Integration: All upstream/downstream impacts
├─ Reconciliation: All 3 levels at 100%
├─ Performance: Stress testing, scalability validation
├─ Parallel Run: 2-4 weeks dual processing
└─ UAT: 4 weeks, comprehensive validation

TESTING BY CRITICALITY:

Critical:
├─ All test levels required
├─ Independent validation team
├─ Parallel run: 4+ weeks
├─ Audit/compliance review
└─ Executive sign-off

High:
├─ Standard test levels
├─ Parallel run: 2 weeks
└─ Business owner sign-off

Medium/Low:
├─ Essential tests only
├─ Parallel run: 1 week or none
└─ Technical validation
```

**Databricks Testing Patterns**:

```python
# Example: Unit test for transformation logic
import pytest
from pyspark.sql import SparkSession

def test_sales_transformation():
    """Test sales amount calculations"""
    # Create test data
    test_data = [
        (1, 100.00, 10.00, 5.00),  # extended, discount, tax
        (2, 200.00, 20.00, 10.00)
    ]
    test_df = spark.createDataFrame(
        test_data,
        ["transaction_id", "extended_price", "discount_amount", "tax_amount"]
    )
    
    # Apply transformation
    result = calculate_net_amount(test_df)
    
    # Validate
    expected_amounts = [95.00, 190.00]  # extended - discount + tax
    actual_amounts = [row.net_amount for row in result.collect()]
    
    assert actual_amounts == expected_amounts, "Net amount calculation incorrect"

# Example: Integration test
def test_end_to_end_pipeline():
    """Test complete pipeline from bronze to gold"""
    # Setup test data in bronze
    setup_test_bronze_data()
    
    # Run pipeline
    run_etl_pipeline()
    
    # Validate gold layer results
    gold_df = spark.table("development.gold_sales_retail.fact_sales_transaction")
    
    assert gold_df.count() > 0, "No data in gold layer"
    assert gold_df.filter("net_amount IS NULL").count() == 0, "Null amounts found"
    assert gold_df.filter("customer_key IS NULL").count() == 0, "Null keys found"
```

**Output**: Test plan with pass/fail criteria

## Phase 5: Cutover and Rollback Strategy

### Decision Point 5.1: Cutover Approach

**Question**: How do we transition from SAS to Databricks?

```
CUTOVER STRATEGIES:

1. BIG BANG CUTOVER
   When to use:
   ✓ Low complexity, low criticality
   ✓ Few downstream consumers
   ✓ Short downtime acceptable
   ✓ Clear validation results
   
   Process:
   1. Freeze SAS processing (Friday evening)
   2. Final data load to Databricks
   3. Validation and smoke testing
   4. Switch all consumers to Databricks (Monday morning)
   5. Monitor closely for 1 week
   6. Decommission SAS after stability confirmed
   
   Duration: 1 weekend
   Risk: Medium (all at once)
   
2. PHASED CUTOVER
   When to use:
   ✓ Moderate complexity/criticality
   ✓ Multiple consumer groups
   ✓ Risk mitigation important
   ✓ Can run both systems temporarily
   
   Process:
   Week 1: Migrate read-only consumers (dashboards, reports)
   Week 2: Migrate analytical consumers
   Week 3: Migrate operational consumers
   Week 4: Migrate critical/regulated consumers
   Week 5: Decommission SAS
   
   Duration: 4-6 weeks
   Risk: Low (gradual, controlled)
   
3. PARALLEL RUN
   When to use:
   ✓ High/critical complexity and priority
   ✓ Regulatory requirements
   ✓ Zero downtime requirement
   ✓ Extended validation needed
   
   Process:
   Month 1-2: Both systems running, compare outputs daily
   Month 2-3: Databricks primary, SAS backup, spot checks
   Month 3-4: SAS read-only, final validation
   Month 4: Decommission SAS
   
   Duration: 3-4 months
   Risk: Very Low (extensive validation)
   
4. DARK LAUNCH
   When to use:
   ✓ Mission-critical systems
   ✓ Cannot afford any disruption
   ✓ A/B testing desired
   ✓ Gradual confidence building
   
   Process:
   Phase 1: Databricks running in shadow mode (not used)
   Phase 2: Route 10% of queries to Databricks, compare
   Phase 3: Increase to 50% traffic
   Phase 4: Increase to 90% traffic
   Phase 5: 100% to Databricks, keep SAS warm
   Phase 6: Decommission SAS
   
   Duration: 4-6 months
   Risk: Minimal (very gradual)
```

**Cutover Decision Matrix**:

| Complexity | Criticality | Consumers | → Cutover Strategy |
|------------|-------------|-----------|-------------------|
| Low | Low-Medium | Few (< 5) | Big Bang |
| Low-Moderate | Medium | Moderate (5-10) | Phased |
| Moderate | High | Many (10+) | Parallel Run |
| High | Critical | Any | Parallel Run or Dark Launch |

**Databricks Cutover Implementation**:

```python
# Cutover orchestration notebook
# Controls which system is active

def get_active_system():
    """Determine which system should serve queries"""
    # Read from configuration table
    config = spark.sql("""
        SELECT system, percentage
        FROM production.system_config.cutover_control
        WHERE is_active = true
    """).collect()[0]
    
    return config.system, config.percentage

def route_query(query_str):
    """Route query to SAS or Databricks based on cutover strategy"""
    system, percentage = get_active_system()
    
    if system == "sas":
        return execute_sas_query(query_str)
    elif system == "databricks":
        return spark.sql(query_str)
    elif system == "split":
        # Dark launch: route percentage to Databricks
        import random
        if random.random() < (percentage / 100):
            return spark.sql(query_str)
        else:
            return execute_sas_query(query_str)

# Configuration table for cutover control
CREATE TABLE production.system_config.cutover_control (
    config_id INT,
    system STRING,  -- 'sas', 'databricks', 'split'
    percentage INT,  -- For split routing
    is_active BOOLEAN,
    updated_timestamp TIMESTAMP,
    updated_by STRING
);
```

**Output**: Cutover plan with timeline and rollback triggers

### Decision Point 5.2: Rollback Planning

**Question**: What is our rollback strategy?

```
ROLLBACK CAPABILITY LEVELS:

LEVEL 1 - IMMEDIATE ROLLBACK (< 1 hour)
Requirements:
✓ SAS environment maintained and operational
✓ Ability to instantly redirect consumers
✓ No irreversible changes in Databricks
✓ DNS/connection string switchback capability

When needed:
• Critical production issue discovered
• Data quality failure
• Severe performance degradation
• Security incident

Implementation:
1. Update connection strings back to SAS
2. Notify all users of rollback
3. Document issue for root cause analysis
4. Plan remediation

LEVEL 2 - SAME-DAY ROLLBACK (< 8 hours)
Requirements:
✓ SAS environment available
✓ Databricks changes are reversible
✓ Data can be reconciled
✓ Documented rollback procedures

When needed:
• Incorrect results discovered
• Unforeseen business impact
• Consumer integration issues
• Compliance concerns

Implementation:
1. Assess impact and scope
2. Execute rollback procedures
3. Re-validate SAS outputs
4. Communicate to stakeholders
5. Investigation and remediation plan

LEVEL 3 - NO EASY ROLLBACK (> 1 day)
Situation:
• SAS decommissioned
• Irreversible transformations applied
• Historical data affected
• Complex consumer dependencies

Mitigation (Prevention):
→ Extended parallel run period
→ Comprehensive pre-cutover testing
→ Phased cutover approach
→ Intensive hypercare support
→ Gradual SAS decommissioning
```

**Rollback Decision Matrix**:

| Criticality | Rollback Level Required | Maintenance Period |
|-------------|------------------------|-------------------|
| Critical | Level 1 | 4 weeks post-cutover |
| High | Level 2 | 2 weeks post-cutover |
| Medium | Level 2 | 1 week post-cutover |
| Low | Level 3 | No specific requirement |

**Databricks Rollback Features**:

```sql
-- Use Delta Lake time travel for data rollback
-- Restore table to version before cutover
RESTORE TABLE production.gold_sales_retail.fact_sales_transaction
TO VERSION AS OF 42;  -- Version before cutover

-- Or restore to timestamp
RESTORE TABLE production.gold_sales_retail.fact_sales_transaction
TO TIMESTAMP AS OF '2024-01-15 00:00:00';

-- View history to identify correct version
DESCRIBE HISTORY production.gold_sales_retail.fact_sales_transaction;

-- Clone table for safety before making changes
CREATE TABLE production.gold_sales_retail.fact_sales_transaction_backup
SHALLOW CLONE production.gold_sales_retail.fact_sales_transaction
VERSION AS OF 42;
```

**Rollback Checklist**:

```
Pre-Cutover:
- [ ] Document rollback procedures
- [ ] Test rollback in lower environment
- [ ] Define rollback triggers (what constitutes rollback scenario)
- [ ] Identify rollback decision makers
- [ ] Set up monitoring and alerting
- [ ] Maintain SAS environment in operational state
- [ ] Document connection string changes needed
- [ ] Create Delta table version snapshots

During Cutover:
- [ ] Monitor key metrics continuously
- [ ] Have rollback team on standby
- [ ] Document table versions at cutover
- [ ] Maintain detailed log of changes
- [ ] Keep communication channels open

Post-Cutover:
- [ ] Continue monitoring for [X] weeks
- [ ] Keep SAS available for [Y] period
- [ ] Review rollback triggers daily
- [ ] Document any issues encountered
- [ ] Update rollback procedures based on learnings
```

**Output**: Rollback plan with triggers and procedures

## Decision Documentation Template

Document all migration decisions using this template:

```markdown
# SAS to Databricks Migration Decision Record

## Asset Information
**SAS Program**: sales_monthly_rollup.sas
**SAS Datasets**: 
  - Input: sales.daily_transactions, crm.customers
  - Output: sales.monthly_summary
**Business Owner**: Jane Smith (jane.smith@company.com)
**Technical Owner**: Data Engineering Team
**Migration Wave**: Wave 2 - Q1 2024

## Complexity Assessment
**Total Score**: 15 / 27
**Classification**: MODERATE COMPLEXITY
**Estimated Duration**: 1-2 weeks
**Resource Allocation**: 2 developers + reviewer

### Scoring Detail:
- Lines of Code: 280 lines → 2 pts
- Input Sources: 2 tables → 1 pt
- Transformations: Aggregations + joins → 2 pts
- Macros: None → 1 pt
- Dependencies: 2 upstream jobs → 1 pt
- Output Datasets: 1 dataset → 1 pt
- Business Logic: Well documented → 2 pts
- Data Volume: 50M rows → 2 pts
- SAS Features: PROC SQL + PROC MEANS → 2 pts

## Business Criticality: HIGH
**Rationale**: Monthly executive dashboard, no tolerance for late data
**Testing Requirements**: 
- Parallel run: 2 weeks
- Full reconciliation (Levels 1-2, sample Level 3)
- Business owner sign-off required

## Migration Decisions

### 1. Translation Approach: DIRECT TRANSLATION
**Rationale**: 
- Well-documented SAS code with good performance
- No business logic changes required
- Tight timeline (6 weeks to delivery)
- Logic maps cleanly to Spark SQL

### 2. Databricks Technology Stack:
**Table Type**: Aggregate fact table (agg_sales_customer_monthly)
**Load Strategy**: MERGE (incremental, update existing months)
**Organization**: LIQUID CLUSTERING on (month_date_key, customer_key)
**Processing**: BATCH (daily refresh at 2 AM)
**Language**: Spark SQL (team strength, simple aggregation logic)
**CDF**: Enabled (downstream ML models need changes)

### 3. Load Pattern: INCREMENTAL with Monthly Full Reconciliation
**Rationale**:
- 50M base rows, ~5M changes daily
- Monthly snapshot pattern aligns with business process
- Full monthly reconciliation catches any edge cases

### 4. SCD Strategy: N/A (Aggregate table, not dimension)

### 5. Testing Approach: STANDARD PLUS
**Unit Tests**: SQL validation of aggregation logic
**Integration Tests**: End-to-end with dependencies
**Reconciliation**: 
  - Level 1: 100% (count validation)
  - Level 2: 100% (sum by customer and month)
  - Level 3: 10% sample (detailed row comparison)
**Performance**: Load test with full month data
**UAT**: 2 weeks with business team

### 6. Cutover Strategy: PHASED
**Timeline**:
- Week 1: Databricks shadow mode (parallel comparison)
- Week 2: Switch dashboard to Databricks, SAS backup
- Week 3: Validate with business, monitor issues
- Week 4: Decommission SAS pipeline

**Rollback Capability**: Level 2 (same-day rollback, 2-week maintenance)

## Risk Assessment

### High Risks
1. **Risk**: Month-end timing sensitivity
   **Mitigation**: Schedule cutover mid-month, not at month boundary
   
2. **Risk**: Dashboard dependencies not fully mapped
   **Mitigation**: Comprehensive downstream impact analysis before cutover

### Medium Risks
1. **Risk**: Historical data comparison complexity
   **Mitigation**: Focus reconciliation on most recent 6 months

## Dependencies
**Upstream**: 
- silver.sales_transactions (migrated in Wave 1)
- gold_customer_360.dim_customer (migrated in Wave 1)

**Downstream**: 
- Executive Sales Dashboard (Power BI)
- Sales Forecast ML Model (Databricks ML)
- Monthly Management Reports (5 reports identified)

## Implementation Notes

### Databricks Catalog Structure:
```
production.gold_sales_retail.agg_sales_customer_monthly
```

### Table Properties:
```sql
TBLPROPERTIES (
    'sas_source_program' = 'sales_monthly_rollup.sas',
    'sas_source_dataset' = 'sales.monthly_summary',
    'business_owner' = 'jane.smith@company.com',
    'data_classification' = 'confidential',
    'delta.enableChangeDataFeed' = 'true'
)
```

### Liquid Clustering:
```sql
CLUSTER BY (month_date_key, customer_key)
```

## Approvals

| Role | Name | Date | Status |
|------|------|------|--------|
| Data Architect | John Doe | 2024-01-15 | Approved |
| Business Owner | Jane Smith | 2024-01-16 | Approved |
| Technical Lead | Bob Johnson | 2024-01-15 | Approved |

## Revision History

| Date | Change | Author |
|------|--------|--------|
| 2024-01-15 | Initial decision record | Data Engineering Team |
| 2024-01-20 | Updated cutover timeline | Project Manager |
```

## Escalation Criteria

Escalate to Data Architecture Team if:

- [ ] Complexity score > 20 (High complexity)
- [ ] Critical business criticality with tight timeline
- [ ] Circular dependencies identified
- [ ] Business logic cannot be understood/validated
- [ ] Major discrepancies in reconciliation (> 0.1%)
- [ ] SAS uses unsupported features
- [ ] Performance requirements cannot be met with standard approach
- [ ] Security or compliance concerns
- [ ] Budget/timeline overrun > 25%
- [ ] Breaking changes impact > 5 downstream consumers

## Best Practices Summary

### Do's ✅

1. **Always start with complexity assessment** - Set realistic expectations
2. **Document SAS source in metadata** - Traceability is critical
3. **Use Liquid Clustering by default** - Better than partitioning for most cases
4. **Enable Change Data Feed** - Future-proof for downstream consumers
5. **Reconcile at all 3 levels** - Appropriate to criticality
6. **Implement SCD Type 2 for dimensions** - Business needs history
7. **Use Delta MERGE for upserts** - Efficient and ACID-compliant
8. **Test rollback procedures** - Hope for best, plan for worst
9. **Parallel run for critical data** - Confidence through validation
10. **Document everything** - Migration decisions and rationale

### Don'ts ❌

1. **Don't skip complexity assessment** - Leads to missed timelines
2. **Don't ignore dependencies** - Causes cascading failures
3. **Don't cut corners on testing** - Technical debt compounds
4. **Don't assume SAS logic is correct** - Validate with SMEs
5. **Don't rush cutover** - Preparation prevents problems
6. **Don't forget to enable CDF** - Hard to add later
7. **Don't use traditional partitioning** - Liquid Clustering is better
8. **Don't skip documentation** - Future you will thank present you
9. **Don't work in isolation** - Collaboration catches issues early
10. **Don't decommission SAS too early** - Keep rollback option open

## Quick Reference Decision Flow

```
START: New SAS Asset to Migrate
         ↓
[1] Classify Asset & Score Complexity
         ↓ ← Document in decision record
[2] Assess Business Criticality
         ↓ ← Determine testing requirements
[3] Select Migration Pattern
    ├─ Low complexity + good code → Direct Translation
    ├─ Moderate → Hybrid Approach
    └─ High complexity or poor code → Re-Architecture
         ↓
[4] Choose Databricks Technologies
    ├─ MERGE vs INSERT
    ├─ Liquid Clustering (default)
    ├─ Batch vs Streaming
    └─ Enable CDF
         ↓
[5] Define Load Strategy
    ├─ Volume < 1M → Full Load
    ├─ Volume > 100M → Incremental (required)
    └─ Volume 1M-100M → Evaluate change rate
         ↓
[6] Define SCD Strategy (for dimensions)
    ├─ Type 1: Overwrite (email, phone)
    ├─ Type 2: History (segment, tier)
    └─ Type 0: Never changes (birth date)
         ↓
[7] Plan Testing & Reconciliation
    ├─ Critical → All 3 levels, 100%
    ├─ High → Levels 1-2 full, Level 3 sample
    └─ Medium/Low → Levels 1-2 only
         ↓
[8] Select Cutover Strategy
    ├─ Low/Low → Big Bang
    ├─ Moderate → Phased
    └─ High/Critical → Parallel Run
         ↓
[9] Define Rollback Plan
    ├─ Critical → Level 1 (instant rollback)
    ├─ High → Level 2 (same-day rollback)
    └─ Medium/Low → Level 3 (no specific requirement)
         ↓
[10] Document & Get Approvals
    ├─ Use decision record template
    ├─ Data Architect approval
    ├─ Business Owner approval
    └─ Technical Lead approval
         ↓
END: Ready for Implementation
```

---

**Document Control**

| Attribute | Value |
|-----------|-------|
| Version | 1.0 (Databricks Edition) |
| Last Updated | January 2026 |
| Owner | Data Architecture Team |
| Target Platform | Databricks Lakehouse |
| Review Date | April 2026 |
| Related Docs | Gold Layer Master, Naming Conventions, Modeling Approach |
