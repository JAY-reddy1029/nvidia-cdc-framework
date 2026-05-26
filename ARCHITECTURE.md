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

---

## Data Flow

┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   SOURCE        │         │   PIPELINE       │         │    OUTPUT       │
│   (SAP-like)    │────────▶│   (Dataform)     │────────▶│   (Analytics)   │
│   raw_layer     │         │   transformations│         │   cdc_layer     │
└─────────────────┘         └─────────┬────────┘         └─────────────────┘
│
│ logs to
▼
┌─────────────────┐
│   MONITORING    │
│   audit_layer   │
│   (7 tables)    │
└─────────────────┘

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
- `customers` — Customer master data
- `orders` — Sales orders with CDC events

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

**Tables:**
- `customers` — Full Load pattern (drops and recreates)
- `orders` — Delta Load with CDC handling
- `orders_backfill_staging` — Temporary staging for backfill operations

**Design Rationale:** This is the silver layer where data becomes usable. Business users query this layer, not raw_layer.

---

### Layer 3: audit_layer (Operational)

**Purpose:** Comprehensive operational monitoring and observability.

**Design Rationale:** Production data engineering requires complete observability. When pipelines fail at 3 AM, engineers need to query these tables to understand what happened. Each table serves a specific operational need.

#### 3.1 pipeline_audit
Tracks every pipeline run. Populated automatically by post_operations blocks.
- Key columns: pipeline_name, start_time, end_time, rows_processed, status
- Use case: "Did yesterday's pipeline run successfully?"

#### 3.2 error_log
Categorized errors with severity. Manual or automated insertion.
- Categories: SYNTAX, DATA_QUALITY, SCHEMA_DRIFT, PERMISSION
- Severity: CRITICAL, ERROR, WARNING
- Use case: "What critical errors happened this week?"

#### 3.3 duplicate_log
Detected duplicates in source data. Populated by pre_operations.
- Types: EXACT, LOGICAL, CONFLICTING
- Use case: "Are there data quality issues at the source?"

#### 3.4 backfill_log
Historical reprocessing operations.
- Tracks: date range, reason, requestor, status
- Use case: "Why was data backfilled last week and by whom?"

#### 3.5 schema_registry
Source of truth for expected schemas.
- Used by schema drift detection
- Use case: "What columns should cdc_layer.orders have?"

#### 3.6 schema_drift_log
Detected schema mismatches.
- Drift types: MISSING_COLUMN, EXTRA_COLUMN, TYPE_MISMATCH
- Severity: CRITICAL, WARNING
- Use case: "Did anyone modify our tables without telling us?"

#### 3.7 pipeline_checkpoint
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

## Pipeline Patterns

### Pattern 1: Full Load (cdc_customers)

Used for small, slowly-changing master data.

Drop existing cdc_layer.customers
SELECT all rows from raw_layer.customers
Add cdc_processed_at timestamp
Create cdc_layer.customers
Run assertions (uniqueKey, nonNull, rowConditions)
Log to pipeline_audit (post_operations)

**Trade-off:** Simple but inefficient for large tables.

---

### Pattern 2: Delta Load with CDC (cdc_orders)

Used for transaction data with frequent changes.

Detect duplicates in raw data (pre_operations) → duplicate_log
Read raw_layer.orders
Apply ROW_NUMBER() PARTITION BY order_id ORDER BY last_updated DESC
Filter WHERE rn = 1 (keep latest version)
Filter WHERE changetype != 'D' (remove deletes)
Add cdc_processed_at timestamp
Write to cdc_layer.orders
Run assertions
Log to pipeline_audit (post_operations)

**Trade-off:** Handles CDC complexity correctly but requires more compute.

---

### Pattern 3: Backfill (Staging + MERGE)

Used for safe historical reprocessing.Step 1: cdc_orders_backfill.sqlx

Read raw_layer.orders filtered by date range from vars
Apply same deduplication logic
Write to cdc_layer.orders_backfill_staging
Log to backfill_log as IN_PROGRESS
Step 2: cdc_orders_backfill_merge.sqlx

MERGE staging into cdc_layer.orders
WHEN MATCHED → UPDATE
WHEN NOT MATCHED → INSERT
Update backfill_log to COMPLETED

**Design Rationale:** Staging table provides safety. If merge fails, staging data is preserved for retry.

---

### Pattern 4: Schema Drift Detection

Runs as a separate operation, not blocking main pipelines.

Read expected columns from audit_layer.schema_registry
Read actual columns from cdc_layer.INFORMATION_SCHEMA.COLUMNS
FULL OUTER JOIN to find differences
Categorize drift (MISSING, EXTRA, TYPE_MISMATCH)
Assign severity (CRITICAL, WARNING)
Insert findings into schema_drift_log

**Design Rationale:** Uses BigQuery's built-in metadata for actual schemas. Registry provides expected. FULL OUTER JOIN catches both directions of drift.

---

## Design Decisions

### Why Three Datasets (raw, cdc, audit)?

| Dataset | Why |
|---------|-----|
| raw_layer | Single source of truth, never modified |
| cdc_layer | Clean data for consumers, can be rebuilt anytime |
| audit_layer | Operational data separate from business data |

### Why Use Dataform's "operations" Type?

For DML statements (MERGE, INSERT, UPDATE) that don't create tables:
- schema_drift_check
- cdc_orders_backfill_merge

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

---

## Dependencies

### File Dependencies (Dataform refs)

raw_layer.customers ───▶ cdc_layer.customers
raw_layer.orders ───▶ cdc_layer.orders
raw_layer.orders ───▶ cdc_layer.orders_backfill_staging
cdc_layer.orders_backfill_staging ───▶ cdc_layer.orders (via MERGE)
audit_layer.schema_registry ───▶ schema_drift_check (read)

### Execution Order (Daily Pipeline)

cdc_customers (Full Load)
cdc_orders (Delta Load)
schema_drift_check (after both)
Assertions run automatically per table

---

## Scalability Considerations

### Current Scale (Development)
- raw_layer.orders: 27 rows
- cdc_layer.orders: 13 rows

### Production Scale (NVIDIA-like)
- raw_layer tables: Millions of rows
- Pipelines need: Partitioning, clustering, incremental processing

### Future Optimizations Needed
1. Add `incremental` type for very large tables
2. Partition by order_date for time-based queries
3. Cluster by customer_id for filtered queries
4. Add slot reservations for predictable costs

---

## Security Considerations

### Current State (Development)
- All access controlled at GCP project level
- Owner role: project owner (you)

### Production Requirements
- IAM roles per dataset (read-only vs read-write)
- Column-level security for PII
- Audit logging via Cloud Audit Logs
- Data classification labels on tables

---

## Future Enhancements

### Phase 4: Analytics Layer
- Looker Studio dashboards
- BigQuery views for common queries
- Aggregation tables for performance

### Phase 5: Orchestration
- Cloud Composer (Airflow) DAGs
- Scheduled daily runs
- Slack/email alerting

### Phase 6: CI/CD
- Git-based deployment
- Cloud Build pipelines
- Dev/staging/prod environments

### Phase 7: Advanced Features
- SCD Type 2 for historical tracking
- Real-time CDC via Pub/Sub
- ML-based anomaly detection on audit logs

---

## References

- [Dataform Documentation](https://cloud.google.com/dataform/docs)
- [BigQuery Best Practices](https://cloud.google.com/bigquery/docs/best-practices-performance-overview)
- [Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [CDC Patterns](https://en.wikipedia.org/wiki/Change_data_capture)

---

## Document Metadata

- **Author:** Jayachandra Reddy (Jay)
- **Created:** May 26, 2026
- **Status:** Living document
