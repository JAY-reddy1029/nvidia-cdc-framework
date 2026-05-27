# Architecture Documentation

Technical design document for the CDC Framework on Google Cloud Platform.

---

## System Overview

This framework implements a layered data architecture pattern (medallion-style) for processing CDC events from source systems to analytics-ready data, with comprehensive operational monitoring.

### Core Design Principles

1. **Separation of Concerns** — Raw data, transformed data, and operational data live in separate datasets
2. **Single Source of Truth** — Raw data is never modified; pipelines only read from it
3. **Audit Everything** — Every operation is logged for traceability
4. **Fail Loud** — Pipelines fail visibly when data quality issues are detected
5. **Recovery First** — Built-in mechanisms for backfill and resume from failure
6. **History Preservation** — SCD Type 2 for dimensional data changes

---

## Data Flow

The data flows in one direction from source to output, with operational events logged to a separate monitoring layer.

SOURCE (raw_layer) -> PIPELINE (Dataform) -> OUTPUT (cdc_layer)
|
v
MONITORING (audit_layer)
|
v
ANALYTICS (analytics_layer)

---

## Layer Architecture

### Layer 1: raw_layer (Source)

**Purpose:** Holds source data exactly as received from upstream systems.

**Characteristics:**
- Read-only for pipelines
- Mirrors source schema exactly
- No transformations applied
- Acts as single source of truth
- Allows reprocessing without re-extracting from source

**Tables:**
- customers - Customer master data
- orders - Sales orders with CDC events

**Design Rationale:** By keeping raw data untouched, we can always reprocess if pipeline logic changes or bugs are discovered. This is the medallion architecture's bronze layer.

---

### Layer 2: cdc_layer (Transformed)

**Purpose:** Holds clean, business-ready data for analytics and downstream consumers.

**Characteristics:**
- Deduplicated using ROW_NUMBER deduplication
- Deletes filtered (changetype = 'D')
- Enriched with cdc_processed_at timestamp
- Validated by Dataform assertions
- Schema controlled by schema_registry
- Optimized with partitioning + clustering

**Tables:**
- customers - Full Load pattern with clustering by country, customer_type
- orders - Delta Load with partitioning by order_date, clustering by customer_id, status
- orders_backfill_staging - Temporary staging for backfill operations
- dim_customers_scd2 - Slowly Changing Dimension Type 2 (preserves full history)

**Design Rationale:** This is the silver layer where data becomes usable. Business users query this layer, not raw_layer.

---

### Layer 3: audit_layer (Operational)

**Purpose:** Comprehensive operational monitoring and observability.

**Design Rationale:** Production data engineering requires complete observability. When pipelines fail at 3 AM, engineers need to query these tables to understand what happened. Each table serves a specific operational need.

**Table 3.1: pipeline_audit**
Tracks every pipeline run. Populated automatically by post_operations blocks.
- Key columns: pipeline_name, start_time, end_time, rows_processed, status
- Use case: "Did yesterday's pipeline run successfully?"

**Table 3.2: error_log**
Categorized errors with severity. Manual or automated insertion.
- Categories: SYNTAX, DATA_QUALITY, SCHEMA_DRIFT, PERMISSION
- Severity: CRITICAL, ERROR, WARNING
- Use case: "What critical errors happened this week?"

**Table 3.3: duplicate_log**
Detected duplicates in source data. Populated by pre_operations.
- Types: EXACT, LOGICAL, CONFLICTING
- Use case: "Are there data quality issues at the source?"

**Table 3.4: backfill_log**
Historical reprocessing operations.
- Tracks: date range, reason, requestor, status
- Use case: "Why was data backfilled last week and by whom?"

**Table 3.5: schema_registry**
Source of truth for expected schemas.
- Used by schema drift detection
- Use case: "What columns should cdc_layer.orders have?"

**Table 3.6: schema_drift_log**
Detected schema mismatches.
- Drift types: MISSING_COLUMN, EXTRA_COLUMN, TYPE_MISMATCH
- Severity: CRITICAL, WARNING
- Use case: "Did anyone modify our tables without telling us?"

**Table 3.7: pipeline_checkpoint**
Per-table execution status for resume-from-failure.
- Status: PENDING, IN_PROGRESS, COMPLETED, FAILED, SKIPPED
- Use case: "Which tables need to run after the pipeline failed?"

---

### Layer 4: dataform_assertions (Quality Failures)

**Purpose:** Auto-populated when data quality assertions fail.

**Characteristics:**
- Created automatically by Dataform
- Each assertion failure creates a table containing violating rows
- Empty dataset means data is clean
- Engineers query failed rows to understand quality issues

**Design Rationale:** Failed rows are isolated for inspection without polluting the main data tables.

---

### Layer 5: analytics_layer (Business-Ready)

**Purpose:** Business-friendly views and aggregated tables for analytics consumption.

**Characteristics:**
- Separate from cdc_layer to abstract pipeline complexity
- Includes both live views and pre-computed aggregation tables
- Aggregation tables are partitioned for performance
- Business users query this layer for analytics

**Views (Live Queries):**
- customer_summary - Customer 360 view with order stats
- daily_orders_summary - Daily order aggregations
- product_performance - Product-level metrics
- country_performance - Country-level metrics
- pipeline_health_dashboard - Pipeline operations metrics

**Aggregation Tables (Pre-computed):**
- daily_business_metrics - Partitioned by metric_date, clustered by country
- monthly_business_summary - Partitioned by month
- pipeline_health_metrics - Partitioned by metric_date

**Cost Monitoring Views:**
- recent_query_activity - Last 24 hours of queries
- daily_query_costs - Cost trends over time
- top_expensive_queries - Most expensive queries

**Design Rationale:** Separation of analytics from operational data allows independent optimization and access control.

---

## Pipeline Patterns

### Pattern 1: Full Load (cdc_customers)

Used for small, slowly-changing master data.

Steps:
1. Drop existing cdc_layer.customers
2. SELECT all rows from raw_layer.customers
3. Add cdc_processed_at timestamp
4. Create cdc_layer.customers (with clustering optimization)
5. Run assertions (uniqueKey, nonNull, rowConditions)
6. Log to pipeline_audit (post_operations)

Optimization: Clustered by country, customer_type for filtered queries.

**Trade-off:** Simple but inefficient for large tables.

---

### Pattern 2: Delta Load with CDC (cdc_orders)

Used for transaction data with frequent changes.

Steps:
1. Detect duplicates in raw data (pre_operations) -> duplicate_log
2. Read raw_layer.orders
3. Apply ROW_NUMBER() PARTITION BY order_id ORDER BY last_updated DESC
4. Filter WHERE rn = 1 (keep latest version)
5. Filter WHERE changetype != 'D' (remove deletes)
6. Add cdc_processed_at timestamp
7. Write to cdc_layer.orders (with partitioning + clustering)
8. Run assertions
9. Log to pipeline_audit (post_operations)

Optimization: Partitioned by order_date, clustered by customer_id, status.

**Trade-off:** Handles CDC complexity correctly but requires more compute.

---

### Pattern 3: Backfill (Staging + MERGE)

Used for safe historical reprocessing.

Step A: cdc_orders_backfill.sqlx
- Read raw_layer.orders filtered by date range from vars
- Apply same deduplication logic
- Write to cdc_layer.orders_backfill_staging
- Log to backfill_log as IN_PROGRESS

Step B: cdc_orders_backfill_merge.sqlx
- MERGE staging into cdc_layer.orders
  - WHEN MATCHED -> UPDATE
  - WHEN NOT MATCHED -> INSERT
- Update backfill_log to COMPLETED

**Design Rationale:** Staging table provides safety. If merge fails, staging data is preserved for retry.

---

### Pattern 4: Schema Drift Detection

Runs as a separate operation, not blocking main pipelines.

Steps:
1. Read expected columns from audit_layer.schema_registry
2. Read actual columns from cdc_layer.INFORMATION_SCHEMA.COLUMNS
3. FULL OUTER JOIN to find differences
4. Categorize drift (MISSING, EXTRA, TYPE_MISMATCH)
5. Assign severity (CRITICAL, WARNING)
6. Insert findings into schema_drift_log

**Design Rationale:** Uses BigQuery's built-in metadata for actual schemas. Registry provides expected. FULL OUTER JOIN catches both directions of drift.

---

### Pattern 5: SCD Type 2 (Slowly Changing Dimensions)

Used for tracking historical changes to dimension data (customers, products, etc.).

Two-step MERGE pattern:

Step 1 (MERGE statement):
- New customers -> INSERT new row (is_current = TRUE)
- Changed customers -> UPDATE existing row (valid_to = today, is_current = FALSE)
- Unchanged customers -> No action

Step 2 (INSERT statement):
- For customers just closed in Step 1
- INSERT new current version with updated data
- Idempotency check: skip if current version already exists

Step 3 (LOG):
- Insert entry into pipeline_audit

Key columns added:
- scd_id - Unique ID per version
- valid_from - When this version became active
- valid_to - When this version was replaced (9999-12-31 for current)
- is_current - TRUE for current version, FALSE for historical
- scd_action - INSERT / UPDATE / CLOSED
- scd_change_reason - Why the change happened

**Design Rationale:**
- Two steps required because BigQuery's MERGE cannot INSERT on MATCHED rows
- Idempotency check prevents duplicates on re-runs
- Time-travel queries enabled: WHERE as_of_date BETWEEN valid_from AND valid_to
- Industry-standard pattern at companies like NVIDIA, Snowflake, Netflix

---

## Design Decisions

### Why Five Datasets?

- raw_layer: Single source of truth, never modified
- cdc_layer: Clean data for consumers, can be rebuilt anytime
- audit_layer: Operational data separate from business data
- dataform_assertions: Quality failures isolated
- analytics_layer: Business consumption abstracted from pipelines

### Why Use Dataform's operations Type?

For DML statements (MERGE, INSERT, UPDATE) that don't create tables:
- schema_drift_check
- cdc_orders_backfill_merge
- dim_customers_scd2_merge

This avoids creating unnecessary tables for what is essentially a maintenance operation.

### Why Staging Table for Backfill?

If we merged directly from raw_layer to cdc_layer.orders and failed midway, the main table could be in an inconsistent state. The staging pattern:
1. Isolates risky operations
2. Provides rollback capability
3. Allows inspection before merge
4. Standard pattern at companies like Netflix, Snowflake

### Why FULL OUTER JOIN for Schema Drift?

Need to detect:
- Columns in registry but missing from table (LEFT side)
- Columns in table but missing from registry (RIGHT side)
- Columns in both with different types

A FULL OUTER JOIN returns all three cases. INNER JOIN would miss missing/extra columns.

### Why SCD Type 2 Instead of Type 1?

- Type 1 (overwrite) loses history
- Type 2 (versions) preserves full history with time-travel capability
- Required for audit compliance and historical reporting
- Industry standard for dimension tables

### Why Partitioning + Clustering?

- Partitioning: Physical split for time-based queries (massive cost savings at scale)
- Clustering: Sort within partitions for filtered queries
- Combined: 99%+ cost reduction on common queries at scale

---

## Dependencies

### File Dependencies (Dataform refs)

- raw_layer.customers -> cdc_layer.customers
- raw_layer.orders -> cdc_layer.orders
- raw_layer.orders -> cdc_layer.orders_backfill_staging
- cdc_layer.orders_backfill_staging -> cdc_layer.orders (via MERGE)
- raw_layer.customers -> cdc_layer.dim_customers_scd2 (via MERGE)
- audit_layer.schema_registry -> schema_drift_check (read)

### Execution Order (Daily Pipeline)

1. cdc_customers (Full Load)
2. cdc_orders (Delta Load)
3. dim_customers_scd2_merge (SCD Type 2 update)
4. schema_drift_check (after main tables)
5. Assertions run automatically per table

---

## Optimization Strategy

### Tables Optimized (Production-Grade)

- cdc_layer.orders: PARTITION BY order_date, CLUSTER BY customer_id, status
- cdc_layer.customers: CLUSTER BY country, customer_type
- analytics_layer.daily_business_metrics: PARTITION BY metric_date
- analytics_layer.monthly_business_summary: PARTITION BY month_start_date
- analytics_layer.pipeline_health_metrics: PARTITION BY metric_date

### Query Optimization Rules

1. Always filter on partition column for partition pruning
2. SELECT specific columns instead of SELECT *
3. Use clustering columns in WHERE clauses
4. LIMIT during exploration
5. Avoid functions on partition columns

### Cost Monitoring

INFORMATION_SCHEMA.JOBS_BY_USER queries provide:
- Recent query activity tracking
- Daily cost trends
- Top expensive queries identification

---

## Scalability Considerations

### Current Scale (Development)
- raw_layer.orders: ~20-30 rows
- cdc_layer.orders: ~13 rows
- dim_customers_scd2: ~10-15 rows with versions

### Production Scale (NVIDIA-like)
- raw_layer tables: Millions of rows
- Pipelines need: Partitioning, clustering, incremental processing
- Dimension tables: Hundreds of thousands of versions over time

### Future Optimizations Needed
1. Add incremental type for very large tables
2. Partition by order_date for time-based queries (DONE)
3. Cluster by customer_id for filtered queries (DONE)
4. Add slot reservations for predictable costs
5. Materialize frequently-queried analytics views

---

## Security Considerations

### Current State (Development)
- All access controlled at GCP project level
- Owner role: project owner

### Production Requirements
- IAM roles per dataset (read-only vs read-write)
- Column-level security for PII
- Audit logging via Cloud Audit Logs
- Data classification labels on tables

---

## Future Enhancements

### Phase 6: Orchestration
- Cloud Composer (Airflow) DAGs
- Scheduled daily runs
- Slack/email alerting

### Phase 7: CI/CD
- Git-based deployment
- Cloud Build pipelines
- Dev/staging/prod environments

### Phase 8: Advanced Features
- SCD Type 2 for products and other dimensions
- Real-time CDC via Pub/Sub
- ML-based anomaly detection on audit logs

---

## References

- Dataform Documentation: https://cloud.google.com/dataform/docs
- BigQuery Best Practices: https://cloud.google.com/bigquery/docs/best-practices-performance-overview
- Medallion Architecture: https://www.databricks.com/glossary/medallion-architecture
- CDC Patterns: https://en.wikipedia.org/wiki/Change_data_capture
- Slowly Changing Dimensions: https://en.wikipedia.org/wiki/Slowly_changing_dimension

---

## Document Metadata

- **Author:** Jayachandra Reddy (Jay)
- **Created:** May 26, 2026
- **Last Updated:** May 27, 2026
- **Status:** Living document

- 
