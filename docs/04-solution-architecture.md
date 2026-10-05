# 04 — Solution Architecture

## Purpose

This document defines the technical architecture for the Multi-Source MIS & Management Reporting Automation project.

The architecture is designed to support:

- multi-source data ingestion
- incremental loading
- centralized analytical storage
- governed transformations
- management KPI calculation
- data-quality validation
- scheduled execution
- operational monitoring
- low ongoing cost
- simple maintenance

The design intentionally avoids unnecessary complexity such as streaming platforms or distributed processing frameworks because the reporting requirements do not require them.

---

# 1. Architecture Goals

The solution should satisfy the following technical goals.

## Reliability

Pipelines should be restartable and should avoid duplicate data when the same ingestion window is processed more than once.

## Traceability

Every reporting record should be traceable back to:

```text
dashboard
→ data mart
→ transformation
→ staging
→ source system
```

## Incremental Processing

Large transactional datasets should not be reloaded in full on every pipeline execution.

## Data Quality

Important business and technical validations should run automatically.

## Maintainability

The solution should be understandable and supportable by a small data or analytics team.

## Cost Efficiency

The architecture should minimize:

- unnecessary infrastructure
- unnecessary BigQuery scans
- always-running compute
- duplicate storage
- excessive pipeline frequency

---

# 2. High-Level Architecture

```text
                       SOURCE SYSTEMS
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
   PostgreSQL ERP       Excel / CSV       Logistics API
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
                   Python Ingestion Layer
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        Extraction      Validation      Audit Logging
              │              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                      BigQuery Platform
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
       Staging          Transformation       Control
       Layer               Layer              Layer
          │                  │
          └──────────────┬───┘
                         │
                         ▼
                  Reporting Data Mart
                         │
                         ▼
                    Looker Studio
                         │
                         ▼
                      Management
```

Monitoring and data-quality controls operate across the ingestion and transformation workflow.

---

# 3. Architectural Style

The platform follows a lightweight batch ELT architecture.

The main pattern is:

```text
Extract
   ↓
Load
   ↓
Transform
```

Data is first loaded into BigQuery staging tables with minimal transformation.

Business logic is applied afterwards inside BigQuery.

This approach provides:

- easier debugging
- better traceability
- reproducible transformations
- separation between source ingestion and business rules
- simpler reprocessing

---

# 4. Source Layer

The platform integrates three source patterns.

## PostgreSQL ERP

Used for:

- master data
- orders
- order lines
- invoices
- payments
- inventory

Primary ingestion method:

```text
Python + PostgreSQL driver
```

Load patterns:

```text
Master tables
→ Full or incremental

Transactional tables
→ Incremental

Inventory
→ Daily snapshot
```

---

## Excel / CSV

Used for:

- sales targets
- branch budgets
- operating expenses
- product margin adjustments

Primary ingestion method:

```text
Python + pandas/openpyxl
```

Files should be validated before loading.

Recommended validation includes:

- expected filename or source type
- required columns
- column data types
- duplicate business keys
- null mandatory fields
- reference-value validation

---

## Logistics REST API

Used for shipment and delivery information.

Primary ingestion method:

```text
Python HTTP client
```

The API pipeline must support:

- authentication
- pagination
- timeouts
- retries
- incremental extraction
- response validation
- duplicate prevention
- HTTP error logging

---

# 5. Python Ingestion Layer

Python acts as the common ingestion framework for all source types.

This keeps the implementation simple while still allowing source-specific behavior.

Suggested structure:

```text
ingestion/
│
├── common/
│   ├── config.py
│   ├── logging.py
│   ├── bigquery.py
│   ├── pipeline_control.py
│   └── exceptions.py
│
├── postgres/
│   ├── extract.py
│   └── pipelines.py
│
├── files/
│   ├── excel.py
│   ├── csv.py
│   └── validators.py
│
└── api/
    ├── client.py
    └── shipments.py
```

The objective is to share reusable infrastructure without building an unnecessarily large internal framework.

---

# 6. Pipeline Run Model

Each pipeline execution should have a unique run identifier.

Example:

```text
RUN_20261006_060001_ERP_ORDERS
```

Each run should record:

```text
pipeline_run_id
pipeline_name
source_system
started_at
completed_at
status
watermark_from
watermark_to
rows_extracted
rows_loaded
rows_rejected
error_message
```

Possible statuses:

```text
STARTED
SUCCESS
FAILED
PARTIAL_SUCCESS
```

---

# 7. Incremental Loading Strategy

Transactional PostgreSQL tables should use timestamp-based watermarks.

Example source query:

```sql
SELECT *
FROM sales_orders
WHERE updated_at > :watermark_from
  AND updated_at <= :watermark_to;
```

The pipeline should determine `watermark_to` at the beginning of the run.

This creates a stable extraction window.

Example:

```text
Previous successful watermark:
2026-10-05 06:00:00

Current pipeline starts:
2026-10-06 06:00:05

Extraction window:

updated_at > 2026-10-05 06:00:00
AND
updated_at <= 2026-10-06 06:00:05
```

The new watermark is committed only after the pipeline succeeds.

---

# 8. Why Watermarks Are Important

Without controlled watermarks, pipelines can:

- miss late records
- create overlapping loads
- duplicate records
- lose track of failed extraction windows

The control table therefore becomes part of the pipeline state.

Example:

```text
pipeline_name              postgres_sales_orders
last_successful_watermark  2026-10-06 06:00:05
last_run_status            SUCCESS
rows_loaded                184
```

---

# 9. Idempotency

Pipelines should be safe to rerun.

If the same extraction window is processed twice, the target should not contain duplicate business records.

For mutable transactional tables, the preferred pattern is:

```text
Extract changed rows
      ↓
Load temporary/staging batch
      ↓
MERGE into persistent staging table
```

Example BigQuery pattern:

```sql
MERGE staging.sales_orders AS target
USING load_batch.sales_orders AS source
ON target.order_id = source.order_id

WHEN MATCHED THEN
  UPDATE SET
    order_status = source.order_status,
    total_amount = source.total_amount,
    updated_at = source.updated_at,
    _ingestion_timestamp = source._ingestion_timestamp

WHEN NOT MATCHED THEN
  INSERT ROW;
```

This allows updates to existing source transactions without duplicating them.

---

# 10. BigQuery Dataset Architecture

The initial design will use separate logical datasets.

```text
BigQuery Project
│
├── control
├── staging
├── transform
└── mart
```

A dedicated quality dataset may be introduced later if required.

---

# 11. Control Dataset

Suggested dataset:

```text
mis_control
```

Purpose:

Store operational metadata rather than business reporting data.

Potential tables:

```text
pipeline_runs
pipeline_watermarks
data_quality_results
file_load_history
reconciliation_results
```

---

# 12. Staging Dataset

Suggested dataset:

```text
mis_staging
```

Purpose:

Hold standardized source data with minimal business transformation.

Examples:

```text
stg_branches
stg_warehouses
stg_sales_representatives
stg_customers
stg_products
stg_sales_orders
stg_sales_order_lines
stg_invoices
stg_payments
stg_inventory_snapshot

stg_sales_targets
stg_branch_budget
stg_operating_expenses
stg_margin_adjustments

stg_shipments
```

Staging should retain source-oriented field names where practical.

---

# 13. Standard Staging Metadata

Most staging tables should contain technical metadata such as:

```text
_source_system
_source_table
_pipeline_run_id
_ingestion_timestamp
_source_updated_at
```

File-based data may additionally contain:

```text
_source_file_name
_source_file_modified_at
```

API data may contain:

```text
_api_extracted_at
```

This metadata supports traceability and troubleshooting.

---

# 14. Transformation Dataset

Suggested dataset:

```text
mis_transform
```

Purpose:

Apply:

- standardization
- cleansing
- business logic
- joins
- enrichment
- deduplication
- derived values

Potential models:

```text
tr_customer
tr_product
tr_order
tr_order_line
tr_invoice
tr_payment
tr_inventory
tr_shipments
tr_sales_targets
tr_branch_budget
```

This layer is where source-system differences are resolved.

---

# 15. Data Mart Dataset

Suggested dataset:

```text
mis_mart
```

Purpose:

Provide stable reporting models for Looker Studio and management analysis.

The mart should be modeled around business processes rather than source tables.

Potential dimensions:

```text
dim_date
dim_branch
dim_customer
dim_product
dim_sales_rep
dim_warehouse
```

Potential fact tables:

```text
fact_sales
fact_receivables
fact_inventory_snapshot
fact_sales_target
fact_branch_budget
fact_operating_expense
fact_shipments
```

Exact design will be finalized in the data-model document.

---

# 16. Layer Responsibilities

The separation of responsibilities should remain clear.

## Staging

Question:

> What did the source system provide?

Minimal business transformation.

---

## Transformation

Question:

> How should the source data be interpreted and standardized?

This layer contains cleansing and business rules.

---

## Mart

Question:

> What structure does management reporting need?

This layer contains curated analytical models and KPIs.

---

# 17. Example Data Flow — Revenue

```text
PostgreSQL
invoices
    │
    ▼
Python Incremental Extraction
    │
    ▼
mis_staging.stg_invoices
    │
    ▼
validation
status mapping
cancellation handling
    │
    ▼
mis_transform.tr_invoice
    │
    ▼
fact_sales / reporting model
    │
    ▼
Recognized Revenue
    │
    ▼
Looker Studio
```

---

# 18. Example Data Flow — Sales Target

```text
sales_targets_2026.xlsx
        │
        ▼
File Validation
        │
        ▼
Python File Ingestion
        │
        ▼
mis_staging.stg_sales_targets
        │
        ▼
Employee / Branch Mapping
        │
        ▼
mis_transform.tr_sales_targets
        │
        ▼
mis_mart.fact_sales_target
        │
        ▼
Actual vs Target Dashboard
```

---

# 19. Example Data Flow — Shipments

```text
Logistics API
     │
     ▼
API Client
pagination + retries
     │
     ▼
stg_shipments
     │
     ▼
status standardization
order mapping
delivery metrics
     │
     ▼
fact_shipments
     │
     ▼
Logistics Dashboard
```

---

# 20. Inventory Snapshot Design

The source ERP contains current inventory.

To support historical reporting, the analytical platform will create a daily snapshot.

Example:

```text
2026-10-05
Warehouse A
SKU001
quantity_on_hand = 200

2026-10-06
Warehouse A
SKU001
quantity_on_hand = 173
```

The mart grain becomes:

```text
snapshot_date
+
warehouse_id
+
product_id
```

This supports:

- stock trends
- inventory-value trends
- slow-moving analysis
- historical availability

---

# 21. File Ingestion Architecture

Files should not be immediately trusted.

Recommended process:

```text
Incoming File
     │
     ▼
File Discovery
     │
     ▼
Schema Validation
     │
     ▼
Business Validation
     │
     ├── FAIL → Reject / Log
     │
     ▼
Standardize
     │
     ▼
Load Staging
     │
     ▼
Record File History
```

File-level audit metadata should include:

```text
file_name
file_type
file_hash
file_modified_at
load_timestamp
rows_read
rows_loaded
rows_rejected
pipeline_run_id
```

A file hash can help prevent accidental duplicate ingestion.

---

# 22. API Ingestion Architecture

Recommended flow:

```text
Start Pipeline
     │
     ▼
Read Previous Watermark
     │
     ▼
Authenticate
     │
     ▼
Request Page
     │
     ▼
Validate Response
     │
     ▼
More Pages? ─── Yes ──> Request Next Page
     │
     No
     ▼
Load Staging
     │
     ▼
Run Validation
     │
     ▼
Update Watermark
```

---

# 23. Retry Strategy

Temporary API or network failures should not immediately fail a pipeline.

Example retry strategy:

```text
Attempt 1
↓
wait 2 seconds

Attempt 2
↓
wait 5 seconds

Attempt 3
↓
wait 15 seconds

Failure
↓
Log pipeline as FAILED
```

Retries should only be used for errors that are likely to be temporary.

Examples:

```text
HTTP 429
HTTP 500
HTTP 502
HTTP 503
connection timeout
```

Permanent errors such as invalid authentication should fail quickly.

---

# 24. Data Quality Architecture

Data-quality checks will run at multiple stages.

```text
Source
  │
  ▼
Ingestion Validation
  │
  ▼
Staging Validation
  │
  ▼
Transformation Validation
  │
  ▼
Business Reconciliation
  │
  ▼
Mart Publication
```

This prevents relying on a single final validation step.

---

# 25. Data Quality Levels

## Level 1 — Technical Validation

Examples:

```text
file exists
API response valid
table accessible
required columns present
```

## Level 2 — Structural Validation

Examples:

```text
primary key not null
duplicate IDs
valid data types
valid date ranges
```

## Level 3 — Referential Validation

Examples:

```text
order.customer_id exists
order_line.product_id exists
shipment.order_id maps to ERP
```

## Level 4 — Business Validation

Examples:

```text
invoice due date >= invoice date
payment amount >= 0
inventory quantity valid
delivery date >= dispatch date
```

## Level 5 — Reconciliation

Examples:

```text
source invoice total
=
staging invoice total
```

within defined tolerance.

---

# 26. Quality Result Storage

Data-quality results should be written to a control table.

Example:

```text
check_id
pipeline_run_id
dataset_name
table_name
check_name
severity
records_checked
failed_records
status
executed_at
details
```

Example:

```text
DQ_10234
RUN_20261006_INVOICES
mis_staging
stg_invoices
duplicate_invoice_id
CRITICAL
4821
0
PASS
2026-10-06 06:12:03
NULL
```

---

# 27. Pipeline Publication Rule

A pipeline completing technically does not automatically mean the data is suitable for reporting.

Example:

```text
Pipeline completed
      ↓
Critical DQ checks
      │
      ├── FAIL
      │     ↓
      │  Do not publish affected mart
      │
      └── PASS
            ↓
      Run transformations
            ↓
      Reconciliation
            │
            ├── FAIL → Flag / Stop publication
            │
            └── PASS
                  ↓
            Reporting mart ready
```

This separates:

```text
data loaded
```

from:

```text
data trusted
```

---

# 28. Transformation Execution

BigQuery SQL will perform most transformations.

Benefits include:

- less data movement
- easy inspection
- scalable SQL execution
- simple lineage
- lower Python complexity

Python should mainly handle:

- extraction
- source connectivity
- load orchestration
- operational control

BigQuery SQL should mainly handle:

- joins
- calculations
- standardization
- aggregations
- dimensional modelling
- business logic

---

# 29. Scheduling Strategy

Initial scheduling should remain simple.

Suggested schedule:

| Pipeline                     | Frequency            |
| ---------------------------- | -------------------- |
| PostgreSQL Master Data       | Daily                |
| Orders / Invoices / Payments | Daily                |
| Inventory Snapshot           | Daily                |
| Sales Target Files           | Daily Check          |
| Finance Files                | Daily Check          |
| Logistics API                | Every 4 Hours        |
| Transformations              | After Ingestion      |
| Data Quality                 | During Each Pipeline |
| Mart Refresh                 | After Validation     |

---

# 30. Example Daily Workflow

```text
05:30
PostgreSQL ingestion starts

05:45
PostgreSQL ingestion completes

05:50
Inventory snapshot completes

06:00
File ingestion checks

06:10
Staging DQ checks

06:20
Transformations begin

06:35
Reconciliation completes

06:40
Data mart updated

07:00
Management dashboard ready
```

Exact times are implementation details and may change.

---

# 31. Orchestration Options

Several orchestration approaches are possible.

## Initial Portfolio Implementation

A lightweight scheduler is preferred.

Possible implementation:

```text
Python scheduler / cron
```

This is sufficient for the project scale.

---

## Possible Production Evolution

If pipeline complexity grows, orchestration could move to:

```text
Google Cloud Scheduler
+
Cloud Run Jobs
```

or a workflow orchestrator.

The portfolio project should not introduce orchestration infrastructure solely to increase the technology count.

---

# 32. Deployment Model

A practical implementation can separate code execution from storage.

```text
GitHub Repository
      │
      ▼
Python Pipeline Runtime
      │
      ├── PostgreSQL
      ├── Files
      ├── API
      │
      ▼
BigQuery
      │
      ▼
Looker Studio
```

The Python runtime could initially run:

- locally during development
- in Docker
- on an existing VPS
- later in Cloud Run Jobs

The code should remain portable.

---

# 33. Containerization

Python ingestion can be packaged into a Docker image.

Example:

```text
mis-pipeline
│
├── Python application
├── dependencies
├── configuration
└── pipeline entry points
```

Benefits:

- repeatable environment
- simple deployment
- easier migration between VPS and cloud runtime
- clean dependency management

---

# 34. Configuration Management

Configuration should not be hardcoded.

Example environment variables:

```text
POSTGRES_HOST
POSTGRES_PORT
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD

GCP_PROJECT_ID
BQ_STAGING_DATASET
BQ_TRANSFORM_DATASET
BQ_MART_DATASET
BQ_CONTROL_DATASET

LOGISTICS_API_BASE_URL
LOGISTICS_API_KEY
```

Secrets should never be committed to GitHub.

---

# 35. Repository Configuration

A `.env.example` may be committed.

Example:

```text
POSTGRES_HOST=
POSTGRES_PORT=5432
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=

GCP_PROJECT_ID=
BQ_STAGING_DATASET=mis_staging
BQ_TRANSFORM_DATASET=mis_transform
BQ_MART_DATASET=mis_mart
BQ_CONTROL_DATASET=mis_control

LOGISTICS_API_BASE_URL=
LOGISTICS_API_KEY=
```

Actual `.env` files should be excluded using `.gitignore`.

---

# 36. Logging

Each pipeline should produce structured logs.

Example:

```text
timestamp
level
pipeline_run_id
pipeline_name
event
message
```

Example:

```text
2026-10-06T06:00:02Z
INFO
RUN_20261006_ORDERS
postgres_sales_orders
EXTRACTION_STARTED
Extracting records after watermark 2026-10-05T06:00:00Z
```

Logging should help answer:

- what ran
- when it ran
- what it loaded
- where it failed
- why it failed

---

# 37. Monitoring

Initial monitoring will use metadata stored in the control tables.

Key operational metrics:

```text
pipeline status
last successful run
rows extracted
rows loaded
rows rejected
pipeline duration
data freshness
DQ failures
reconciliation failures
```

A lightweight operational dashboard can later be created from these tables.

---

# 38. Failure Handling

Pipeline failures should be classified.

## Source Failure

Examples:

```text
PostgreSQL unavailable
API unavailable
file missing
```

## Validation Failure

Examples:

```text
required column missing
duplicate business key
invalid file schema
```

## Load Failure

Examples:

```text
BigQuery load failure
schema incompatibility
```

## Transformation Failure

Examples:

```text
SQL transformation error
unexpected join explosion
```

## Reconciliation Failure

Examples:

```text
source revenue != mart revenue
```

Each failure should be recorded with sufficient context for troubleshooting.

---

# 39. Alerting Strategy

The initial portfolio solution does not require an enterprise alerting platform.

At minimum, the system should be able to identify:

```text
FAILED pipeline
STALE dataset
CRITICAL DQ failure
RECONCILIATION failure
```

Possible later notification channels include:

- email
- Slack
- Microsoft Teams

The notification mechanism can be added after core pipeline monitoring is working.

---

# 40. BigQuery Cost Controls

Cost awareness is part of the solution design.

Recommended controls include:

## Partitioning

Large fact tables should be partitioned by relevant date fields.

Examples:

```text
fact_sales
→ invoice_date

fact_inventory_snapshot
→ snapshot_date

fact_shipments
→ dispatch_date
```

---

## Clustering

Frequently filtered dimensions may be considered for clustering.

Examples:

```text
branch_id
customer_id
product_id
```

Clustering should be added based on realistic query patterns rather than automatically applied everywhere.

---

## Incremental Transformations

Avoid rebuilding complete historical fact tables when only recent data changed.

---

## Select Required Columns

Avoid patterns such as:

```sql
SELECT *
```

in dashboard-facing analytical queries.

---

## Materialized Reporting Tables

Repeated expensive transformations should be calculated once in the mart rather than executed repeatedly by Looker Studio.

---

# 41. BigQuery Storage Strategy

The project should retain:

```text
source-oriented staging
+
curated analytical mart
```

but avoid creating unnecessary copies of the same data.

Temporary load tables may be removed after successful merge operations.

---

# 42. Looker Studio Architecture

Looker Studio should connect to:

```text
mis_mart
```

not directly to:

```text
mis_staging
```

or source systems.

This provides:

- stable KPI definitions
- simplified dashboard queries
- better governance
- lower query complexity
- less risk of inconsistent calculations

---

# 43. Dashboard Data Preparation

Complex calculations should generally happen before the dashboard.

Preferred:

```text
BigQuery:
gross_margin_pct
target_achievement_pct
aging_bucket
delivery_status
```

rather than repeatedly recreating complex business rules inside Looker Studio.

Looker Studio should primarily handle:

- visualization
- simple filtering
- basic display calculations

---

# 44. Security Architecture

The project uses synthetic data, but the solution will follow the principle of least privilege.

Conceptually:

```text
Pipeline Service
→ Read source
→ Write staging
→ Execute transformations

Data Engineering
→ Access staging + transformation + mart

Reporting
→ Read mart

Management
→ Access dashboard
```

Sensitive credentials should be stored outside source code.

---

# 45. Environment Strategy

For a portfolio project, two logical environments are sufficient.

## Development

Used for:

- code development
- schema changes
- small test datasets
- transformation testing

## Production-Like

Used for:

- complete synthetic dataset
- scheduled execution
- dashboard integration
- final portfolio demonstration

A full enterprise DEV/UAT/PROD setup would add unnecessary cost for this project.

---

# 46. Technology Decisions

| Requirement        | Technology                     | Reason                                        |
| ------------------ | ------------------------------ | --------------------------------------------- |
| Operational Source | PostgreSQL                     | Realistic transactional source                |
| File Processing    | Python                         | Flexible validation and parsing               |
| API Integration    | Python                         | Retry, pagination, and authentication support |
| Data Warehouse     | BigQuery                       | Managed analytical warehouse                  |
| Transformation     | BigQuery SQL                   | Keep processing close to data                 |
| Dashboard          | Looker Studio                  | Low-cost reporting                            |
| Scheduling         | Python / Cron initially        | Appropriate for project scale                 |
| Packaging          | Docker                         | Portable runtime                              |
| Version Control    | GitHub                         | Code and documentation                        |
| Monitoring         | BigQuery control tables + logs | Lightweight and transparent                   |

---

# 47. Technologies Intentionally Not Used

The initial implementation will not use:

```text
Kafka
Spark
Kubernetes
large Airflow deployment
real-time CDC platform
```

These technologies are valuable in the appropriate context, but they do not solve a requirement in this scenario.

Avoiding them is a deliberate architecture decision rather than a limitation.

---

# 48. Architecture Decision Summary

The target architecture is:

```text
Multi-Source Batch Inputs
         ↓
Reusable Python Ingestion
         ↓
BigQuery Staging
         ↓
Validation + Standardization
         ↓
BigQuery Transformations
         ↓
Management Data Mart
         ↓
Looker Studio
```

supported by:

```text
watermarks
+
idempotent loading
+
audit metadata
+
data quality
+
reconciliation
+
logging
+
monitoring
```

This provides enough enterprise-style control without overengineering the solution.

---

# 49. Architecture Outcome

The design separates the system into four clear responsibilities:

```text
INGEST
Get data reliably into the platform

VALIDATE
Confirm that data is technically and logically usable

MODEL
Convert source records into business-ready information

SERVE
Provide stable management reporting
```

The resulting solution is designed to be:

- practical
- maintainable
- auditable
- cost-conscious
- scalable beyond the initial portfolio dataset

---

## Next Step

The next design document should be:

**`05-postgresql-source-schema.md`**

This will define the fictional ERP database in detail, including:

- tables
- columns
- data types
- primary keys
- foreign keys
- indexes
- status fields
- timestamps
- relationships
- realistic source-system constraints

After that, we can create the actual PostgreSQL DDL and synthetic data generation scripts.
