# 09 — Data Quality Framework

## Purpose

This document defines the data-quality framework for the Multi-Source MIS & Management Reporting Automation project.

The objective is to ensure that data reaching the management reporting layer is:

- complete
- valid
- consistent
- timely
- reconcilable
- traceable
- suitable for business use

The framework applies across:

- PostgreSQL ingestion
- Excel / CSV ingestion
- Logistics API ingestion
- BigQuery staging
- transformations
- fact and dimension tables
- KPI reconciliation
- dashboard publication

The principle is simple:

> Successful data movement does not automatically mean trusted data.

---

# 1. Data Quality Objectives

The framework should answer five questions.

## Completeness

Did all expected data arrive?

## Validity

Are values structurally and logically valid?

## Consistency

Do related records agree with each other?

## Accuracy / Reconciliation

Do analytical totals reconcile with authoritative sources?

## Freshness

Is the data recent enough for the agreed reporting SLA?

---

# 2. Quality Control Architecture

```text
Source Systems
      │
      ▼
Ingestion Validation
      │
      ▼
BigQuery Staging
      │
      ▼
Structural & Referential Checks
      │
      ▼
Transformation Layer
      │
      ▼
Business Rule Checks
      │
      ▼
Data Mart
      │
      ▼
Reconciliation & Freshness
      │
      ▼
Dashboard Publication
```

Quality checks should run at several points rather than only after the entire pipeline finishes.

---

# 3. Quality Dimensions

The project will use the following quality dimensions.

| Dimension | Meaning |
|---|---|
| Completeness | Required records and fields are present |
| Uniqueness | Business keys are not duplicated |
| Validity | Values follow expected types, ranges, and formats |
| Consistency | Related fields and systems agree |
| Referential Integrity | Foreign/business references exist |
| Timeliness | Data is refreshed within SLA |
| Reconciliation | Totals align across source and target |
| Conformity | Values follow agreed standards |

---

# 4. Quality Check Levels

Quality checks will be grouped into five levels.

## Level 1 — Technical Availability

Questions:

```text
Can the source be reached?
Does the expected file exist?
Did the API respond?
Is the target writable?
```

Examples:

- PostgreSQL connection succeeds
- Excel file can be opened
- API returns valid HTTP response
- BigQuery dataset exists

---

## Level 2 — Structural Validation

Questions:

```text
Does the expected schema exist?
Are mandatory columns present?
Are primary keys populated?
Are data types usable?
```

Examples:

- `invoice_id` exists
- `invoice_date` is parseable
- `payment_amount` is numeric
- required spreadsheet columns are present

---

## Level 3 — Referential Validation

Questions:

```text
Do related records exist?
Can source identifiers be mapped?
```

Examples:

```text
sales_order.customer_id
→ valid customer

invoice_line.product_id
→ valid product

shipment.order_id
→ valid order
```

---

## Level 4 — Business Validation

Questions:

```text
Does the record make business sense?
```

Examples:

- invoice due date is not before invoice date
- payment amount is positive
- delivery date is not before dispatch date
- inventory available quantity is valid

---

## Level 5 — Reconciliation

Questions:

```text
Does the target reproduce the source correctly?
Do KPI totals reconcile?
```

Examples:

- invoice totals source vs staging
- payment totals source vs mart
- finance budget file vs mart
- source margin adjustments vs allocated adjustments

---

# 5. Severity Levels

Each quality check should have a severity.

Recommended levels:

```text
INFO
WARNING
CRITICAL
```

---

# 6. INFO

Used for non-problematic observations.

Examples:

```text
No new records received
Optional field missing
Very small source batch
```

INFO checks never block publication.

---

# 7. WARNING

Used when data is usable but attention may be required.

Examples:

- missing customer industry
- unmatched non-critical shipment record
- unusual order value
- late source file
- unexpected but valid category value

Warnings should be logged and visible.

They normally do not block reporting.

---

# 8. CRITICAL

Used when the issue can materially affect reporting accuracy.

Examples:

- duplicate invoice IDs
- missing mandatory source column
- revenue reconciliation failure
- duplicated sales target key
- invalid fact grain
- source pipeline incomplete
- critical reference-data failure

Critical failures may prevent mart publication.

---

# 9. Blocking vs Non-Blocking Checks

Severity and publication behavior should be explicit.

Example:

| Check | Severity | Blocks Publication |
|---|---|---|
| Duplicate invoice ID | Critical | Yes |
| Missing customer industry | Warning | No |
| Shipment references unknown order | Warning / Critical depending volume | Usually No |
| Revenue reconciliation mismatch | Critical | Yes |
| Dashboard data stale > SLA | Critical | Yes / Flag |
| Missing optional product brand | Info | No |

---

# 10. Data Quality Result Storage

Quality results should be stored centrally.

Recommended table:

```text
mis_control.data_quality_results
```

Suggested fields:

```text
check_id
pipeline_run_id
check_name
quality_dimension
severity

dataset_name
table_name

records_checked
failed_records
failure_rate

status

threshold_value
actual_value

executed_at
details
```

Possible statuses:

```text
PASS
FAIL
WARNING
```

---

# 11. Example Quality Result

```text
check_name:
duplicate_invoice_id

quality_dimension:
UNIQUENESS

severity:
CRITICAL

table_name:
mis_staging.stg_invoices

records_checked:
4,821

failed_records:
0

status:
PASS
```

---

# 12. Quality Check Naming Convention

Recommended format:

```text
<layer>_<object>_<rule>
```

Examples:

```text
stg_invoices_invoice_id_unique
stg_payments_amount_positive
mart_fact_sales_invoice_line_unique
mart_fact_inventory_snapshot_grain_unique
mart_fact_shipments_delivery_date_valid
```

This makes checks easier to search and manage.

---

# 13. Source Availability Checks

## PostgreSQL

Check:

```text
Can database connection be established?
```

Failure:

```text
CRITICAL
```

because no valid extraction can occur.

---

## Excel / CSV

Check:

```text
Does expected file exist?
Can it be opened?
```

Severity depends on whether the file is expected for the current reporting period.

---

## REST API

Check:

```text
API reachable
authentication valid
response valid
```

Authentication failures are:

```text
CRITICAL
```

Temporary availability failures may be retried before being classified as failed.

---

# 14. PostgreSQL Structural Checks

For transactional tables, validate:

- source table exists
- primary key exists
- `updated_at` exists for incremental tables
- required columns present
- extract query executes successfully

Example critical requirement:

```text
sales_orders.updated_at
```

must exist because incremental ingestion depends on it.

---

# 15. Staging Uniqueness Checks

## Customers

```text
customer_id
```

must be unique.

---

## Products

```text
product_id
sku
```

should both be unique.

---

## Sales Orders

```text
order_id
order_number
```

must be unique.

---

## Invoice Lines

```text
invoice_line_id
```

must be unique.

---

## Shipments

```text
shipment_id
```

must be unique.

Repeated API extraction should update existing staging records rather than create duplicates.

---

# 16. File Business-Key Checks

## Sales Targets

Business key:

```text
month
+
sales_rep_code
```

Duplicate example:

```text
2026-10 | SR001 | 2,500,000
2026-10 | SR001 | 2,600,000
```

This is critical because the pipeline cannot determine which target is authoritative.

---

## Branch Budget

Business key:

```text
month
+
branch_code
```

Duplicate key:

```text
CRITICAL
```

---

## Operating Expenses

Expected key:

```text
month
+
branch_code
+
expense_category
```

Duplicates may be valid only if the source explicitly supports multiple expense entries.

For this project, the file should contain one aggregated row per key.

---

# 17. Null Checks

Not every null should be treated equally.

## Critical Mandatory Fields

Examples:

```text
invoice_id
invoice_date
customer_id
invoice_amount

order_id
order_date

product_id
sku

payment_id
payment_date
payment_amount
```

Missing values:

```text
CRITICAL
```

---

## Optional Fields

Examples:

```text
customer.industry
product.brand
product.subcategory
```

Missing values may generate:

```text
WARNING
```

or no issue depending on business importance.

---

# 18. Numeric Validation

Examples:

## Invoice Amount

```text
invoice_amount >= 0
```

## Payment Amount

```text
payment_amount > 0
```

## Product Cost

```text
unit_cost >= 0
```

## Inventory Quantity

```text
quantity_on_hand >= 0
quantity_reserved >= 0
quantity_available >= 0
```

---

# 19. Inventory Consistency

Expected relationship:

```text
quantity_available
=
quantity_on_hand
-
quantity_reserved
```

A tolerance should be allowed only if fractional quantities or system behavior justify it.

For the project:

```text
difference = 0
```

is the expected rule.

---

# 20. Date Validation

Examples:

## Invoice

```text
due_date >= invoice_date
```

## Shipment

```text
expected_delivery_date >= dispatch_date
```

where business logic requires.

For delivered shipments:

```text
actual_delivery_date >= dispatch_date
```

---

# 21. Order Date Validation

Order dates should fall within reasonable business history.

Example:

```text
order_date >= 2024-01-01
```

for the generated portfolio dataset.

Future order dates beyond allowed business behavior should be flagged.

---

# 22. Referential Checks — Orders

Validate:

```text
sales_orders.customer_id
→ stg_customers.customer_id
```

```text
sales_orders.sales_rep_id
→ stg_sales_representatives.sales_rep_id
```

```text
sales_orders.branch_id
→ stg_branches.branch_id
```

Unknown references in PostgreSQL should be unlikely due to foreign keys.

If they occur in staging, the ingestion process itself may have failed.

---

# 23. Referential Checks — Order Lines

Validate:

```text
sales_order_lines.order_id
→ sales_orders.order_id
```

and:

```text
sales_order_lines.product_id
→ products.product_id
```

Failure rate should normally be:

```text
0%
```

---

# 24. Referential Checks — Invoices

Validate:

```text
invoice.order_id
→ sales_orders.order_id
```

```text
invoice.customer_id
→ customers.customer_id
```

and:

```text
invoice_line.invoice_id
→ invoice.invoice_id
```

---

# 25. Referential Checks — Payments

Validate:

```text
payment.invoice_id
→ invoice.invoice_id
```

and:

```text
payment.customer_id
=
invoice.customer_id
```

The second check is a business-consistency validation.

---

# 26. Referential Checks — Logistics

Validate:

```text
shipment.order_id
→ known sales order
```

Because the logistics API is external, a small number of unmatched records may realistically occur.

Example threshold:

```text
unmatched rate <= 0.1%
```

can be:

```text
WARNING
```

Above threshold:

```text
CRITICAL
```

---

# 27. Status Validation

Values should belong to approved domains.

## Sales Orders

Allowed:

```text
DRAFT
CONFIRMED
PROCESSING
DISPATCHED
COMPLETED
CANCELLED
REJECTED
```

Unknown statuses:

```text
CRITICAL or WARNING
```

depending on whether they affect KPI logic.

---

# 28. Invoice Status

Allowed:

```text
POSTED
PARTIALLY_PAID
PAID
CANCELLED
VOID
```

Unknown invoice status should be treated as critical because revenue recognition depends on it.

---

# 29. Payment Status

Allowed:

```text
SUCCESS
PENDING
FAILED
REVERSED
```

Unknown payment status should be flagged before receivables are calculated.

---

# 30. Shipment Status

Expected:

```text
DISPATCHED
IN_TRANSIT
OUT_FOR_DELIVERY
DELAYED
DELIVERED
CANCELLED
```

New external statuses should be detected rather than silently mapped.

---

# 31. Order Total Validation

Order header totals should reconcile with order lines within expected logic.

Conceptually:

```text
SUM(line net amounts)
≈
order subtotal after discount logic
```

The exact formula depends on how tax and header discounts are modeled.

A tolerance may be needed for rounding.

Example:

```text
₹1
```

or another documented tolerance.

---

# 32. Invoice Total Validation

Validate:

```text
SUM(invoice_line.net_amount)
≈
invoice.subtotal_amount
```

before tax.

Also:

```text
subtotal_amount
+
tax_amount
≈
invoice_amount
```

subject to cancellation behavior.

---

# 33. Payment vs Invoice Validation

A successful payment should not normally result in:

```text
total successful payments
>
recognized invoice value
```

unless overpayments are explicitly supported.

For this project, overpayment can be treated as:

```text
CRITICAL
```

---

# 34. Receivable Balance Validation

Expected:

```text
outstanding_amount >= 0
```

Negative outstanding balances indicate:

- overpayment
- incorrect payment allocation
- transformation error

and should fail validation.

---

# 35. Receivable Aging Validation

For:

```text
days_overdue <= 0
```

expected bucket:

```text
NOT_DUE
```

For:

```text
1 to 30
```

expected:

```text
1_30
```

and so forth.

This ensures bucket logic matches the underlying calculation.

---

# 36. Gross Margin Validation

Basic rule:

```text
recognized_revenue
cost_of_goods_sold
gross_margin
```

must satisfy the documented formula.

Example:

```text
gross_margin
=
recognized_revenue
-
cost_of_goods_sold
-
margin_adjustment
```

within numeric tolerance.

---

# 37. Margin Percentage Validation

Expected:

```text
gross_margin_pct
=
gross_margin / recognized_revenue
```

where revenue is non-zero.

If revenue equals zero:

```text
gross_margin_pct
```

should be null rather than infinite or invalid.

---

# 38. Margin Outlier Checks

Values such as:

```text
gross_margin_pct < -100%
```

or:

```text
gross_margin_pct > 100%
```

should be flagged.

Not every unusual margin is necessarily invalid.

Therefore these may initially be:

```text
WARNING
```

rather than automatically critical.

---

# 39. Sales Target Validation

Check:

```text
revenue_target >= 0
```

and:

```text
sales_rep_code
```

maps to a known sales representative.

Unknown salesperson target:

```text
CRITICAL
```

because actual vs target reporting cannot be correctly generated.

---

# 40. Budget Validation

Check:

```text
revenue_budget >= 0
expense_budget >= 0
```

and branch codes map to known branches.

Missing branch mapping should be critical.

---

# 41. Margin Adjustment Validation

Checks include:

```text
SKU exists
effective month valid
adjustment type recognized
amount valid
```

Unknown SKU may be:

```text
WARNING
```

if the row is quarantined and excluded.

A large number of unknown SKUs should escalate to critical.

---

# 42. File Schema Validation

Before file ingestion, validate exact required fields.

Example sales-target file:

```text
month
sales_rep_code
branch_code
revenue_target
```

Missing:

```text
revenue_target
```

should cause:

```text
CRITICAL
```

and prevent file ingestion.

---

# 43. Unexpected File Columns

Additional columns should not automatically fail the pipeline.

Possible approach:

```text
required expected fields
→ enforce

extra fields
→ log
```

This avoids unnecessary brittleness while still detecting meaningful schema changes.

---

# 44. Data-Type Validation

Example spreadsheet:

```text
revenue_target = "₹25,00,000"
```

may be normalized successfully.

But:

```text
revenue_target = "twenty five lakh"
```

cannot be safely interpreted.

This should become a rejected row or file validation failure.

---

# 45. File Duplicate Detection

File checksum:

```text
SHA-256
```

should detect an identical file already loaded.

Expected behavior:

```text
same hash
→ NO_CHANGE
```

not duplicate ingestion.

---

# 46. API Schema Validation

Required shipment fields:

```text
shipment_id
order_id
dispatch_date
expected_delivery_date
status
updated_at
```

Optional:

```text
actual_delivery_date
```

for open shipments.

---

# 47. API Duplicate Handling

If the same `shipment_id` is returned repeatedly:

```text
latest valid updated_at
```

should win.

The staging layer should not accumulate duplicates.

---

# 48. API Staleness

If the API is reachable but returns no updates for an unusually long period, the source may still be stale.

Example:

```text
last shipment source update
> 8 hours ago
```

when normal activity is expected.

This should generate a freshness warning.

---

# 49. Freshness Framework

Each reporting domain should have an expected maximum age.

| Domain | SLA |
|---|---:|
| Sales / Revenue | 24 hours |
| Receivables | 24 hours |
| Inventory | 24 hours |
| Logistics | 6 hours |
| Targets | Latest approved version |
| Budget | Latest approved version |

---

# 50. Freshness Calculation

Conceptually:

```text
current_timestamp
-
last_successful_data_timestamp
```

Example:

```text
Invoices expected:
24 hours

Current age:
31 hours

Status:
STALE
```

---

# 51. Freshness vs Pipeline Success

A successful pipeline does not always mean fresh data.

Example:

```text
Pipeline:
SUCCESS

Rows:
0
```

If the source itself has not changed unexpectedly for three days, reporting may still be stale.

Freshness should therefore consider source data timestamps, not only pipeline execution timestamps.

---

# 52. Source-to-Staging Row Reconciliation

For each extraction window:

```text
source record count
=
loaded record count
```

for clean PostgreSQL ingestion.

Example:

```text
Source:
184 orders

Target batch:
184 orders

PASS
```

---

# 53. Source-to-Staging Value Reconciliation

For financial datasets, row counts are insufficient.

Example invoice batch:

```text
source count
source invoice sum
```

should be compared with:

```text
staging count
staging invoice sum
```

---

# 54. Revenue Reconciliation

Management Recognized Revenue must eventually reconcile:

```text
PostgreSQL invoice / invoice lines
        ↓
staging
        ↓
transform
        ↓
fact_sales
```

Differences must be explained by documented rules such as:

- cancellations
- void invoices
- finance adjustments

---

# 55. Revenue Reconciliation Result

Possible metrics:

```text
source_valid_revenue
mart_recognized_revenue
difference
difference_pct
```

Tolerance:

```text
0
```

for exact synthetic financial data unless rounding creates an expected difference.

---

# 56. Payment Reconciliation

Compare:

```text
successful source payments
```

with:

```text
fact_payments.successful_payment_amount
```

for the same period.

Expected:

```text
exact match
```

---

# 57. Target Reconciliation

Compare:

```text
accepted Excel target total
```

with:

```text
SUM(fact_sales_target.revenue_target)
```

for the loaded version.

---

# 58. Budget Reconciliation

Compare:

```text
source budget file totals
```

to:

```text
fact_branch_budget
```

at:

```text
month
+
branch
```

grain.

---

# 59. Margin Adjustment Reconciliation

If product-level Finance adjustments are allocated to invoice lines:

```text
SUM(allocated adjustment)
```

must equal:

```text
source adjustment amount
```

for each:

```text
month
+
product
```

within tolerance.

---

# 60. Fact Grain Validation

Every fact table needs an explicit uniqueness test.

## `fact_sales`

```text
invoice_line_id
```

unique.

---

## `fact_orders`

```text
order_id
```

unique.

---

## `fact_payments`

```text
payment_id
```

unique.

---

## `fact_receivables_snapshot`

```text
snapshot_date
+
invoice_id
```

unique.

---

## `fact_inventory_snapshot`

```text
snapshot_date
+
warehouse_key
+
product_key
```

unique.

---

## `fact_sales_target`

```text
month
+
sales_rep_key
```

unique.

---

## `fact_shipments`

```text
shipment_id
```

unique.

---

# 61. Dimension Key Validation

Fact foreign keys should resolve to known dimension rows.

Example:

```text
fact_sales.customer_key
→ dim_customer.customer_key
```

Unknown member:

```text
-1
```

may be allowed temporarily if designed deliberately.

However, the number of unknown mappings should be measured.

---

# 62. Unknown Dimension Thresholds

Example:

```text
customer unknown mapping:
0 expected

product unknown mapping:
0 expected

shipment branch mapping:
<0.1% warning threshold
```

Any unexpected increase should be visible.

---

# 63. Volume Anomaly Checks

Record volume can reveal pipeline or source problems.

Example:

Normal daily invoices:

```text
150–250
```

Current run:

```text
7
```

The data may be valid, but the drop should be flagged.

---

# 64. Volume Threshold Approach

Initial project approach:

Compare current volume to recent baseline.

Example:

```text
current_count
vs
7-day average
```

Warning if:

```text
current < 50% of baseline
```

or:

```text
current > 200% of baseline
```

These should initially be warnings.

---

# 65. Business Exceptions vs Data Quality Failures

This distinction is important.

## Business Exception

Example:

```text
Customer invoice 120 days overdue
```

This is valid data describing a business problem.

---

## Data Quality Failure

Example:

```text
invoice due date missing
```

This prevents correct aging calculation.

Only the second is a data-quality problem.

---

# 66. Examples of Business Exceptions

These should appear in the dashboard but should not fail the pipeline:

- overdue invoice
- low-stock product
- delayed shipment
- low salesperson target achievement
- slow-moving inventory
- negative gross margin sale if commercially valid

---

# 67. Quarantine Strategy

Some invalid external records may be retained separately for investigation.

Possible table:

```text
mis_control.rejected_records
```

Suggested fields:

```text
rejection_id
pipeline_run_id
source_name
source_record_key
check_name
severity
reason
raw_record
rejected_at
```

---

# 68. When to Quarantine

Appropriate for:

- unknown SKU in CSV
- invalid spreadsheet row
- malformed API record
- unrecognized shipment order ID

Not appropriate for:

- completely missing mandatory file schema
- failed source connection
- corrupted entire API payload

Those should fail the pipeline.

---

# 69. Reject Row vs Reject Batch

## Reject Individual Row

Use when:

```text
small number of independent bad rows
```

and reporting can safely proceed without them.

---

## Reject Entire Batch

Use when:

- duplicate authoritative target keys
- required column missing
- financial extract incomplete
- major schema mismatch

because selective loading would create misleading reporting.

---

# 70. Quality Thresholds

Not all checks need zero tolerance.

Examples:

| Check | Threshold |
|---|---:|
| Duplicate invoice ID | 0 |
| Missing invoice ID | 0 |
| Unknown product in invoice line | 0 |
| Missing customer industry | <5% warning |
| Unknown shipment order | <0.1% warning |
| Revenue reconciliation | 0 difference |
| Inventory equation mismatch | 0 |
| Source file duplicate | 0 duplicate accepted |

Thresholds should be documented explicitly.

---

# 71. Threshold Configuration

Rather than hardcoding every threshold in SQL, some may be maintained through configuration.

Conceptually:

```text
check_name
threshold_type
threshold_value
severity
```

For the first implementation, SQL-defined thresholds are acceptable if documented.

---

# 72. Quality Execution Order

Recommended sequence:

```text
1. Source availability
2. Schema validation
3. Extraction
4. Staging structural checks
5. Referential checks
6. Business validation
7. Transformations
8. Fact grain checks
9. Reconciliation
10. Freshness
11. Publication
```

---

# 73. Publication Gate

Conceptually:

```text
All Critical Checks Passed?
        │
   ┌────┴────┐
   │         │
  Yes        No
   │         │
   ▼         ▼
Publish    Block affected
Mart       reporting output
```

Warnings remain visible but do not normally block.

---

# 74. Publication Status

A control table may track:

```text
reporting_domain
reporting_date
publication_status
published_at
blocking_check_count
warning_count
```

Possible statuses:

```text
READY
BLOCKED
STALE
```

---

# 75. Domain-Level Publication

A failure in one domain should not necessarily block everything.

Example:

```text
Logistics API failed
```

but:

```text
Sales
Finance
Inventory
```

are valid.

Possible behavior:

```text
Sales dashboard → READY
Receivables → READY
Inventory → READY
Logistics → STALE
```

This is more practical than making the entire platform unavailable.

---

# 76. Data Quality SQL Structure

Suggested repository structure:

```text
bigquery/
└── quality/
    ├── staging/
    │   ├── check_stg_customers.sql
    │   ├── check_stg_products.sql
    │   ├── check_stg_orders.sql
    │   ├── check_stg_invoices.sql
    │   └── check_stg_payments.sql
    │
    ├── marts/
    │   ├── check_fact_sales.sql
    │   ├── check_fact_receivables.sql
    │   ├── check_fact_inventory.sql
    │   └── check_fact_shipments.sql
    │
    └── reconciliation/
        ├── reconcile_revenue.sql
        ├── reconcile_payments.sql
        ├── reconcile_targets.sql
        └── reconcile_budget.sql
```

---

# 77. Reusable Check Pattern

Example concept:

```sql
SELECT
    COUNT(*) AS failed_records
FROM `mis_staging.stg_invoices`
WHERE invoice_id IS NULL;
```

The Python orchestration layer can execute the check and write the result into:

```text
mis_control.data_quality_results
```

---

# 78. Example Uniqueness Check

```sql
SELECT
    COUNT(*) AS failed_records
FROM (
    SELECT
        invoice_id
    FROM `mis_staging.stg_invoices`
    GROUP BY invoice_id
    HAVING COUNT(*) > 1
);
```

Expected:

```text
0
```

---

# 79. Example Referential Check

```sql
SELECT
    COUNT(*) AS failed_records
FROM `mis_staging.stg_invoice_lines` il
LEFT JOIN `mis_staging.stg_products` p
    ON il.product_id = p.product_id
WHERE p.product_id IS NULL;
```

Expected:

```text
0
```

---

# 80. Example Inventory Validation

```sql
SELECT
    COUNT(*) AS failed_records
FROM `mis_staging.stg_inventory_snapshot`
WHERE quantity_available
      != quantity_on_hand - quantity_reserved;
```

Expected:

```text
0
```

---

# 81. Example Revenue Reconciliation

Conceptually:

```sql
WITH source_total AS (
    SELECT
        SUM(valid_revenue) AS amount
    FROM ...
),

mart_total AS (
    SELECT
        SUM(recognized_revenue) AS amount
    FROM `mis_mart.fact_sales`
)

SELECT
    source_total.amount,
    mart_total.amount,
    source_total.amount - mart_total.amount AS difference;
```

---

# 82. Data Quality Runner

Python may provide a generic quality-check runner.

Conceptually:

```text
load check definition
       ↓
execute SQL
       ↓
read failed count / metric
       ↓
compare threshold
       ↓
write result
       ↓
raise blocking error if required
```

This keeps orchestration consistent.

---

# 83. Quality Check Definition

A check could conceptually contain:

```text
name
sql_file
severity
threshold
blocking
dataset
table
```

Example:

```text
name:
fact_sales_invoice_line_unique

severity:
CRITICAL

threshold:
0

blocking:
true
```

---

# 84. Automated Quality Summary

Each pipeline run should produce a summary.

Example:

```text
Pipeline:
postgres_invoices

Checks:
12

Passed:
11

Warnings:
1

Failed Critical:
0

Publication:
READY
```

---

# 85. Operational Data Quality Dashboard

A small internal monitoring page may later visualize:

- failed checks
- warnings
- failed records
- stale datasets
- reconciliation differences
- quality trend over time

This is separate from the management dashboard.

---

# 86. Suggested Monitoring KPIs

Operational quality metrics:

```text
Critical checks failed
Warnings active
Rejected rows
Unknown dimension mappings
Stale pipelines
Revenue reconciliation difference
Last successful load
```

---

# 87. Data Quality Trend

Quality should be monitored over time.

Example:

```text
October 1:
3 warnings

October 2:
5 warnings

October 3:
18 warnings
```

A sudden increase may indicate:

- source-system change
- file-format change
- integration failure

---

# 88. Controlled Test Cases

The synthetic environment should contain known scenarios.

## DQ-001 — Duplicate Sales Target

Source:

```text
Excel
```

Expected:

```text
CRITICAL
batch rejected
```

---

## DQ-002 — Unknown SKU Adjustment

Source:

```text
CSV
```

Expected:

```text
WARNING
row quarantined
```

---

## DQ-003 — Unknown Shipment Order

Source:

```text
API
```

Expected:

```text
WARNING
```

unless threshold exceeded.

---

## DQ-004 — Missing Customer Industry

Expected:

```text
WARNING
```

reporting continues.

---

## DQ-005 — Duplicate API Shipment

Expected:

```text
handled through MERGE
no duplicate staging record
```

---

## DQ-006 — Late Invoice Update

Expected:

```text
captured through updated_at watermark
not a quality failure
```

---

## DQ-007 — Invoice Reconciliation Mismatch

Expected:

```text
CRITICAL
fact_sales publication blocked
```

---

## DQ-008 — Stale Logistics Data

Expected:

```text
Logistics domain marked STALE
```

---

# 89. Testing Quality Rules

Each critical rule should have at least one failure-path test.

For example:

```text
Create duplicate target row
        ↓
Run quality checks
        ↓
Expected:
FAIL
```

Tests should verify both:

```text
PASS behavior
```

and:

```text
FAIL behavior
```

---

# 90. False Positives

Quality rules should not be so aggressive that valid business conditions trigger constant alerts.

For example:

```text
gross margin < 0
```

may be commercially unusual but still valid.

Therefore:

```text
negative margin
```

may be a business warning rather than a blocking data-quality check.

---

# 91. Quality Rule Ownership

In a real consulting engagement, different rules may have different owners.

Conceptually:

| Rule Type | Owner |
|---|---|
| Pipeline / schema | Data Engineering |
| Revenue reconciliation | Finance + Data |
| Sales target validity | Sales Operations |
| Budget validity | Finance |
| Logistics status mapping | Operations |
| KPI calculation | Analytics / Business Owner |

The portfolio project will document ownership even though one person implements the complete solution.

---

# 92. Auditability

For a reported KPI, we should be able to answer:

```text
Which source records contributed?

Which pipeline loaded them?

Which quality checks passed?

Which transformation created the measure?

Did it reconcile?
```

That is the practical meaning of auditability in this project.

---

# 93. Quality Logging Retention

Quality results should be retained for the full portfolio history.

This enables:

- incident analysis
- quality trend reporting
- demonstration of operational controls

Quality metadata volume will be small relative to business facts.

---

# 94. Incident Example

Suppose:

```text
Revenue dashboard
shows ₹12.7 Cr
```

but Finance expects:

```text
₹12.4 Cr
```

Investigation path:

```text
Dashboard KPI
      ↓
fact_sales
      ↓
Revenue reconciliation
      ↓
Transformation logic
      ↓
staging invoice lines
      ↓
source invoices
```

Quality and lineage metadata should make this investigation possible.

---

# 95. Data Quality Documentation

Each critical KPI should eventually document:

```text
source
business definition
quality rules
reconciliation rule
expected freshness
publication condition
```

This connects business KPI governance with technical data quality.

---

# 96. Initial Critical Checks

At minimum, the first production-like release should implement:

```text
invoice ID uniqueness
invoice-line ID uniqueness
payment ID uniqueness
shipment ID uniqueness

invoice mandatory fields
payment mandatory fields

order/customer reference validity
invoice/customer reference validity
invoice-line/product reference validity

invoice date logic
payment amount validity
inventory quantity equation
shipment delivery-date logic

sales target grain uniqueness

fact_sales grain uniqueness
fact_inventory_snapshot grain uniqueness
fact_receivables_snapshot grain uniqueness

source-to-staging row reconciliation
revenue reconciliation
payment reconciliation

data freshness
```

---

# 97. Quality Framework Definition of Done

The data-quality layer is complete when:

1. quality rules are stored and executed consistently
2. every critical fact has a uniqueness test
3. mandatory fields are validated
4. important reference relationships are checked
5. file schemas are validated
6. critical file duplicates are rejected
7. API records are validated
8. source-to-target counts are reconciled
9. financial values are reconciled
10. data freshness is measured
11. quality results are stored centrally
12. critical failures can block publication
13. warnings remain visible without unnecessarily blocking data
14. rejected external records can be investigated
15. controlled test cases behave as expected

---

# 98. Core Principle

The framework should avoid two extremes.

## Too Weak

```text
Pipeline completed
therefore data must be good
```

This is unsafe.

## Too Strict

```text
One optional field missing
therefore stop all reporting
```

This is impractical.

The desired approach is:

```text
validate what matters
+
measure exceptions
+
block only when trust is materially affected
```

---

# 99. Data Quality Outcome

The quality framework creates a controlled path from:

```text
data received
```

to:

```text
data trusted
```

The complete flow becomes:

```text
Source
   ↓
Ingest
   ↓
Validate
   ↓
Transform
   ↓
Reconcile
   ↓
Publish
   ↓
Monitor
```

This is a core part of the consulting solution because management reporting is valuable only when users trust the numbers.

---

## Next Step

The next document should be:

**`10-looker-studio-dashboard.md`**

It will define:

- dashboard audience
- page structure
- executive overview
- sales & profitability page
- receivables page
- inventory page
- logistics page
- KPI cards
- visuals
- filters
- drill-downs
- comparison periods
- dashboard interactions
- BigQuery serving views
- dashboard performance considerations
- screenshots required for the final portfolio case study