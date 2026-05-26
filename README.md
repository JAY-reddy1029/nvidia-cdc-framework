# Production-Grade CDC Framework on Google Cloud Platform

A complete Change Data Capture (CDC) data engineering framework built on BigQuery and Dataform, implementing 8 enterprise features for data quality, monitoring, and reliability.

This project mirrors production patterns used at companies like NVIDIA for processing SAP source data into clean analytics layers.

---

## Project Overview

This framework processes data from SAP-like source tables through a clean, audited transformation layer. It includes comprehensive monitoring, automated quality checks, and recovery capabilities — everything needed for production data engineering.

**Tech Stack:**
- **Google Cloud Platform** (GCP)
- **BigQuery** — Data warehouse
- **Dataform** — SQL-based transformation tool
- **SQLX** — Dataform's enhanced SQL syntax
- **Python** — For orchestration (future)

**Region:** asia-south1 (Mumbai)

---

## Architecture

The framework follows a medallion architecture pattern with separate operational monitoring:

```
                    ┌─────────────────────────┐
                    │   SAP / Source System   │
                    │      (Simulated)        │
                    └───────────┬─────────────┘
                                │
                                ▼
                  ┌──────────────────────────┐
                  │      raw_layer           │
                  │  (Bronze - Source Data)  │
                  ├──────────────────────────┤
                  │  • customers (10 rows)   │
                  │  • orders (27 rows)      │
                  └───────────┬──────────────┘
                              │
                              │ Dataform Pipelines
                              ▼
                  ┌──────────────────────────┐       ┌──────────────────────┐
                  │      cdc_layer           │       │   audit_layer        │
                  │  (Silver - Clean Data)   │──────▶│  (Operational Data)  │
                  ├──────────────────────────┤ logs  ├──────────────────────┤
                  │  • customers (10)        │       │  • pipeline_audit    │
                  │  • orders (13)           │       │  • error_log         │
                  │  • orders_backfill_stg   │       │  • duplicate_log     │
                  └───────────┬──────────────┘       │  • backfill_log      │
                              │                      │  • schema_registry   │
                              │ Assertions           │  • schema_drift_log  │
                              ▼                      │  • pipeline_checkpoint│
                  ┌──────────────────────────┐       └──────────────────────┘
                  │   dataform_assertions    │
                  │  (Quality Failures Only) │
                  └──────────────────────────┘
```

### Data Flow

1. **Source data** lands in raw_layer (untouched, read-only)
2. **Dataform pipelines** transform data into cdc_layer (clean, deduplicated)
3. **Audit events** logged to audit_layer (7 specialized tables)
4. **Failed assertions** isolated in dataform_assertions

See [ARCHITECTURE.md](./ARCHITECTURE.md) for detailed technical design.
---

## Features

This framework implements **8 production-grade features** typical of enterprise CDC frameworks:

| # | Feature | Purpose |
|---|---------|---------|
| 1 | **Load Types (Full + Delta)** | Full reload for master data, delta processing for transaction data |
| 2 | **Audit Table** | Logs every pipeline execution with timing and row counts |
| 3 | **Error Logging** | Categorized error tracking with severity levels |
| 4 | **Duplicate Handling** | Detects and categorizes duplicates (EXACT, LOGICAL, CONFLICTING) |
| 5 | **Backfill** | Safe historical reprocessing using staging + MERGE pattern |
| 6 | **Schema Drift Detection** | Auto-detects unexpected schema changes |
| 7 | **Data Quality Assertions** | Built-in validation with uniqueKey, nonNull, rowConditions |
| 8 | **Resume from Failure** | Checkpoint-based recovery for partial pipeline failures |

---

## Data Layers

### raw_layer (Source)
Mimics SAP source tables. Read-only for pipelines.
- `customers` — Customer master data (10 rows)
- `orders` — Sales orders with CDC events I/U/D (27 rows)

### cdc_layer (Transformed)
Clean, business-ready data. Output of pipelines.
- `customers` — Clean customer records
- `orders` — Deduplicated orders with deletes filtered

### audit_layer (Operational)
Operational monitoring across 7 tables.
- `pipeline_audit` — Tracks every pipeline run
- `error_log` — Categorized errors with severity
- `duplicate_log` — Detected duplicates and actions
- `backfill_log` — Backfill executions with reason
- `schema_registry` — Expected schemas (source of truth)
- `schema_drift_log` — Detected schema drifts
- `pipeline_checkpoint` — Per-table execution status

### dataform_assertions (Quality)
Auto-populated when data quality assertions fail.

---

## Quick Start

### Prerequisites
- GCP account with BigQuery and Dataform enabled
- Region: asia-south1 (or your preferred region)

### Setup Order

1. **Create datasets** in BigQuery: `raw_layer`, `cdc_layer`, `audit_layer`
2. **Run DDL scripts** to create source and audit tables (see `src/ddl/`)
3. **Load sample data** into raw_layer tables
4. **Set up Dataform repository** with the SQLX files (see `src/dataform/`)
5. **Execute pipelines** in order: customers, orders, schema_drift_check

See [PIPELINES.md](./PIPELINES.md) for detailed pipeline documentation.

---

## Project Structure

nvidia-cdc-framework/ ├── README.md # This file ├── ARCHITECTURE.md # Technical design ├── PIPELINES.md # Pipeline documentation
├── RUNBOOK.md # Operations guide ├── LICENSE # MIT License │ ├── docs/ │ └── architecture-diagram.png │ └── src/ ├── ddl/ # CREATE TABLE statements ├── dataform/ # SQLX pipeline files └── queries/ # Analytics queries


---

## Key Concepts Demonstrated

- **CDC Patterns** — Insert/Update/Delete event handling
- **Deduplication** — `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` 
- **Upserts** — MERGE statements for safe data merging
- **Schema-Qualified References** — `${ref({schema, name})}` syntax
- **Data Lineage** — Tracked through dependencies
- **Metadata Queries** — Using INFORMATION_SCHEMA
- **Audit Trails** — Pre/post operation logging
- **Recovery Patterns** — Backfill staging and checkpoint-based resume

---

## Operations

For operational guidance — what to do when things break, how to backfill, how to handle schema drift — see [RUNBOOK.md](./RUNBOOK.md).

---

## Tech Highlights

- **Dataform Types Used:** declaration, table, operations
- **SQL Patterns:** CTEs, window functions, FULL OUTER JOIN, MERGE
- **Audit Coverage:** 7 dedicated audit tables
- **Quality Coverage:** uniqueKey, nonNull, rowConditions assertions

---

## Documentation

| Document | Purpose |
|----------|---------|
| [README.md](./README.md) | Project overview (you are here) |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Technical design and data flow |
| [PIPELINES.md](./PIPELINES.md) | Each pipeline explained in detail |
| [RUNBOOK.md](./RUNBOOK.md) | Operations and troubleshooting guide |

---

## License

This project is licensed under the MIT License — see the [LICENSE](./LICENSE) file for details.

---

## Acknowledgments

Built as a hands-on learning project to prepare for production data engineering at NVIDIA. Inspired by patterns from Google Cloud, Netflix, and Snowflake architectures.

---

## Contact

**Jayachandra Reddy (Jay)**  
Email: p.v.jay2003@gmail.com  
Location: Hyderabad, India

