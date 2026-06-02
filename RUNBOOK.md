# Operations Runbook

Step-by-step operational guide for the CDC Framework. Use this when something breaks or needs intervention.

---

## Quick Reference

| Scenario | Section |
|----------|---------|
| Pipeline failed | Section 1: Pipeline Failure |
| Need to backfill data | Section 2: Backfill Procedure |
| Schema drift detected | Section 3: Schema Drift Response |
| Data quality assertion failed | Section 4: Data Quality Failures |
| Daily health check | Section 5: Daily Operations |
| Need to add new table | Section 6: Adding New Tables |
| Troubleshooting common errors | Section 7: Common Errors |

---

## Section 1: Pipeline Failure

### Symptoms
- Email alert: pipeline failed
- Dataform Executions tab shows red FAILED
- Downstream analytics show stale data

### Step 1: Identify Which Pipeline Failed

Query the audit table:
SELECT pipeline_name, table_name, status, error_message, start_time
FROM project-a2f79f6e-db67-4726-be8.audit_layer.pipeline_audit
WHERE status = 'FAILED'
ORDER BY start_time DESC
LIMIT 10;


### Step 2: Check Error Log
SELECT error_category, severity, error_message, occurred_at
FROM project-a2f79f6e-db67-4726-be8.audit_layer.error_log
WHERE resolved = FALSE
ORDER BY occurred_at DESC;


### Step 3: Investigate Root Cause

Common causes by category:
- SYNTAX: Code bug in SQLX file - check recent commits
- DATA_QUALITY: Bad data from source - check raw_layer
- SCHEMA_DRIFT: Source schema changed - check schema_drift_log
- PERMISSION: IAM issue - check user roles

### Step 4: Fix and Re-run

1. Fix the issue (code, data, or permissions)
2. Go to Dataform - Workspace
3. Click "Start execution"
4. Select the failed action
5. Click "Start execution" button
6. Verify in Executions tab

### Step 5: Mark Error as Resolved
UPDATE project-a2f79f6e-db67-4726-be8.audit_layer.error_log
SET resolved = TRUE, resolved_at = CURRENT_TIMESTAMP()
WHERE error_id = '<the_error_id>';


---

## Section 2: Backfill Procedure

### When to Backfill

- Source data was incorrect for a date range
- Pipeline logic changed and historical data needs reprocessing
- Missed processing days due to outage

### Step 1: Update Variables

Edit workflow_settings.yaml in Dataform workspace:
vars: backfill_start_date: "2026-05-05" backfill_end_date: "2026-05-07" backfill_reason: "Pipeline outage recovery" backfill_requested_by: "your.email@gmail.com"


### Step 2: Run Staging Pipeline

1. Go to Dataform - jay-dev workspace
2. Click "Start execution"
3. Tick: cdc_orders_backfill_staging
4. Click "Start execution"
5. Wait for completion (~10 seconds)

### Step 3: Verify Staging Data
SELECT COUNT(*) AS staging_rows,
MIN(order_date) AS min_date,
MAX(order_date) AS max_date
FROM project-a2f79f6e-db67-4726-be8.cdc_layer.orders_backfill_staging;


Verify count and date range match expectations.

### Step 4: Run Merge Pipeline

1. Click "Start execution" again
2. Tick: cdc_orders_backfill_merge
3. Click "Start execution"
4. Wait for completion

### Step 5: Verify Backfill
SELECT * FROM project-a2f79f6e-db67-4726-be8.audit_layer.backfill_log
ORDER BY execution_start DESC LIMIT 5;


Confirm status = 'COMPLETED' and rows_affected is correct.

### Step 6: Verify Main Table
SELECT order_date, COUNT(*) AS orders
FROM project-a2f79f6e-db67-4726-be8.cdc_layer.orders
WHERE order_date BETWEEN '<start>' AND '<end>'
GROUP BY order_date;


---

## Section 3: Schema Drift Response

### Symptoms
- schema_drift_log has new CRITICAL entries
- Downstream reports show missing columns
- Pipeline succeeds but data is wrong

### Step 1: Check Drift Details
SELECT dataset_name, table_name, drift_type, column_name,
expected_value, actual_value, severity, detected_at
FROM project-a2f79f6e-db67-4726-be8.audit_layer.schema_drift_log
WHERE resolved = FALSE
ORDER BY severity, detected_at DESC;


### Step 2: Determine Drift Type and Action

**MISSING_COLUMN (CRITICAL):**
- A column we expect is missing from the actual table
- Cause: Someone dropped it OR pipeline didn't create it
- Action: Investigate why, restore the column, re-run pipeline

**EXTRA_COLUMN (WARNING):**
- A column exists that we don't expect
- Cause: Source schema changed OR manual ALTER TABLE
- Actions: 
  - If column should be added: Update schema_registry, then ignore
  - If column is unwanted: ALTER TABLE DROP COLUMN

**TYPE_MISMATCH (CRITICAL):**
- Column exists but with wrong data type
- Cause: Schema definition changed
- Action: Investigate and fix the type, or update registry

### Step 3: Update Schema Registry (if needed)

If the drift is intentional (e.g., new column added):
INSERT INTO project-a2f79f6e-db67-4726-be8.audit_layer.schema_registry (registry_id, dataset_name, table_name, column_name, expected_data_type, is_required, column_order, registered_by) VALUES ('REG_XXX', 'cdc_layer', 'table_name', 'new_column', 'STRING', TRUE, 99, 'your.email@gmail.com');


### Step 4: Mark Drift as Resolved
UPDATE project-a2f79f6e-db67-4726-be8.audit_layer.schema_drift_log SET resolved = TRUE, resolved_at = CURRENT_TIMESTAMP(), resolved_by = 'your.email@gmail.com' WHERE drift_id = '<the_drift_id>';


### Step 5: Re-run Schema Drift Check

In Dataform, execute schema_drift_check to verify no new drifts.

---

## Section 4: Data Quality Failures

### Symptoms
- Dataform shows ASSERTION FAILED in Executions
- New tables appear in dataform_assertions dataset
- Pipeline status is FAILED

### Step 1: Identify Failed Assertions

In BigQuery, check dataform_assertions dataset:
- cdc_layer_customers_assertions_uniqueKey_0
- cdc_layer_orders_assertions_rowConditions
- (etc.)

Tables with rows mean assertions failed.

### Step 2: Query the Failed Rows
SELECT * FROM project-a2f79f6e-db67-4726-be8.dataform_assertions.cdc_layer_orders_assertions_rowConditions;


### Step 3: Determine the Issue

Common assertion failures:
- quantity <= 0: Bad source data
- amount <= 0: Bad source data
- Invalid changetype: Source system bug
- Invalid status: New status not in allowed list
- NULL in required field: Source system issue

### Step 4: Fix Source Data OR Update Rules

**If source data is wrong:**
- Contact source system team
- Or fix manually:
UPDATE project-a2f79f6e-db67-4726-be8.raw_layer.orders
SET status = 'Confirmed'
WHERE order_id = 'ORD_XYZ' AND status = 'Returned';


**If business rules changed:**
- Update assertion in SQLX file
- Example: Add 'Returned' to valid statuses
- Commit and re-run pipeline

### Step 5: Re-run Pipeline

After fix, re-execute the pipeline and verify assertions pass.

---

## Section 5: Daily Operations

### Morning Health Check (5 minutes)

Run these queries every morning:

**Check overnight pipeline runs:**
SELECT pipeline_name, status, rows_processed, duration_seconds, start_time
FROM project-a2f79f6e-db67-4726-be8.audit_layer.pipeline_audit
WHERE DATE(start_time) = CURRENT_DATE()
ORDER BY start_time DESC;


**Check for new errors:**
SELECT COUNT(*) AS new_errors_today
FROM project-a2f79f6e-db67-4726-be8.audit_layer.error_log
WHERE DATE(occurred_at) = CURRENT_DATE()
AND resolved = FALSE;


**Check for schema drift:**
SELECT COUNT(*) AS unresolved_drifts
FROM project-a2f79f6e-db67-4726-be8.audit_layer.schema_drift_log
WHERE resolved = FALSE;


**Check assertion failures:**
- Browse dataform_assertions dataset
- Look for tables with rows

### Weekly Cleanup

**Resolve old WARNING entries:**
UPDATE project-a2f79f6e-db67-4726-be8.audit_layer.error_log
SET resolved = TRUE, resolved_at = CURRENT_TIMESTAMP()
WHERE severity = 'WARNING'
AND DATE_DIFF(CURRENT_DATE(), DATE(occurred_at), DAY) > 7;


---

## Section 6: Adding New Tables

### Step 1: Add Source Table to raw_layer
CREATE TABLE project-a2f79f6e-db67-4726-be8.raw_layer.new_table (
-- columns
);


### Step 2: Create Declaration File

Create definitions/sources/raw_new_table.sqlx:
config {
type: "declaration",
database: "project-a2f79f6e-db67-4726-be8",
schema: "raw_layer",
name: "new_table"
}


### Step 3: Create Transformation File

Create definitions/transformations/cdc_new_table.sqlx with:
- config block with type, schema, name, tags, assertions
- pre_operations (optional)
- Main SELECT query with ${ref()}
- post_operations for audit logging

### Step 4: Register Expected Schema
INSERT INTO project-a2f79f6e-db67-4726-be8.audit_layer.schema_registry
(registry_id, dataset_name, table_name, column_name, expected_data_type,
is_required, column_order, registered_by)
VALUES
('REG_XXX', 'cdc_layer', 'new_table', 'column1', 'STRING', TRUE, 1, 'your.email');
-- repeat for each column


### Step 5: Test in Dataform Workspace

1. Verify SQLX compiles green
2. Execute the new action
3. Check audit_layer.pipeline_audit for log entry
4. Verify assertions pass

### Step 6: Update Documentation

- Add new table to PIPELINES.md
- Update ARCHITECTURE.md if architecture changed
- Commit changes

---

## Section 7: Common Errors and Solutions

### Error: "Ambiguous Action name"

**Cause:** Table name exists in multiple schemas

**Fix:** Use schema-qualified ref()
- Wrong: ${ref("customers")}
- Right: ${ref({schema: "raw_layer", name: "customers"})}

### Error: "Duplicate canonical target detected"

**Cause:** Two SQLX files target same table

**Fix:** Each SQLX must write to a unique table. Rename the output table in config.

### Error: "Required field cannot be null"

**Cause:** BigQuery NOT NULL constraint at schema level rejecting bad insert

**Fix:** Either fix the data OR remove NOT NULL constraint if intentional

### Error: "Table not found in location"

**Cause:** Referencing a table that doesn't exist yet OR in wrong region

**Fix:** 
- Check table actually exists
- Check region matches (asia-south1)
- Run pre-requisite pipelines first

### Error: "Unexpected property 'schema'"

**Cause:** Wrong syntax for Dataform dependencies

**Fix:** Use tags instead of dependencies, or use correct property names per Dataform version

### Error: Assertion failed

**Cause:** Bad data violates business rules

**Fix:** See Section 4: Data Quality Failures

### Error: BigQuery timeout

**Cause:** Query too complex or table too large

**Fix:** 
- Add filters to limit data scanned
- Add partitioning to large tables
- Cluster by frequently filtered columns

---

## Emergency Contacts

### Internal Team
- Data Engineering Lead: TBD
- On-call rotation: TBD

### External
- GCP Support: cloud.google.com/support
- Dataform GitHub: github.com/dataform-co/dataform

---

## Useful Queries Reference

### View Recent Pipeline History
SELECT pipeline_name, status, rows_processed, duration_seconds, start_time
FROM project-a2f79f6e-db67-4726-be8.audit_layer.pipeline_audit
WHERE start_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY start_time DESC;


### View All Open Issues
SELECT 'Error' AS type, error_id AS id, severity, error_message AS details, occurred_at
FROM project-a2f79f6e-db67-4726-be8.audit_layer.error_log
WHERE resolved = FALSE
UNION ALL
SELECT 'Drift', drift_id, severity, CONCAT(drift_type, ': ', column_name), detected_at
FROM project-a2f79f6e-db67-4726-be8.audit_layer.schema_drift_log
WHERE resolved = FALSE
ORDER BY occurred_at DESC;


### Check Resume from Failure Status
WITH failed_run AS (
SELECT run_id FROM project-a2f79f6e-db67-4726-be8.audit_layer.pipeline_checkpoint
WHERE status = 'FAILED'
ORDER BY start_time DESC LIMIT 1
)
SELECT cp.table_name, cp.status,
CASE
WHEN cp.status = 'COMPLETED' THEN 'SKIP'
WHEN cp.status = 'FAILED' THEN 'RETRY'
WHEN cp.status = 'PENDING' THEN 'RUN'
END AS action_needed
FROM project-a2f79f6e-db67-4726-be8.audit_layer.pipeline_checkpoint cp
INNER JOIN failed_run f ON cp.run_id = f.run_id
ORDER BY cp.table_order;


---

## Runbook Maintenance

- Review and update this runbook monthly
- After every incident, add lessons learned
- Keep contact information current

---

## Document Metadata

- **Author:** Jayachandra Reddy (Jay)
- **Created:** May 26, 2026
- **Last Updated:** May 26, 2026
- **Status:** Living document - update as project evolves
