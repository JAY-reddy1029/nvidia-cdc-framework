# Pipeline Documentation

Detailed documentation of every pipeline file in the CDC Framework.

---

## Overview

The framework contains 8 SQLX files organized into two folders:

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

**Code Structure:**
- config block specifies type, database, schema, name
- No SELECT statement
- No transformations

**Why we need it:** Without declaration files, we cannot use ${ref()} to reference source tables in transformation files. Declarations tell Dataform "this table exists outside Dataform's control."

---

## File: definitions/sources/raw_orders.sqlx

**Type:** declaration
**Output Table:** None
**Purpose:** Declares that raw_layer.orders exists

**Code Structure:** Same pattern as raw_customers.sqlx, but for orders table.

**Why we need it:** To reference raw_layer.orders in cdc_orders.sqlx using ${ref({schema: "raw_layer", name: "orders"})}.

---

## File: definitions/transformations/cdc_customers.sqlx

**Type:** table (Full Load)
**Output Table:** cdc_layer.customers
**Purpose:** Transforms raw customers data to clean CDC layer

### What It Does

1. Reads all rows from raw_layer.customers
2. Adds cdc_processed_at timestamp column
3. Writes to cdc_layer.customers (drops and recreates)
4. Runs assertions (uniqueKey, nonNull, rowConditions)
5. Logs execution to pipeline_audit (post_operations)

### Key Components

**Config Block:**
- type: "table" - Full load mode
- schema: "cdc_layer"
- name: "customers"
- tags: ["full_load", "daily"]
- assertions: { uniqueKey, nonNull, rowConditions }

**Assertions:**
- uniqueKey: ["customer_id"] - No duplicate customer IDs
- nonNull: ["customer_id", "customer_name", "country"] - Required fields
- rowConditions: 
  - "credit_limit >= 0" - Non-negative credit limit
  - "customer_type IN ('Retail', 'Wholesale')" - Valid types

**Main Query:**
- SELECT all columns from raw_layer.customers
- Add CURRENT_TIMESTAMP() AS cdc_processed_at

**Post Operations:**
- INSERT into audit_layer.pipeline_audit with run details

### Design Notes

- Full Load chosen because customer master data is small and slow-changing
- Drops table each run - simple but inefficient for large data
- Production scale would use incremental type instead

---

## File: definitions/transformations/cdc_orders.sqlx

**Type:** table (Delta Load pattern)
**Output Table:** cdc_layer.orders
**Purpose:** Transforms raw orders with CDC event handling

### What It Does

1. Detects duplicates in source (pre_operations) - logs to duplicate_log
2. Reads raw_layer.orders
3. Deduplicates using ROW_NUMBER (keeps latest per order_id)
4. Filters out deleted records (changetype = 'D')
5. Writes 13 clean rows to cdc_layer.orders
6. Runs assertions
7. Logs to pipeline_audit (post_operations)

### Key Components

**Config Block:**
- type: "table"
- schema: "cdc_layer"
- name: "orders"
- tags: ["delta_load", "daily"]
- assertions: { uniqueKey, nonNull, rowConditions }

**Assertions:**
- uniqueKey: ["order_id"]
- nonNull: ["order_id", "customer_id", "product_name", "last_updated"]
- rowConditions:
  - "quantity > 0"
  - "amount > 0"
  - "changetype IN ('I', 'U', 'D')"
  - "status IN ('Confirmed', 'Cancelled', 'Pending')"

**Pre Operations (Duplicate Detection):**
- Groups raw_layer.orders by order_id
- Counts duplicates and unique timestamps
- Categorizes: EXACT, LOGICAL, CONFLICTING
- Inserts into duplicate_log

**Main Query Pattern:**
- WITH latest_changes AS (...)
  - ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY last_updated DESC)
  - Filter WHERE rn = 1
- SELECT from latest_changes
- WHERE changetype != 'D'
- Adds cdc_processed_at

**Post Operations:**
- INSERT into audit_layer.pipeline_audit

### Design Notes

- Most complex pipeline due to CDC handling
- ROW_NUMBER pattern is industry-standard for CDC deduplication
- pre_operations + post_operations show full audit lifecycle
- Handles all CDC event types (I/U/D) correctly

---

## File: definitions/transformations/cdc_orders_backfill.sqlx

**Type:** table (Backfill Staging)
**Output Table:** cdc_layer.orders_backfill_staging
**Purpose:** Reprocesses historical data for specific date range

### What It Does

1. Logs backfill start to backfill_log (pre_operations)
2. Reads raw_layer.orders filtered by date range from variables
3. Applies same deduplication logic as cdc_orders
4. Writes to staging table (NOT main orders table)

### Key Components

**Config Block:**
- type: "table"
- name: "orders_backfill_staging" (different from main!)
- tags: ["backfill", "on_demand"]
- NOT tagged "daily" - won't run with regular schedule

**Variables Used:**
- ${dataform.projectConfig.vars.backfill_start_date}
- ${dataform.projectConfig.vars.backfill_end_date}
- ${dataform.projectConfig.vars.backfill_reason}
- ${dataform.projectConfig.vars.backfill_requested_by}

**Pre Operations:**
- INSERT into backfill_log with status = 'IN_PROGRESS'

**Main Query:**
- Same deduplication as cdc_orders
- PLUS: WHERE order_date BETWEEN start and end dates

### Design Notes

- Different output table avoids conflict with main cdc_orders.sqlx
- Variables make date range dynamic
- Tags separate it from daily pipeline
- Staging pattern provides safety before MERGE

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

**Config Block:**
- type: "operations" - Critical! No table created
- tags: ["backfill", "on_demand"]

**MERGE Statement:**
- Target: cdc_layer.orders
- Source: cdc_layer.orders_backfill_staging
- ON condition: T.order_id = S.order_id
- All 10 columns updated/inserted

**Post-MERGE Update:**
- UPDATE backfill_log SET status = 'COMPLETED' WHERE in-progress

### Design Notes

- Uses "operations" type because no new table needed
- Two-file split (staging + merge) for safety
- If MERGE fails, staging data is preserved
- Industry-standard upsert pattern

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

**Config Block:**
- type: "operations"
- tags: ["data_quality", "schema_drift"]

**Two CTEs:**
- expected_schemas: from schema_registry table
- actual_schemas: from INFORMATION_SCHEMA.COLUMNS

**FULL OUTER JOIN Logic:**
- Joins on dataset_name, table_name, column_name
- Returns: missing, extra, or type-mismatched columns

**CASE Statements:**
- Drift type: based on which side is NULL
- Severity: CRITICAL for missing/type changes, WARNING for extras

**INSERT Target:**
- audit_layer.schema_drift_log

### Design Notes

- Uses BigQuery built-in metadata (no external dependencies)
- FULL OUTER JOIN catches both missing and extra columns
- Runs as separate operation (doesn't block main pipelines)
- Foundation for proactive schema governance

---

## Pipeline Execution Order

### Daily Schedule

1. cdc_customers (Full Load)
2. cdc_orders (Delta Load)
3. schema_drift_check
4. Assertions run automatically per table

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
- load_type (FULL_LOAD / DELTA_LOAD)
- start_time, end_time
- rows_processed (SELECT COUNT FROM ${self()})
- status (always SUCCESS in post_operations)
- SESSION_USER() for created_by

---

## Document Metadata

- **Author:** Jayachandra Reddy (Jay)
- **Created:** May 26, 2026
- **Total Files Documented:** 8
- **Status:** Living document
