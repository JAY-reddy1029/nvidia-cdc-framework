# Pipeline Documentation

Detailed documentation of every pipeline file in the CDC Framework.

---

## Overview

The framework contains 9 SQLX files organized into two folders:

- definitions/sources/ - Source table declarations
- definitions/transformations/ - Data transformation logic

Configuration is centralized in workflow_settings.yaml.

---

## File: workflow_settings.yaml

**Type:** Configuration file
**Purpose:** Project-level settings and variables

**Contents:**
- defaultProject: GCP project ID
- defaultDataset: Default output dataset (cdc_layer)
- defaultLocation: Region (asia-south1)
- dataformCoreVersion: Dataform version
- vars: Variables for dynamic pipelines

**Variables Defined:**
- backfill_start_date: Start of backfill range
- backfill_end_date: End of backfill range
- backfill_reason: Why backfill is needed
- backfill_requested_by: Who requested it

**Why it matters:** Variables defined here are referenced in SQLX using ${dataform.projectConfig.vars.X}. This makes pipelines configurable without code changes.

---

## File: definitions/sources/raw_customers.sqlx

**Type:** declaration
**Output Table:** None (just a pointer)
**Purpose:** Declares that raw_layer.customers exists

**Why we need it:** Without declaration files, we cannot use ${ref()} to reference source tables in transformation files.

---

## File: definitions/sources/raw_orders.sqlx

**Type:** declaration
**Output Table:** None
**Purpose:** Declares that raw_layer.orders exists

**Why we need it:** To reference raw_layer.orders in cdc_orders.sqlx using ${ref({schema: "raw_layer", name: "orders"})}.

---

## File: definitions/transformations/cdc_customers.sqlx

**Type:** table (Full Load with clustering)
**Output Table:** cdc_layer.customers
**Purpose:** Transforms raw customers data to clean CDC layer with optimization

### What It Does

1. Reads all rows from raw_layer.customers
2. Adds cdc_processed_at timestamp column
3. Writes to cdc_layer.customers (drops and recreates with clustering)
4. Runs assertions (uniqueKey, nonNull, rowConditions)
5. Logs execution to pipeline_audit (post_operations)

### Optimization

- Clustered by country, customer_type
- No partitioning (table too small to benefit)

### Key Components

**Config Block:**
- type: "table" - Full load mode
- bigquery: { clusterBy: ["country", "customer_type"] }
- assertions: { uniqueKey, nonNull, rowConditions }

---

## File: definitions/transformations/cdc_orders.sqlx

**Type:** table (Delta Load pattern with partitioning + clustering)
**Output Table:** cdc_layer.orders
**Purpose:** Transforms raw orders with CDC event handling and optimization

### What It Does

1. Detects duplicates in source (pre_operations) - logs to duplicate_log
2. Reads raw_layer.orders
3. Deduplicates using ROW_NUMBER (keeps latest per order_id)
4. Filters out deleted records (changetype = 'D')
5. Writes to cdc_layer.orders (with partitioning + clustering)
6. Runs assertions
7. Logs to pipeline_audit (post_operations)

### Optimization

- Partitioned by order_date (for time-based queries)
- Clustered by customer_id, status (for filtered queries)
- Industry-standard for large transaction tables

### Key Components

**Config Block:**
- type: "table"
- bigquery: { partitionBy: "order_date", clusterBy: ["customer_id", "status"] }
- assertions: uniqueKey, nonNull, rowConditions

---

## File: definitions/transformations/cdc_orders_backfill.sqlx

**Type:** table (Backfill Staging)
**Output Table:** cdc_layer.orders_backfill_staging
**Purpose:** Reprocesses historical data for specific date range

### What It Does

1. Logs backfill start to backfill_log (pre_operations)
2. Reads raw_layer.orders filtered by date range from variables
3. Applies same deduplication logic as cdc_orders
4. Writes to staging table

### Key Features

- Dataform variables for dynamic dates
- Different table name (orders_backfill_staging) to avoid conflict
- Tags: backfill, on_demand

---

## File: definitions/transformations/cdc_orders_backfill_merge.sqlx

**Type:** operations (DML only, no table created)
**Output:** None (modifies cdc_layer.orders)
**Purpose:** Merges backfill staging data into main orders table

### What It Does

1. MERGEs staging table into cdc_layer.orders
   - WHEN MATCHED -> UPDATE existing rows
   - WHEN NOT MATCHED -> INSERT new rows
2. Updates backfill_log to status = 'COMPLETED'

### Key Components

- type: "operations" - Critical! No table created
- MERGE statement for industry-standard upsert
- Tags: backfill, on_demand

---

## File: definitions/transformations/schema_drift_check.sqlx

**Type:** operations
**Output:** None (inserts into schema_drift_log)
**Purpose:** Detects schema mismatches between expected and actual

### What It Does

1. Reads expected columns from audit_layer.schema_registry
2. Reads actual columns from cdc_layer.INFORMATION_SCHEMA.COLUMNS
3. FULL OUTER JOIN to find differences
4. Categorizes: MISSING_COLUMN, EXTRA_COLUMN, TYPE_MISMATCH
5. Assigns severity: CRITICAL or WARNING
6. Inserts findings into schema_drift_log

### Key Components

- type: "operations"
- Uses BigQuery built-in INFORMATION_SCHEMA
- FULL OUTER JOIN for difference detection

---

## File: definitions/transformations/dim_customers_scd2_merge.sqlx

**Type:** operations
**Output Table:** cdc_layer.dim_customers_scd2 (modifies existing)
**Purpose:** Maintains SCD Type 2 dimension table with full history of customer changes

### What It Does

Step 1 (MERGE):
1. Detects unchanged customers (skips them)
2. Inserts brand new customers (NOT MATCHED)
3. Closes existing records for customers with data changes (MATCHED)

Step 2 (INSERT):
1. Adds new current versions for just-closed customers
2. Includes idempotency check (won't double-insert)

Step 3 (LOG):
1. Logs execution to pipeline_audit

### Key Components

**Config:**
- type: "operations" (no table created, just runs DML)
- tags: ["scd2", "daily", "dimension"]

**Change Detection:**
- Compares all customer columns: name, city, country, type, credit_limit
- Only acts on actual differences

**Idempotency:**
- AND raw.customer_id NOT IN (current records check)
- Safe to run multiple times

### Design Notes

- Two-step pattern required because BigQuery MERGE doesn't allow INSERT on MATCHED
- Idempotency check prevents production bugs on retries
- Industry-standard SCD Type 2 implementation
- Production-grade with audit logging
- Pattern: Close old version, INSERT new current version

### Time-Travel Queries Enabled

Once SCD2 is populated, you can query historical state:

WHERE as_of_date BETWEEN valid_from AND valid_to

This returns the version that was active on any specific date.

---

## Pipeline Execution Order

### Daily Schedule

1. cdc_customers (Full Load with clustering)
2. cdc_orders (Delta Load with partitioning + clustering)
3. dim_customers_scd2_merge (SCD Type 2 history tracking)
4. schema_drift_check
5. Assertions run automatically per table

### Backfill Schedule (On-Demand)

1. Update vars in workflow_settings.yaml
2. Run cdc_orders_backfill (creates staging)
3. Run cdc_orders_backfill_merge (merges to main)

### Manual Triggers Needed

Currently, Dataform requires manual execution. Future Composer DAG will automate this.

---

## Tags Reference

Tags help organize and selectively execute pipelines:

- full_load: Full reload pipelines
- delta_load: CDC delta pipelines
- daily: Routine daily pipelines
- backfill: On-demand recovery pipelines
- on_demand: Manual trigger only
- data_quality: Data validation operations
- schema_drift: Schema monitoring operations
- scd2: SCD Type 2 pipelines
- dimension: Dimension table maintenance

To run all daily pipelines: Execute by tag "daily"

---

## Common Patterns Across Files

### Pattern: Schema-Qualified ref()

All ref() calls use schema-qualified syntax:
- ${ref({schema: "raw_layer", name: "customers"})}

This avoids ambiguity when tables share names across schemas.

### Pattern: pre_operations

Used for:
- Logging operation start
- Detecting duplicates before processing

### Pattern: post_operations

Used for:
- Logging operation completion
- Updating audit_logs with row counts

### Pattern: Audit Logging

All transformations log to pipeline_audit with:
- GENERATE_UUID() for unique ID
- pipeline_name (identifier)
- load_type (FULL_LOAD / DELTA_LOAD / SCD_TYPE_2)
- start_time, end_time
- rows_processed (SELECT COUNT FROM ${self()})
- status (always SUCCESS in post_operations)
- SESSION_USER() for created_by

### Pattern: BigQuery Optimization

Tables include bigquery config for performance:
- partitionBy: Time-based column for large tables
- clusterBy: Up to 4 frequently-filtered columns

### Pattern: SCD Type 2 (Two-Step MERGE)

For dimension tables that need history:
1. MERGE handles new inserts + closes for changed
2. INSERT adds new current versions
3. Idempotency check prevents duplicates

---

## Document Metadata

- **Author:** Jayachandra Reddy (Jay)
- **Created:** May 26, 2026
- **Last Updated:** May 27, 2026
- **Total Files Documented:** 9
- **Status:** Living document

- 
