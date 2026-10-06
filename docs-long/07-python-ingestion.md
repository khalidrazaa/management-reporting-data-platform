# 07 — Python Ingestion Pipeline Design

## Purpose

This document defines the Python ingestion framework for the Multi-Source MIS & Management Reporting Automation project.

The ingestion layer is responsible for reliably moving data from:

- PostgreSQL ERP
- Excel / CSV business files
- Logistics REST API

into the BigQuery staging layer.

The ingestion framework should provide:

- reusable source connectors
- incremental extraction
- idempotent loading
- schema validation
- file validation
- API pagination and retries
- pipeline audit logging
- watermark management
- error handling
- data-quality integration
- repeatable local and containerized execution

The objective is to create production-style ingestion without building an unnecessarily complex ETL framework.

---

# 1. Ingestion Responsibilities

The Python layer will primarily handle:

```text
CONNECT
    ↓
EXTRACT
    ↓
VALIDATE
    ↓
STANDARDIZE TECHNICAL TYPES
    ↓
LOAD
    ↓
AUDIT
```

Business transformations should generally remain in BigQuery.

Python should not become the primary location for management KPI calculations.

---

# 2. Source-to-Target Flow

```text
                     SOURCE SYSTEMS

        PostgreSQL       Excel / CSV       REST API
             │                │                │
             └────────────────┼────────────────┘
                              │
                              ▼
                    Python Ingestion
                              │
                    ┌─────────┼─────────┐
                    │         │         │
                 Extract   Validate   Audit
                    │         │         │
                    └─────────┼─────────┘
                              │
                              ▼
                   BigQuery Load Batch
                              │
                              ▼
                     MERGE / APPEND
                              │
                              ▼
                    BigQuery Staging
                              │
                              ▼
                       DQ Validation
```

---

# 3. Design Principles

## Source-Agnostic Framework

Common concerns such as:

- configuration
- logging
- BigQuery loading
- pipeline runs
- watermarks
- exceptions

should be shared across ingestion pipelines.

Source-specific extraction logic should remain isolated.

---

## Idempotency

A pipeline should be safe to rerun.

Running the same extraction twice should not create duplicate business records.

---

## Incremental Processing

Transactional source tables should load changed records rather than repeatedly extracting all historical data.

---

## Traceability

Every loaded record should be traceable to a pipeline execution.

---

## Fail Clearly

Failures should produce useful operational information.

A pipeline should not fail with only:

```text
Something went wrong
```

It should identify:

```text
pipeline
source
stage
run ID
exception
watermark
record counts
```

where applicable.

---

## Keep Business Logic Out of Ingestion

Python may standardize technical representations such as:

```text
timestamps
column names
null formats
numeric types
```

but calculations such as:

```text
gross margin
aging bucket
sales target achievement
on-time delivery percentage
```

belong in BigQuery transformations.

---

# 4. Proposed Repository Structure

```text
ingestion/
│
├── __init__.py
│
├── cli.py
│
│
├── common/
│   ├── __init__.py
│   ├── config.py
│   ├── database.py
│   ├── bigquery.py
│   ├── logging.py
│   ├── pipeline.py
│   ├── control.py
│   ├── watermark.py
│   ├── validation.py
│   └── exceptions.py
│
├── postgres/
│   ├── __init__.py
│   ├── client.py
│   ├── extractor.py
│   ├── config.py
│   └── pipelines.py
│
├── files/
│   ├── __init__.py
│   ├── reader.py
│   ├── validator.py
│   ├── file_registry.py
│   └── pipelines.py
│
└── api/
    ├── __init__.py
    ├── client.py
    ├── pagination.py
    ├── retry.py
    └── shipments.py
```

Additional SQL resources may be stored separately:

```text
bigquery/
├── staging/
├── transformations/
└── marts/
```

---

# 5. Configuration

Pipeline configuration should be separated from application code.

Example environment variables:

```text
POSTGRES_HOST=
POSTGRES_PORT=5432
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=

GCP_PROJECT_ID=

BQ_STAGING_DATASET=mis_staging
BQ_CONTROL_DATASET=mis_control
BQ_TRANSFORM_DATASET=mis_transform
BQ_MART_DATASET=mis_mart

LOGISTICS_API_BASE_URL=
LOGISTICS_API_KEY=

LOG_LEVEL=INFO
```

Actual credentials should never be committed to GitHub.

---

# 6. `.env.example`

The repository can contain:

```text
.env.example
```

with:

```text
POSTGRES_HOST=
POSTGRES_PORT=5432
POSTGRES_DB=apex_erp
POSTGRES_USER=
POSTGRES_PASSWORD=

GCP_PROJECT_ID=
BQ_STAGING_DATASET=mis_staging
BQ_CONTROL_DATASET=mis_control
BQ_TRANSFORM_DATASET=mis_transform
BQ_MART_DATASET=mis_mart

LOGISTICS_API_BASE_URL=
LOGISTICS_API_KEY=

LOG_LEVEL=INFO
```

The real:

```text
.env
```

must be included in `.gitignore`.

---

# 7. Configuration Loader

`config.py` should expose validated application configuration.

Conceptually:

```python
class Settings:
    postgres_host: str
    postgres_port: int
    postgres_db: str

    gcp_project_id: str

    bq_staging_dataset: str
    bq_control_dataset: str

    logistics_api_base_url: str
```

The application should fail immediately if mandatory configuration is missing.

This is preferable to discovering missing credentials halfway through a pipeline.

---

# 8. Pipeline Run Identifier

Every pipeline execution should receive a unique identifier.

Example:

```text
RUN_20261006_053001_POSTGRES_INVOICES
```

A simpler UUID may also be used internally.

The important requirement is uniqueness.

The run ID should be attached to:

- pipeline logs
- staging records
- data-quality results
- control-table records
- reconciliation results

---

# 9. Pipeline Lifecycle

Every ingestion pipeline should follow a common lifecycle.

```text
START
  │
  ▼
Create Pipeline Run
  │
  ▼
Read Configuration
  │
  ▼
Determine Extraction Window
  │
  ▼
Extract
  │
  ▼
Validate
  │
  ▼
Load
  │
  ▼
Run Load Checks
  │
  ▼
Commit Watermark
  │
  ▼
Mark SUCCESS
```

On failure:

```text
Exception
    │
    ▼
Record Error
    │
    ▼
Do NOT advance watermark
    │
    ▼
Mark FAILED
```

---

# 10. Pipeline Control Table

BigQuery dataset:

```text
mis_control
```

Table:

```text
pipeline_runs
```

Suggested structure:

| Column | Description |
|---|---|
| pipeline_run_id | Unique execution identifier |
| pipeline_name | Pipeline name |
| source_system | Source |
| target_table | Target staging table |
| started_at | Start timestamp |
| completed_at | Completion timestamp |
| status | STARTED / SUCCESS / FAILED |
| watermark_from | Beginning of extraction window |
| watermark_to | End of extraction window |
| rows_extracted | Source rows |
| rows_loaded | Loaded rows |
| rows_rejected | Rejected rows |
| error_type | Failure classification |
| error_message | Failure details |

---

# 11. Pipeline Watermark Table

Suggested table:

```text
mis_control.pipeline_watermarks
```

Columns:

```text
pipeline_name
source_object
watermark_column
last_successful_watermark
updated_at
last_pipeline_run_id
```

Example:

```text
pipeline_name:
postgres_invoices

source_object:
public.invoices

watermark_column:
updated_at

last_successful_watermark:
2026-10-05T05:30:00Z
```

---

# 12. Watermark Extraction Pattern

For transactional PostgreSQL tables:

```sql
SELECT
    ...
FROM invoices
WHERE updated_at > :watermark_from
  AND updated_at <= :watermark_to
ORDER BY updated_at;
```

The pipeline determines:

```text
watermark_from
```

from the last successful run.

The pipeline determines:

```text
watermark_to
```

at the start of the current execution.

---

# 13. Watermark Example

Previous successful run:

```text
2026-10-05 05:30:00 UTC
```

Current run starts:

```text
2026-10-06 05:30:04 UTC
```

Extraction:

```text
updated_at > 2026-10-05 05:30:00
AND
updated_at <= 2026-10-06 05:30:04
```

After successful load:

```text
last_successful_watermark
=
2026-10-06 05:30:04
```

---

# 14. Failed Watermark Rule

The watermark must not advance if:

```text
extraction fails
load fails
critical validation fails
```

Example:

```text
Run 1

05-Oct 05:30 → SUCCESS

watermark = 05-Oct 05:30
```

Next:

```text
Run 2

06-Oct 05:30 → FAILED

watermark remains:
05-Oct 05:30
```

Next successful run reprocesses:

```text
05-Oct 05:30
→
07-Oct current watermark
```

This prevents data loss.

---

# 15. PostgreSQL Connection

The PostgreSQL connector should use a pooled or reusable database connection.

Possible libraries:

```text
psycopg
```

or:

```text
SQLAlchemy
```

Since the project also demonstrates data engineering practices, SQLAlchemy may be used for connection management while SQL remains explicit for extraction.

---

# 16. PostgreSQL Extractor

The extractor should be configurable rather than writing a completely different extraction engine for every table.

Conceptual definition:

```python
PipelineDefinition(
    name="postgres_invoices",
    source_table="invoices",
    target_table="stg_invoices",
    load_type="incremental",
    watermark_column="updated_at",
    business_key=["invoice_id"],
)
```

The pipeline can use this metadata to execute common behavior.

---

# 17. PostgreSQL Load Strategies

Not every table requires the same pattern.

## Full Refresh

Recommended for small master tables:

```text
branches
warehouses
```

Possible workflow:

```text
Extract complete source
        ↓
Load temporary table
        ↓
Validate
        ↓
Replace staging table
```

---

## Incremental Merge

Recommended for:

```text
customers
products
sales_representatives
sales_orders
sales_order_lines
invoices
invoice_lines
payments
```

Workflow:

```text
Extract changed records
        ↓
Load batch table
        ↓
MERGE into persistent staging
```

---

## Snapshot

Recommended for:

```text
inventory
```

Workflow:

```text
Read current inventory
        ↓
Add snapshot_date
        ↓
Append to inventory snapshot
```

---

# 18. Extraction Batch Size

The pipeline should support chunked reading for larger source tables.

Example:

```text
10,000
or
25,000 rows per batch
```

The exact value should be configurable.

Example:

```text
POSTGRES_FETCH_SIZE=10000
```

This prevents unnecessarily loading large extracts completely into Python memory.

---

# 19. DataFrame Usage

Pandas may be used for manageable ingestion batches.

However, the pipeline should avoid patterns such as:

```text
load entire million-row source table
into one DataFrame
```

when streaming or chunking is easy.

Pandas should serve as a practical processing utility rather than becoming the data-processing engine for the warehouse.

---

# 20. Standard Ingestion Metadata

Before loading into BigQuery, records should receive technical columns.

Recommended:

```text
_source_system
_source_table
_pipeline_run_id
_ingestion_timestamp
_source_updated_at
```

Example:

```text
_source_system      ERP_POSTGRES
_source_table       invoices
_pipeline_run_id    RUN_20261006_053001_POSTGRES_INVOICES
_ingestion_timestamp 2026-10-06T05:31:14Z
_source_updated_at  2026-10-05T18:24:10Z
```

---

# 21. BigQuery Loading

The Python ingestion framework should use the official Google Cloud BigQuery client.

Conceptually:

```python
from google.cloud import bigquery
```

The pipeline should load records into:

```text
mis_staging
```

rather than directly into the reporting mart.

---

# 22. Load Batch Tables

For incremental loads, create a temporary or short-lived batch table.

Example:

```text
mis_staging_load
    .sales_orders_RUN_20261006
```

or another controlled temporary naming convention.

Flow:

```text
Python Extract
     ↓
Batch Table
     ↓
Validation
     ↓
MERGE
     ↓
Persistent Staging
```

---

# 23. Persistent Staging Tables

Examples:

```text
mis_staging.stg_customers
mis_staging.stg_products
mis_staging.stg_sales_orders
mis_staging.stg_sales_order_lines
mis_staging.stg_invoices
mis_staging.stg_invoice_lines
mis_staging.stg_payments
```

These represent the latest known source record.

---

# 24. BigQuery MERGE Pattern

Example:

```sql
MERGE `mis_staging.stg_invoices` AS target

USING `load_batch.invoices_current_run` AS source

ON target.invoice_id = source.invoice_id

WHEN MATCHED
AND source.updated_at >= target.updated_at
THEN UPDATE SET
    invoice_number = source.invoice_number,
    invoice_status = source.invoice_status,
    invoice_amount = source.invoice_amount,
    cancelled_amount = source.cancelled_amount,
    updated_at = source.updated_at,
    _pipeline_run_id = source._pipeline_run_id,
    _ingestion_timestamp = source._ingestion_timestamp

WHEN NOT MATCHED THEN
INSERT (
    invoice_id,
    invoice_number,
    invoice_status,
    invoice_amount,
    cancelled_amount,
    updated_at,
    _pipeline_run_id,
    _ingestion_timestamp
)
VALUES (
    source.invoice_id,
    source.invoice_number,
    source.invoice_status,
    source.invoice_amount,
    source.cancelled_amount,
    source.updated_at,
    source._pipeline_run_id,
    source._ingestion_timestamp
);
```

---

# 25. Why MERGE Is Used

A record may already exist in BigQuery but later change in PostgreSQL.

Example:

```text
Invoice 100234

Day 1:
POSTED

Day 15:
PARTIALLY_PAID

Day 30:
PAID
```

Incremental ingestion needs to update the staging representation rather than add duplicate invoices.

---

# 26. File Ingestion Sources

Files include:

```text
sales_targets_2026.xlsx
branch_budget_2026.xlsx
monthly_operating_expenses.xlsx
product_margin_adjustments.csv
```

File ingestion requires a different pipeline pattern because files may be manually changed or replaced.

---

# 27. File Pipeline Flow

```text
Locate File
    ↓
Calculate File Hash
    ↓
Check Load History
    ↓
Validate File Structure
    ↓
Read File
    ↓
Validate Rows
    ↓
Add Audit Metadata
    ↓
Load BigQuery
    ↓
Record File History
```

---

# 28. File Hashing

Each file should receive a checksum such as:

```text
SHA-256
```

Example:

```text
file_name:
sales_targets_2026.xlsx

file_hash:
af3957...
```

Before ingestion:

```text
Has this exact hash already been loaded?
```

If yes:

```text
skip duplicate load
```

unless explicitly forced.

---

# 29. File Load History

Suggested control table:

```text
mis_control.file_load_history
```

Fields:

```text
file_load_id
pipeline_run_id
file_name
file_hash
file_modified_at
rows_read
rows_loaded
rows_rejected
status
loaded_at
```

---

# 30. File Schema Validation

Before loading:

```text
required columns present?
unexpected critical columns?
correct worksheet?
valid header?
```

Example sales target schema:

```text
month
sales_rep_code
branch_code
revenue_target
```

If:

```text
revenue_target
```

is renamed to:

```text
target
```

unexpectedly, the pipeline should not silently load the file.

---

# 31. File Business-Key Validation

Example:

```text
sales_targets
business key:

month
+
sales_rep_code
```

Duplicate combinations should trigger a validation failure.

Example invalid data:

```text
2026-10 | SR001 | 2500000
2026-10 | SR001 | 2600000
```

The pipeline should not guess which value is correct.

---

# 32. File Type Standardization

Typical spreadsheet problems include:

```text
"1,250,000"
"₹1,250,000"
"1250000"
```

The ingestion layer may normalize these into numeric values.

Likewise:

```text
Oct-26
2026-10
01/10/2026
```

may be standardized into the expected reporting month representation.

This is technical normalization rather than business transformation.

---

# 33. Rejected File Rows

Where appropriate, invalid rows should be identifiable.

Possible model:

```text
source_file
row_number
error_code
error_message
raw_record
pipeline_run_id
```

For critical file issues such as duplicate business keys, the entire file may be rejected instead of partially loaded.

The decision should depend on severity.

---

# 34. Logistics API Pipeline

The logistics API introduces:

- authentication
- HTTP status handling
- pagination
- rate limits
- retries
- incremental filtering

Recommended endpoint pattern:

```http
GET /shipments?updated_since=...
```

---

# 35. API Extraction Window

Like PostgreSQL ingestion, API ingestion should use an incremental watermark where possible.

Example:

```text
updated_since =
2026-10-06T00:00:00Z
```

The API may return any shipment changed after that time.

---

# 36. API Pagination

Example API:

```http
GET /shipments?page=1&page_size=500
```

Response:

```json
{
  "data": [],
  "page": 1,
  "page_size": 500,
  "total_pages": 14
}
```

The client should continue until all pages are retrieved.

---

# 37. Pagination Logic

Conceptually:

```text
page = 1

while page <= total_pages:

    fetch page

    validate response

    collect / load records

    page += 1
```

For large APIs, pages may be loaded incrementally instead of accumulating all data in memory.

---

# 38. API Timeouts

Every request should define explicit timeouts.

Example:

```text
connect timeout:
5 seconds

read timeout:
30 seconds
```

The pipeline should not wait indefinitely for a remote service.

---

# 39. Retry Strategy

Retry temporary failures such as:

```text
HTTP 429
HTTP 500
HTTP 502
HTTP 503
HTTP 504
connection reset
timeout
```

Example backoff:

```text
Attempt 1
↓
2 seconds

Attempt 2
↓
5 seconds

Attempt 3
↓
15 seconds
```

---

# 40. Non-Retryable API Errors

Do not repeatedly retry obvious permanent failures such as:

```text
HTTP 400
HTTP 401
HTTP 403
```

These usually indicate:

- bad request
- authentication failure
- authorization failure

The pipeline should fail clearly.

---

# 41. Rate Limit Handling

If the API returns:

```text
HTTP 429
```

the client should inspect:

```text
Retry-After
```

where available.

Rate limits should be respected rather than bypassed through excessive requests.

---

# 42. API Response Validation

Before loading:

```text
response is valid JSON
required fields exist
shipment_id exists
updated_at exists
dates are parseable
```

Unexpected schema changes should be detected.

---

# 43. Shipment Idempotency

Business key:

```text
shipment_id
```

Repeated shipment records should update the current staging record.

Example:

```text
Morning:

SHP100
status = IN_TRANSIT
```

Later:

```text
SHP100
status = DELIVERED
```

BigQuery staging should contain the newest source state.

---

# 44. Late-Arriving Updates

A logistics event may occur before the API exposes the update.

Example:

```text
actual delivery:
2026-10-05 16:00

API record updated:
2026-10-06 02:00
```

Using:

```text
updated_at
```

rather than only:

```text
delivery_date
```

allows the pipeline to capture the change.

---

# 45. Inventory Pipeline

Inventory differs from mutable master tables.

The source contains:

```text
current inventory state
```

but reporting requires:

```text
historical inventory state
```

Therefore the ingestion process creates snapshots.

---

# 46. Inventory Snapshot Flow

```text
Extract Current Inventory
        ↓
Validate
        ↓
Add snapshot_date
        ↓
Append BigQuery Snapshot
```

Target grain:

```text
snapshot_date
+
warehouse_id
+
product_id
```

---

# 47. Snapshot Idempotency

If the inventory pipeline is rerun for the same day, it should not create duplicate snapshots.

Possible strategy:

```text
DELETE target partition for snapshot_date
        ↓
INSERT replacement snapshot
```

or:

```text
MERGE
```

using:

```text
snapshot_date
warehouse_id
product_id
```

as the business key.

---

# 48. Initial Historical Load

The first execution differs from daily incremental operation.

Initial load:

```text
PostgreSQL historical tables
        ↓
Full historical extraction
        ↓
BigQuery staging
```

This includes approximately:

```text
3 years
```

of transactional data.

---

# 49. Initial Load Strategy

Large tables should be processed in batches.

Possible methods:

```text
date-range chunks
```

Example:

```text
2024-01
2024-02
2024-03
...
```

or:

```text
primary-key ranges
```

Date-based extraction is easier to audit for this project.

---

# 50. Initial Load vs Incremental Mode

Pipeline CLI should distinguish:

```text
initial
```

and:

```text
incremental
```

Example:

```bash
python -m ingestion.cli \
    postgres \
    invoices \
    --mode initial
```

Later:

```bash
python -m ingestion.cli \
    postgres \
    invoices \
    --mode incremental
```

---

# 51. Proposed CLI

A single pipeline entry point is preferable to many unrelated scripts.

Conceptually:

```bash
python -m ingestion.cli postgres invoices
```

```bash
python -m ingestion.cli postgres sales_orders
```

```bash
python -m ingestion.cli files sales_targets
```

```bash
python -m ingestion.cli api shipments
```

---

# 52. Group Execution

The CLI may support logical groups.

Example:

```bash
python -m ingestion.cli postgres all
```

or:

```bash
python -m ingestion.cli daily
```

where:

```text
daily
```

runs the expected daily ingestion sequence.

---

# 53. Dry Run

A useful portfolio feature is:

```text
--dry-run
```

Example:

```bash
python -m ingestion.cli files sales_targets --dry-run
```

Dry run can:

- validate configuration
- locate source
- validate schema
- display expected rows
- avoid loading BigQuery

This helps operational troubleshooting.

---

# 54. Logging

Logging should be structured and consistent.

Example fields:

```text
timestamp
level
pipeline_run_id
pipeline_name
source
target
event
message
```

Example:

```text
2026-10-06T05:30:02Z
INFO
RUN_20261006_053001_POSTGRES_INVOICES
postgres_invoices
ERP_POSTGRES
mis_staging.stg_invoices
EXTRACTION_STARTED
Incremental extraction started
```

---

# 55. Important Pipeline Events

Examples:

```text
PIPELINE_STARTED

CONNECTION_ESTABLISHED

EXTRACTION_STARTED
EXTRACTION_COMPLETED

VALIDATION_STARTED
VALIDATION_COMPLETED

LOAD_STARTED
LOAD_COMPLETED

MERGE_STARTED
MERGE_COMPLETED

WATERMARK_UPDATED

PIPELINE_SUCCESS
PIPELINE_FAILED
```

---

# 56. Error Classification

Custom exceptions can make failures easier to understand.

Examples:

```text
ConfigurationError
SourceConnectionError
ExtractionError
SchemaValidationError
DataValidationError
BigQueryLoadError
MergeError
APIAuthenticationError
APIResponseError
```

The goal is not to create dozens of unnecessary exception classes, but to distinguish meaningful failure types.

---

# 57. Error Handling Pattern

Conceptually:

```python
try:
    start_pipeline()

    extract()

    validate()

    load()

    commit_watermark()

    mark_success()

except Exception as exc:

    mark_failed(exc)

    raise
```

The pipeline should never mark success if downstream loading failed.

---

# 58. Validation Before Load

Technical validation should happen before records are written.

Examples:

```text
required columns exist
primary business key present
timestamp parseable
numeric columns valid
source batch not unexpectedly empty
```

---

# 59. Empty Dataset Handling

An empty incremental result does not necessarily indicate failure.

Example:

```text
No invoices changed since last run.
```

The pipeline can record:

```text
SUCCESS
rows_extracted = 0
rows_loaded = 0
```

However, an unexpectedly empty full source table may be a warning or failure.

Context matters.

---

# 60. Row Count Validation

For each batch:

```text
rows_extracted
```

should reconcile with:

```text
rows_loaded
+
rows_rejected
```

For clean transactional source ingestion:

```text
rows_extracted
=
rows_loaded
```

should normally hold.

---

# 61. Financial Reconciliation

After ingestion, source-level metrics can be compared.

Example:

```text
PostgreSQL invoices:

COUNT(*)
SUM(invoice_amount)
```

versus:

```text
BigQuery staging:

COUNT(*)
SUM(invoice_amount)
```

for the extraction window.

This provides stronger assurance than row counts alone.

---

# 62. Reconciliation Result Table

Suggested table:

```text
mis_control.reconciliation_results
```

Fields:

```text
reconciliation_id
pipeline_run_id
source_name
target_name
metric_name
source_value
target_value
difference
tolerance
status
executed_at
```

---

# 63. Schema Drift

Source schemas may change.

Examples:

```text
new column
missing column
changed type
renamed field
```

Critical expected-column changes should not silently pass.

For initial implementation:

```text
required fields
```

will be explicitly validated.

Additional non-critical fields may be ignored until mapped.

---

# 64. BigQuery Schema Management

Production staging schemas should ideally be defined explicitly.

Avoid relying entirely on automatic schema inference for critical tables.

Explicit schemas make type behavior predictable.

Example:

```text
invoice_id       INTEGER
invoice_number   STRING
invoice_date     DATE
invoice_amount   NUMERIC
updated_at       TIMESTAMP
```

---

# 65. Numeric Precision

Financial values should map:

```text
PostgreSQL NUMERIC
```

to:

```text
BigQuery NUMERIC
```

where possible.

Avoid converting money values to floating point unnecessarily.

---

# 66. Timestamp Handling

Source timestamps should be normalized to UTC for technical processing.

Example:

```text
2026-10-06T05:30:00Z
```

Business calendar reporting may later interpret dates in the relevant business timezone.

The distinction between:

```text
technical event time
```

and:

```text
business reporting date
```

should remain clear.

---

# 67. Data Loading Order

Certain datasets should be ingested before dependent datasets where practical.

Suggested daily PostgreSQL sequence:

```text
branches
warehouses
sales_representatives
customers
products
        ↓
sales_orders
sales_order_lines
        ↓
invoices
invoice_lines
payments
        ↓
inventory
```

This makes downstream validation easier.

---

# 68. Dependency Handling

A child pipeline should not necessarily crash merely because the parent contains no new rows.

For example:

```text
customers:
0 changes

orders:
150 changes
```

is perfectly valid.

Dependencies relate primarily to availability and successful reference data ingestion, not identical extraction activity.

---

# 69. File Pipeline Schedule

File pipelines can run daily to check whether files changed.

Example:

```text
06:00
check incoming folder
```

If:

```text
file absent
```

behavior depends on the source expectation.

For example:

### Sales Targets

If the latest approved target file is already loaded:

```text
No new file
→ SUCCESS / NO_CHANGE
```

### Required Month-End Expense File

If expected by deadline but missing:

```text
WARNING
or
FAILED SLA check
```

---

# 70. Pipeline Statuses

Recommended statuses:

```text
STARTED
SUCCESS
FAILED
PARTIAL_SUCCESS
NO_CHANGE
```

`NO_CHANGE` is particularly useful for:

- files
- empty incremental loads

---

# 71. Partial Success

Use sparingly.

Example:

```text
API retrieved 10 pages
page 11 repeatedly failed
```

This should normally be treated as:

```text
FAILED
```

rather than partially publishing incomplete data.

`PARTIAL_SUCCESS` is more appropriate when independent sub-components have clear isolation.

---

# 72. Secrets Management

During local development:

```text
.env
```

may be used.

For production-like deployment, secrets can later move to:

```text
environment variables
Docker secrets
cloud secret manager
```

The application interface should remain unchanged.

---

# 73. Google Authentication

For local development, BigQuery authentication can use:

```text
Application Default Credentials
```

or a service account where appropriate.

The repository must never contain service-account private keys.

---

# 74. Dockerization

The ingestion runtime should eventually be containerized.

Conceptual Docker image:

```text
management-reporting-pipeline
```

Containing:

```text
Python runtime
application code
dependencies
pipeline CLI
```

---

# 75. Container Runtime

Example:

```bash
docker run \
  --env-file .env \
  management-reporting-pipeline \
  python -m ingestion.cli postgres invoices
```

The same image can later run scheduled jobs.

---

# 76. Python Dependencies

Initial dependencies may include:

```text
google-cloud-bigquery
google-cloud-bigquery-storage
sqlalchemy
psycopg
pandas
openpyxl
requests
python-dotenv
pydantic
tenacity
```

Not every dependency is mandatory.

For example, retry logic could be implemented directly instead of using `tenacity`.

Keep the dependency set intentional.

---

# 77. Dependency Management

Use:

```text
pyproject.toml
```

or:

```text
requirements.txt
```

A modern project structure using `pyproject.toml` is preferable if practical.

Dependencies should be pinned sufficiently for reproducible builds.

---

# 78. Unit Testing

Unit tests should focus on reusable logic.

Examples:

```text
watermark calculation
file hash generation
schema validation
numeric normalization
API pagination
retry classification
pipeline status handling
```

---

# 79. Integration Testing

Integration tests should verify:

```text
PostgreSQL → Python
Python → BigQuery
file → BigQuery
mock API → BigQuery
```

A development configuration with small data volumes should support these tests.

---

# 80. Idempotency Tests

Important test:

```text
Run invoice pipeline
        ↓
Run same extraction again
        ↓
Target row count must not double
```

Similar tests should exist for:

- sales orders
- payments
- shipments
- inventory snapshots

---

# 81. Watermark Recovery Test

Test scenario:

```text
Last watermark = T1

Pipeline starts for T1 → T2

Load intentionally fails

Expected:
watermark remains T1
```

After fixing:

```text
rerun

Expected:
extract T1 → new T3
```

This demonstrates reliable recovery.

---

# 82. File Duplicate Test

Test:

```text
load sales_targets_2026.xlsx
```

Then run again with the exact same file.

Expected:

```text
file hash already processed

status:
NO_CHANGE
```

The target should not duplicate the data.

---

# 83. API Retry Test

Mock API should simulate:

```text
HTTP 503
```

for the first request.

Second request succeeds.

Expected:

```text
pipeline retries
pipeline succeeds
```

Another test should simulate permanent:

```text
HTTP 401
```

Expected:

```text
fail immediately
```

---

# 84. Observability

A user should be able to answer:

```text
Did the pipeline run?

When was the last successful run?

How many rows were extracted?

How many were loaded?

Did validation pass?

What watermark was processed?

Why did the last failure occur?
```

without reading application source code.

That is the main purpose of the control layer.

---

# 85. Operational Query Examples

Example latest pipeline status:

```sql
SELECT
    pipeline_name,
    status,
    completed_at,
    rows_loaded
FROM `mis_control.pipeline_runs`
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY pipeline_name
    ORDER BY started_at DESC
) = 1;
```

---

# 86. Stale Pipeline Detection

Conceptually:

```text
current_time
-
last_successful_run
>
expected_refresh_interval
```

Then:

```text
dataset status = STALE
```

Example:

```text
invoice pipeline

expected:
24 hours

last successful:
31 hours ago

status:
STALE
```

---

# 87. Initial Pipeline Catalogue

Recommended first pipeline definitions:

| Pipeline | Source | Load Pattern | Target |
|---|---|---|---|
| postgres_branches | PostgreSQL | Full | stg_branches |
| postgres_warehouses | PostgreSQL | Full | stg_warehouses |
| postgres_sales_reps | PostgreSQL | Incremental | stg_sales_representatives |
| postgres_customers | PostgreSQL | Incremental | stg_customers |
| postgres_products | PostgreSQL | Incremental | stg_products |
| postgres_sales_orders | PostgreSQL | Incremental | stg_sales_orders |
| postgres_order_lines | PostgreSQL | Incremental | stg_sales_order_lines |
| postgres_invoices | PostgreSQL | Incremental | stg_invoices |
| postgres_invoice_lines | PostgreSQL | Incremental | stg_invoice_lines |
| postgres_payments | PostgreSQL | Incremental | stg_payments |
| postgres_inventory | PostgreSQL | Snapshot | stg_inventory_snapshot |
| file_sales_targets | Excel | File Version | stg_sales_targets |
| file_branch_budget | Excel | File Version | stg_branch_budget |
| file_operating_expenses | Excel | File Version | stg_operating_expenses |
| file_margin_adjustments | CSV | File Version | stg_margin_adjustments |
| api_shipments | REST API | Incremental | stg_shipments |

---

# 88. Recommended Implementation Order

Do not implement every pipeline simultaneously.

Suggested sequence:

```text
1. Common configuration
2. Logging
3. BigQuery client
4. Pipeline control tables
5. Watermark handling

6. PostgreSQL branches
7. PostgreSQL customers
8. PostgreSQL sales_orders

9. Generalize PostgreSQL pattern

10. Remaining PostgreSQL tables

11. Excel ingestion
12. CSV ingestion

13. Logistics API client

14. Inventory snapshot

15. Reconciliation

16. Integration tests
```

This allows the framework to emerge from working examples rather than being overdesigned upfront.

---

# 89. First End-to-End Pipeline

The first complete implementation should probably be:

```text
PostgreSQL
sales_orders
        ↓
Python incremental extraction
        ↓
BigQuery load batch
        ↓
MERGE
        ↓
mis_staging.stg_sales_orders
        ↓
pipeline audit
        ↓
watermark update
```

Why `sales_orders`?

It demonstrates:

- realistic transactional volume
- incremental extraction
- updates
- business keys
- watermarking
- BigQuery MERGE
- reconciliation

without the additional financial complexity of invoices and payments.

---

# 90. Definition of Done — Individual Pipeline

A pipeline should not be considered complete merely when data appears in BigQuery.

Definition of done:

- connects to source successfully
- extracts expected data
- handles initial load
- handles incremental load
- captures metadata
- loads correct BigQuery types
- is idempotent
- records pipeline execution
- updates watermark only after success
- supports meaningful errors
- passes row-count reconciliation
- has at least basic automated tests
- is documented

---

# 91. Definition of Done — Ingestion Layer

The ingestion layer will be considered complete when:

1. PostgreSQL source tables load automatically.
2. Incremental watermarks work correctly.
3. Re-running a batch does not duplicate transactional records.
4. File ingestion detects duplicate files.
5. File schemas are validated before loading.
6. API pagination works.
7. API retries handle temporary failures.
8. Inventory snapshots preserve daily state.
9. Pipeline runs are recorded centrally.
10. Critical failures do not advance watermarks.
11. Source and target row counts are reconciled.
12. All staging records contain appropriate lineage metadata.
13. The framework runs locally and in Docker.

---

# 92. Engineering Principle

The ingestion layer should remain deliberately boring.

That is a positive quality.

It should reliably answer:

```text
What changed?
↓
Can I trust the extract?
↓
Did I load it correctly?
↓
Can I safely run it again?
↓
Can I explain what happened?
```

The value comes from reliability and maintainability rather than clever code.

---

# 93. Ingestion Outcome

At the completion of this phase, the platform should have:

```text
PostgreSQL
     │
     ├──────────────┐
     │              │
Excel / CSV         │
     │              │
     ├──────────────┤
     │              │
REST API            │
     │              │
     └──────┬───────┘
            │
            ▼
      Python Ingestion
            │
            ├── watermarks
            ├── validation
            ├── retries
            ├── logging
            ├── reconciliation
            └── audit metadata
            │
            ▼
      BigQuery Staging
```

This creates a reliable foundation for all downstream modelling and reporting.

---

## Next Step

The next project document should be:

**`08-bigquery-data-model.md`**

It will define:

- BigQuery datasets
- staging schemas
- transformation models
- dimension tables
- fact tables
- table grain
- surrogate keys
- date dimension
- Slowly Changing Dimension decisions
- partitioning
- clustering
- incremental transformations
- fact-to-dimension relationships
- reporting mart design

This is where the multi-source operational data will be converted into a management-oriented analytical model.