# 08 — BigQuery Layers & Management Data Mart

## Purpose

This document defines the BigQuery analytical architecture for the Multi-Source MIS & Management Reporting Automation project.

The objective is to transform operational source data from PostgreSQL, Excel/CSV, and the logistics API into a governed reporting model that can support:

- management KPIs
- historical analysis
- business reconciliation
- Looker Studio dashboards
- efficient analytical queries
- traceable reporting logic

The BigQuery platform is divided into four logical datasets:

```text
mis_control
mis_staging
mis_transform
mis_mart
```

Each layer has a distinct responsibility.

---

# 1. BigQuery Architecture

```text
                   SOURCE SYSTEMS

        PostgreSQL     Excel / CSV     REST API
             │              │              │
             └──────────────┼──────────────┘
                            │
                            ▼
                     Python Ingestion
                            │
                            ▼
                     mis_staging
                            │
                            ▼
                    mis_transform
                            │
                            ▼
                       mis_mart
                            │
                            ▼
                     Looker Studio


                     mis_control
                         ▲
                         │
             audit / DQ / reconciliation
```

---

# 2. Layer Responsibilities

## `mis_control`

Contains operational metadata.

Examples:

```text
pipeline_runs
pipeline_watermarks
file_load_history
data_quality_results
reconciliation_results
```

This dataset is not used for normal management reporting.

---

## `mis_staging`

Contains standardized source data with minimal business transformation.

Primary question:

> What did the source system provide?

This layer preserves source-level traceability.

---

## `mis_transform`

Contains cleansed, standardized, enriched, and business-ready intermediate models.

Primary question:

> How should source data be interpreted consistently?

This layer resolves:

- source-system inconsistencies
- status rules
- deduplication
- mappings
- derived attributes
- business calculations

---

## `mis_mart`

Contains reporting-oriented dimensions and facts.

Primary question:

> What structure does management reporting need?

Looker Studio should connect primarily to this dataset.

---

# 3. Why Separate the Layers

Combining everything directly into dashboard tables would make the platform difficult to troubleshoot.

The layered design provides:

```text
Source
  ↓
Staging
  ↓
Transformation
  ↓
Mart
  ↓
Dashboard
```

If a dashboard figure is incorrect, the investigation can follow the same path backwards.

This supports:

- lineage
- debugging
- reconciliation
- controlled business logic
- maintainability

---

# 4. Staging Layer

Dataset:

```text
mis_staging
```

The staging layer represents the latest accepted version of each source record.

Recommended tables:

```text
stg_branches
stg_warehouses
stg_sales_representatives
stg_customers
stg_products

stg_sales_orders
stg_sales_order_lines

stg_invoices
stg_invoice_lines
stg_payments

stg_inventory_snapshot

stg_sales_targets
stg_branch_budget
stg_operating_expenses
stg_margin_adjustments

stg_shipments
```

---

# 5. Staging Design Principles

Staging should perform only limited transformation.

Examples of acceptable staging operations:

```text
PostgreSQL timestamp
→ BigQuery TIMESTAMP

PostgreSQL NUMERIC
→ BigQuery NUMERIC

Excel amount text
→ BigQuery NUMERIC

API ISO timestamp
→ BigQuery TIMESTAMP
```

Business rules such as:

```text
gross margin
receivable aging
target achievement
delivery performance
```

should not be calculated in staging.

---

# 6. Staging Metadata

Most staging tables should include:

```text
_source_system
_source_table
_pipeline_run_id
_ingestion_timestamp
_source_updated_at
```

File tables may additionally contain:

```text
_source_file_name
_source_file_hash
_source_file_modified_at
```

API tables may contain:

```text
_api_extracted_at
```

---

# 7. Example `stg_invoices`

Conceptual schema:

| Column | Type |
|---|---|
| invoice_id | INT64 |
| invoice_number | STRING |
| order_id | INT64 |
| customer_id | INT64 |
| branch_id | INT64 |
| invoice_date | DATE |
| due_date | DATE |
| invoice_status | STRING |
| subtotal_amount | NUMERIC |
| tax_amount | NUMERIC |
| invoice_amount | NUMERIC |
| cancelled_amount | NUMERIC |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |
| _pipeline_run_id | STRING |
| _ingestion_timestamp | TIMESTAMP |
| _source_system | STRING |

---

# 8. Transformation Layer

Dataset:

```text
mis_transform
```

The transformation layer will contain cleaned and enriched business models.

Recommended models:

```text
tr_branch
tr_warehouse
tr_sales_rep
tr_customer
tr_product

tr_sales_order
tr_sales_order_line

tr_invoice
tr_invoice_line
tr_payment

tr_inventory_snapshot

tr_sales_target
tr_branch_budget
tr_operating_expense
tr_margin_adjustment

tr_shipment
```

---

# 9. Transformation Responsibilities

This layer should handle:

- deduplication
- status standardization
- valid-record filtering
- reference mapping
- date normalization
- business-rule flags
- calculated amounts
- source harmonization
- integration across sources

---

# 10. Example Invoice Transformation

`tr_invoice` may derive:

```text
valid_invoice_flag

recognized_invoice_amount

outstanding_eligible_flag

days_to_due
```

Example logic:

```text
recognized_invoice_amount
=
invoice_amount
-
cancelled_amount
```

for valid invoice statuses.

---

# 11. Example Invoice Line Transformation

`tr_invoice_line` may calculate:

```text
recognized_line_revenue
transaction_cost
base_gross_margin
```

Example:

```text
recognized_line_revenue
=
net_amount
```

```text
transaction_cost
=
quantity × unit_cost
```

```text
base_gross_margin
=
recognized_line_revenue
-
transaction_cost
```

Finance adjustments can be applied later.

---

# 12. Example Shipment Transformation

`tr_shipment` may derive:

```text
delivery_days
on_time_flag
delayed_flag
open_flag
```

Example:

```text
on_time_flag =
actual_delivery_date <= expected_delivery_date
```

for delivered shipments.

---

# 13. Example Receivable Transformation

Receivables require combining:

```text
invoices
+
payments
```

The transformation layer should first aggregate successful payments by invoice.

Example:

```text
payment_total_by_invoice
```

Then:

```text
outstanding_amount
=
recognized_invoice_amount
-
successful_payment_amount
```

---

# 14. Management Mart

Dataset:

```text
mis_mart
```

The mart will use a dimensional reporting model.

The design should favor:

- clear table grain
- easy joins
- reusable dimensions
- precomputed business measures
- efficient dashboard access

---

# 15. Proposed Dimensions

Core dimensions:

```text
dim_date
dim_branch
dim_warehouse
dim_sales_rep
dim_customer
dim_product
```

Optional later dimensions:

```text
dim_carrier
dim_expense_category
```

For the initial implementation, simple categorical attributes may remain inside fact tables unless a dedicated dimension adds real value.

---

# 16. Proposed Facts

Primary facts:

```text
fact_sales
fact_orders
fact_payments
fact_receivables_snapshot
fact_inventory_snapshot
fact_sales_target
fact_branch_budget
fact_operating_expense
fact_shipments
```

Not every source table becomes its own fact table.

Facts should represent meaningful business processes.

---

# 17. Dimensional Model Overview

```text
                         dim_date
                            │
                            │
dim_branch ────────┐        │        ┌──── dim_customer
                   │        │        │
                   ▼        ▼        ▼
                      fact_sales
                   ▲        ▲        ▲
                   │        │        │
          dim_sales_rep   dim_product
```

Other facts reuse the same dimensions where appropriate.

---

# 18. `dim_date`

`dim_date` will support both calendar and Indian financial-year reporting.

## Grain

One row per calendar date.

## Suggested Columns

```text
date_key
calendar_date

day_of_month
day_name
week_of_year

month_number
month_name
year

quarter_number
calendar_quarter

financial_month_number
financial_quarter
financial_year

month_start_date
month_end_date

is_month_end
is_weekend
```

---

# 19. Date Key

Possible surrogate key format:

```text
YYYYMMDD
```

Example:

```text
20261006
```

This provides a compact integer key while retaining `calendar_date` for normal SQL operations.

---

# 20. Indian Financial Year

Financial year:

```text
April
→
March
```

Example:

```text
2026-04-01
to
2027-03-31
```

can be labelled:

```text
FY2026-27
```

This should be calculated in `dim_date`.

---

# 21. `dim_branch`

## Grain

One row per branch version.

Initial attributes:

```text
branch_key
branch_id
branch_code
branch_name
city
state
region
status
```

`branch_key` is the warehouse surrogate key.

`branch_id` remains the PostgreSQL business/source identifier.

---

# 22. `dim_warehouse`

## Grain

One row per warehouse.

Attributes:

```text
warehouse_key
warehouse_id
warehouse_code
warehouse_name
branch_id
city
state
status
```

---

# 23. `dim_sales_rep`

Attributes:

```text
sales_rep_key
sales_rep_id
employee_code
sales_rep_name
branch_id
manager_id
joining_date
status
```

This dimension is used for:

- sales
- targets
- customer ownership
- receivables responsibility

---

# 24. `dim_customer`

Attributes:

```text
customer_key
customer_id
customer_code
customer_name
customer_segment
industry
city
state
region
credit_limit
payment_terms_days
sales_rep_id
status
```

---

# 25. `dim_product`

Attributes:

```text
product_key
product_id
sku
product_name
category
subcategory
brand
unit_of_measure
reorder_level
status
```

A key modelling decision:

```text
standard_cost
```

should not be used as the historical sale cost.

Transaction facts already contain the applicable unit cost.

---

# 26. Slowly Changing Dimensions

Not every changing attribute needs full history.

For the initial project, most dimensions will use:

```text
SCD Type 1
```

meaning the dimension reflects the latest known attribute value.

Examples:

```text
customer industry correction
product category correction
customer city correction
```

will overwrite previous dimension values.

---

# 27. Selected Historical Attributes

Some changes could matter historically.

Examples:

```text
customer assigned salesperson
salesperson branch
customer segment
```

A production system might use:

```text
SCD Type 2
```

for these.

However, full SCD Type 2 handling is not required in the first portfolio release.

This is a deliberate scope decision.

---

# 28. Future SCD Type 2 Structure

If introduced later, a dimension could include:

```text
effective_from
effective_to
is_current
```

Example:

```text
customer_key
customer_id
sales_rep_id
effective_from
effective_to
is_current
```

This could preserve historical sales ownership accurately.

---

# 29. `fact_sales`

This is one of the most important fact tables.

It represents financially recognized product sales.

## Grain

One row per:

```text
invoice line
```

This is intentionally different from one row per invoice.

Product-level revenue and margin require invoice-line grain.

---

# 30. `fact_sales` Suggested Columns

Keys:

```text
invoice_line_id
invoice_id
order_id
order_line_id

date_key
branch_key
customer_key
product_key
sales_rep_key
```

Degenerate dimensions:

```text
invoice_number
order_number
```

Measures:

```text
quantity

gross_line_amount
discount_amount
recognized_revenue

unit_cost
cost_of_goods_sold

margin_adjustment
gross_margin
gross_margin_pct
```

Operational attributes:

```text
invoice_status
```

---

# 31. Sales Fact Revenue

Primary revenue measure:

```text
recognized_revenue
```

derived from valid invoice-line revenue.

This becomes the basis for:

- total revenue
- customer revenue
- product revenue
- branch revenue
- salesperson revenue

---

# 32. Cost of Goods Sold

For an invoice line:

```text
cost_of_goods_sold
=
quantity
×
unit_cost
```

The transaction-level unit cost should be used.

---

# 33. Margin Adjustments

Finance adjustments may be matched using:

```text
effective_month
+
product
```

depending on adjustment type.

The transformation layer should aggregate applicable adjustments before joining them to sales.

Because an adjustment may apply to an entire product/month rather than an individual invoice line, allocation rules must be explicit.

---

# 34. Adjustment Allocation

If a product/month adjustment affects multiple invoice lines, it may be allocated proportionally using recognized revenue.

Example:

```text
line adjustment
=
monthly product adjustment
×
line recognized revenue
/
monthly product recognized revenue
```

This ensures:

```text
SUM(line adjustment)
=
source finance adjustment
```

The transformation should retain the original adjustment total for reconciliation.

---

# 35. Gross Margin

Final gross margin:

```text
gross_margin
=
recognized_revenue
-
cost_of_goods_sold
-
allocated_margin_adjustment
```

Depending on adjustment sign conventions, the exact expression will be documented during SQL implementation.

---

# 36. Gross Margin Percentage

```text
gross_margin_pct
=
SAFE_DIVIDE(
    gross_margin,
    recognized_revenue
)
```

The dashboard should not need to recreate this business formula repeatedly.

---

# 37. `fact_orders`

`fact_orders` represents commercial demand.

## Grain

One row per sales order.

Suggested fields:

```text
order_id
order_number

order_date_key
branch_key
customer_key
sales_rep_key

order_status

subtotal_amount
discount_amount
tax_amount
booked_sales_amount

order_line_count
```

This supports:

- booked sales
- average order value
- order counts
- order status analysis

---

# 38. Why Sales and Orders Are Separate

The project distinguishes:

```text
Booked Sales
```

from:

```text
Recognized Revenue
```

Therefore:

```text
fact_orders
```

and:

```text
fact_sales
```

serve different management questions.

This prevents order value from being incorrectly used as accounting revenue.

---

# 39. `fact_payments`

## Grain

One row per payment transaction.

Suggested fields:

```text
payment_id
invoice_id

payment_date_key
customer_key
branch_key

payment_method
payment_status

payment_amount
successful_payment_amount
```

This supports:

- collection trends
- payment-method analysis
- collection monitoring

---

# 40. `fact_receivables_snapshot`

Receivables differ from normal transactions because management often needs the balance as of a reporting date.

Therefore a periodic snapshot fact is appropriate.

## Grain

One row per:

```text
snapshot_date
+
invoice
```

for invoices with an outstanding balance.

---

# 41. Receivable Snapshot Columns

Keys:

```text
snapshot_date_key
invoice_id
customer_key
branch_key
sales_rep_key
```

Attributes:

```text
invoice_number
invoice_date
due_date
invoice_status
aging_bucket
```

Measures:

```text
invoice_value
payments_to_date
outstanding_amount
overdue_amount
days_overdue
```

---

# 42. Receivable Aging

Example transformation:

```text
days_overdue =
DATE_DIFF(
    snapshot_date,
    due_date,
    DAY
)
```

Buckets:

```text
NOT_DUE
1_30
31_60
61_90
90_PLUS
```

---

# 43. Why Use a Snapshot Fact

Without snapshots, today's invoice and payment data cannot accurately reproduce a historical question such as:

> What were outstanding receivables at the end of June?

Snapshots preserve the state that existed at each reporting period.

---

# 44. Receivable Snapshot Frequency

Initial implementation:

```text
daily
```

If cost or complexity becomes unnecessary, monthly-end snapshots could be retained longer while daily snapshots are limited to a rolling period.

For this portfolio scale, daily snapshots are reasonable.

---

# 45. `fact_inventory_snapshot`

## Grain

One row per:

```text
snapshot_date
+
warehouse
+
product
```

Suggested keys:

```text
snapshot_date_key
warehouse_key
branch_key
product_key
```

Measures:

```text
quantity_on_hand
quantity_reserved
quantity_available

standard_cost
inventory_value

reorder_level
low_stock_flag

days_since_last_sale
slow_moving_flag
```

---

# 46. Inventory Value

```text
inventory_value
=
quantity_on_hand
×
standard_cost
```

This is a point-in-time inventory valuation for reporting purposes.

---

# 47. Low Stock

```text
low_stock_flag
=
quantity_available <= reorder_level
```

This should be precomputed in the mart.

---

# 48. Slow-Moving Inventory

Initial rule:

```text
quantity_on_hand > 0
AND
days_since_last_sale > 90
```

The threshold should remain configurable.

---

# 49. Last Sale Calculation

The transformation layer can calculate:

```text
MAX(invoice_date)
```

for each product before each snapshot date.

This can be used to derive:

```text
days_since_last_sale
```

Care should be taken not to scan the entire sales history unnecessarily on every daily refresh.

---

# 50. `fact_sales_target`

## Grain

One row per:

```text
month
+
sales representative
```

Suggested fields:

```text
month_date_key
sales_rep_key
branch_key

revenue_target
```

Management target achievement is then calculated by comparing:

```text
fact_sales
```

against:

```text
fact_sales_target
```

at compatible monthly grain.

---

# 51. Target Achievement

A reporting model or aggregate view may expose:

```text
actual_revenue
revenue_target
target_achievement_pct
```

Formula:

```text
actual_revenue
/
revenue_target
```

The atomic fact tables should remain available beneath the aggregated reporting view.

---

# 52. `fact_branch_budget`

## Grain

One row per:

```text
month
+
branch
```

Measures:

```text
revenue_budget
expense_budget
```

Used for:

```text
actual vs budget
```

reporting.

---

# 53. `fact_operating_expense`

## Grain

One row per:

```text
month
+
branch
+
expense_category
```

Measures:

```text
expense_amount
```

This supports:

- branch operating cost
- expense trend
- budget comparison
- management profitability views

---

# 54. `fact_shipments`

## Grain

One row per shipment.

Suggested fields:

```text
shipment_id
order_id

dispatch_date_key
expected_delivery_date_key
actual_delivery_date_key

branch_key
customer_key

carrier_name
delivery_city
delivery_region

shipment_status

delivery_days
on_time_flag
delayed_flag
open_flag
```

---

# 55. Shipment Metrics

Examples:

```text
shipment_count = 1
```

```text
on_time_delivery_count =
IF(on_time_flag, 1, 0)
```

```text
delayed_shipment_count =
IF(delayed_flag, 1, 0)
```

Looker Studio can then aggregate these simple measures efficiently.

---

# 56. On-Time Delivery %

```text
SUM(on_time_delivery_count)
/
COUNTIF(shipment_status = 'DELIVERED')
```

A curated reporting view may expose this directly.

---

# 57. Fact Table Summary

| Fact | Grain | Primary Business Purpose |
|---|---|---|
| fact_sales | Invoice line | Revenue & gross margin |
| fact_orders | Sales order | Booked sales & order trends |
| fact_payments | Payment | Collections |
| fact_receivables_snapshot | Date + invoice | Outstanding & overdue balances |
| fact_inventory_snapshot | Date + warehouse + product | Inventory |
| fact_sales_target | Month + salesperson | Target performance |
| fact_branch_budget | Month + branch | Budget comparison |
| fact_operating_expense | Month + branch + category | Operating costs |
| fact_shipments | Shipment | Delivery performance |

---

# 58. Conformed Dimensions

Where possible, facts should reuse the same dimensions.

For example:

```text
dim_branch
```

should be reused by:

```text
fact_sales
fact_orders
fact_receivables_snapshot
fact_inventory_snapshot
fact_sales_target
fact_branch_budget
fact_operating_expense
fact_shipments
```

This is a conformed dimension.

It allows consistent branch filtering across multiple dashboard areas.

---

# 59. Surrogate Keys

The mart will use warehouse-generated surrogate keys such as:

```text
customer_key
product_key
branch_key
```

Source identifiers remain present as attributes.

Example:

```text
customer_key = 100238
customer_id  = 1844
```

This decouples analytical identities from source implementation details.

---

# 60. Surrogate Key Generation

For a Type 1 dimension, deterministic keys can be generated from source identifiers.

Possible approach:

```text
FARM_FINGERPRINT(
    CONCAT('ERP|CUSTOMER|', customer_id)
)
```

or sequential/deterministic analytical identifiers.

The implementation should prioritize stability.

---

# 61. Unknown Dimension Member

Facts may occasionally arrive before valid master-data mapping.

Instead of creating null foreign keys everywhere, dimensions may include:

```text
UNKNOWN
```

member.

Example:

```text
customer_key = -1
customer_name = 'Unknown Customer'
```

This can be useful for data-quality analysis.

However, critical reference failures should still be logged.

---

# 62. Reporting Views

Looker Studio does not necessarily need direct access to every fact table.

The mart may expose curated views such as:

```text
vw_executive_summary
vw_sales_performance
vw_receivables
vw_inventory
vw_logistics
```

These views can simplify dashboard development.

---

# 63. Executive Summary View

Example grain:

```text
date
+
branch
```

Potential measures:

```text
recognized_revenue
booked_sales
gross_margin
gross_margin_pct

sales_target
target_achievement_pct

outstanding_receivables
overdue_receivables

inventory_value

shipment_count
on_time_delivery_pct
```

Care must be taken when combining facts with different grains.

---

# 64. Avoiding Fact-to-Fact Join Errors

The platform should not directly join:

```text
fact_sales
```

to:

```text
fact_inventory_snapshot
```

at atomic grain.

Doing so could multiply rows.

Instead:

```text
aggregate each fact
to required reporting grain
```

before combining metrics.

---

# 65. Example

Incorrect:

```text
invoice lines
JOIN
inventory rows
```

directly by product.

This may create duplicate revenue.

Correct:

```text
aggregate sales
by date/product
        ↓

aggregate inventory
by date/product
        ↓

join aggregates
```

only when needed.

---

# 66. Dashboard Serving Models

For the dashboard, we may create purpose-built aggregate tables or views.

Examples:

```text
agg_daily_sales
agg_monthly_sales_rep
agg_monthly_branch_performance
agg_customer_receivables
agg_inventory_status
agg_carrier_performance
```

These are optional optimizations.

They should only be added if they simplify reporting or reduce BigQuery scanning.

---

# 67. Partitioning Strategy

Large fact tables should be partitioned by meaningful dates.

Recommended:

```text
fact_sales
PARTITION BY invoice_date
```

```text
fact_orders
PARTITION BY order_date
```

```text
fact_payments
PARTITION BY payment_date
```

```text
fact_receivables_snapshot
PARTITION BY snapshot_date
```

```text
fact_inventory_snapshot
PARTITION BY snapshot_date
```

```text
fact_shipments
PARTITION BY dispatch_date
```

---

# 68. Why Partition

Most management queries ask for a specific reporting period.

Partitioning allows BigQuery to scan only relevant partitions.

Example:

```sql
WHERE invoice_date >= '2026-09-01'
  AND invoice_date < '2026-10-01'
```

should avoid scanning unrelated years.

---

# 69. Clustering Strategy

Potential clustering columns include:

```text
branch_key
customer_key
product_key
sales_rep_key
```

For example:

```text
fact_sales
PARTITION BY invoice_date
CLUSTER BY branch_key, customer_key, product_key
```

Actual clustering should reflect frequent filter and join patterns.

---

# 70. Do Not Over-Cluster

Clustering should not be added mechanically to every table.

Small tables such as:

```text
dim_branch
fact_sales_target
```

may not benefit materially.

Cost optimization should follow query behavior.

---

# 71. Incremental Transformations

Large facts should not be rebuilt from complete history every day.

Example pattern:

```text
changed invoices
        ↓
identify affected invoice IDs
        ↓
recalculate affected fact_sales rows
        ↓
MERGE into fact_sales
```

---

# 72. Changed-Record Propagation

Because staging captures:

```text
_source_updated_at
```

the transformation layer can identify records updated since the last successful mart refresh.

For example:

```sql
WHERE _source_updated_at > @last_mart_watermark
```

---

# 73. Reprocessing Windows

Some calculations may depend on late-arriving changes.

For these models, it may be simpler to recompute a recent rolling period.

Example:

```text
rebuild previous 7 days
```

or:

```text
rebuild current month
```

rather than implementing complex row-level dependency tracking.

This is often a reasonable trade-off at this scale.

---

# 74. Receivables Refresh

Receivable balances can change whenever payments arrive.

The daily snapshot process should calculate balances as of the snapshot date.

Current open invoices can be recalculated daily.

Historical snapshot partitions should normally remain unchanged unless a correction requires reprocessing.

---

# 75. Inventory Refresh

Each daily inventory load creates one new snapshot partition.

Example:

```text
2026-10-06
```

Only that partition needs to be written during the normal daily run.

---

# 76. Sales Target Refresh

Sales-target files may be revised.

The mart should replace or merge affected:

```text
month
+
sales_rep
```

records when a newly approved file is loaded.

---

# 77. Data Quality in the Mart

The mart should not silently hide serious upstream issues.

Examples of critical checks:

```text
fact_sales invoice-line uniqueness

fact_orders order uniqueness

fact_inventory_snapshot grain uniqueness

fact_sales_target month/sales_rep uniqueness

fact_shipments shipment uniqueness
```

---

# 78. Fact Grain Tests

Examples:

```text
fact_sales
UNIQUE(invoice_line_id)
```

```text
fact_inventory_snapshot
UNIQUE(
    snapshot_date,
    warehouse_key,
    product_key
)
```

```text
fact_sales_target
UNIQUE(
    target_month,
    sales_rep_key
)
```

---

# 79. Reconciliation — Sales

Required reconciliation:

```text
SUM(mis_mart.fact_sales.recognized_revenue)
```

must reconcile to transformed valid invoice-line revenue.

It should also reconcile back to source invoice totals after known adjustments and exclusions.

---

# 80. Reconciliation — Payments

```text
SUM(successful_payment_amount)
```

should reconcile across:

```text
source
→ staging
→ transform
→ mart
```

---

# 81. Reconciliation — Finance Files

For example:

```text
SUM(fact_branch_budget.revenue_budget)
```

should equal the accepted source budget file total for the same period.

---

# 82. Reconciliation — Margin Adjustments

The allocated line-level adjustments must reconcile to the source Finance adjustment amount.

Example:

```text
SUM(allocated_adjustment)
=
source monthly product adjustment
```

within an accepted numeric tolerance.

---

# 83. Data Mart Security

Looker Studio should access:

```text
mis_mart
```

rather than:

```text
mis_staging
mis_transform
```

This creates a clean reporting boundary.

Conceptually:

```text
Engineering
→ all datasets

Reporting service
→ mis_mart read access

Management
→ Looker Studio only
```

---

# 84. BigQuery Naming Convention

Recommended conventions:

```text
stg_*
```

for staging.

```text
tr_*
```

for transformation.

```text
dim_*
fact_*
agg_*
vw_*
```

for mart assets.

Examples:

```text
stg_invoices
tr_invoice_line
dim_customer
fact_sales
agg_monthly_sales_rep
vw_executive_summary
```

---

# 85. SQL Repository Structure

Suggested:

```text
bigquery/
│
├── staging/
│   └── schemas/
│
├── transformations/
│   ├── tr_customer.sql
│   ├── tr_product.sql
│   ├── tr_invoice.sql
│   ├── tr_invoice_line.sql
│   ├── tr_payment.sql
│   └── tr_shipment.sql
│
├── marts/
│   ├── dimensions/
│   │   ├── dim_date.sql
│   │   ├── dim_branch.sql
│   │   ├── dim_customer.sql
│   │   ├── dim_product.sql
│   │   └── dim_sales_rep.sql
│   │
│   ├── facts/
│   │   ├── fact_sales.sql
│   │   ├── fact_orders.sql
│   │   ├── fact_payments.sql
│   │   ├── fact_receivables_snapshot.sql
│   │   ├── fact_inventory_snapshot.sql
│   │   ├── fact_sales_target.sql
│   │   ├── fact_branch_budget.sql
│   │   ├── fact_operating_expense.sql
│   │   └── fact_shipments.sql
│   │
│   └── views/
│       ├── vw_executive_summary.sql
│       ├── vw_sales_performance.sql
│       ├── vw_receivables.sql
│       ├── vw_inventory.sql
│       └── vw_logistics.sql
│
└── quality/
```

---

# 86. Reporting Model — Executive Dashboard

The Executive Overview should consume a small number of prepared models.

Suggested sources:

```text
vw_executive_summary
agg_monthly_branch_performance
agg_monthly_company_performance
```

This avoids having Looker Studio repeatedly join multiple large fact tables.

---

# 87. Reporting Model — Sales Dashboard

Primary sources:

```text
fact_sales
fact_orders
fact_sales_target
```

Potential aggregate:

```text
agg_monthly_sales_rep
```

with:

```text
month
sales_rep
branch

recognized_revenue
booked_sales
gross_margin
revenue_target
target_achievement_pct
```

---

# 88. Reporting Model — Receivables Dashboard

Primary source:

```text
fact_receivables_snapshot
```

This should already contain:

```text
outstanding_amount
overdue_amount
days_overdue
aging_bucket
```

so the dashboard does not need complicated invoice/payment logic.

---

# 89. Reporting Model — Inventory Dashboard

Primary source:

```text
fact_inventory_snapshot
```

including:

```text
inventory_value
low_stock_flag
slow_moving_flag
days_since_last_sale
```

---

# 90. Reporting Model — Logistics Dashboard

Primary source:

```text
fact_shipments
```

including:

```text
delivery_days
on_time_flag
delayed_flag
open_flag
```

---

# 91. Cost-Conscious Design

The mart should reduce BigQuery cost by:

- partitioning large facts
- using appropriate clustering
- precomputing repeated business calculations
- limiting dashboard access to curated tables
- avoiding `SELECT *`
- avoiding repeated full-history transformations
- using incremental refreshes
- creating aggregate models only where justified

---

# 92. Example Dashboard Query

Preferred:

```sql
SELECT
    month,
    branch_name,
    SUM(recognized_revenue) AS revenue,
    SUM(gross_margin) AS gross_margin
FROM `mis_mart.fact_sales`
WHERE invoice_date BETWEEN @start_date AND @end_date
GROUP BY
    month,
    branch_name;
```

with partition filtering.

Avoid building complex multi-source joins directly inside Looker Studio.

---

# 93. Materialized Views

Materialized views may be considered later for frequently repeated, expensive aggregations.

They are not required initially.

The first implementation should use normal tables or views unless performance data demonstrates a need.

---

# 94. Data Retention

For the portfolio dataset, three years of transactional history can be retained.

Potential strategy:

```text
staging
→ complete available history

mart facts
→ complete reporting history

daily snapshots
→ complete portfolio period
```

At real enterprise scale, snapshot retention policies may be more selective.

---

# 95. Schema Evolution

Adding a new source attribute should follow:

```text
source
↓
staging
↓
transformation
↓
mart
```

Only attributes required for business reporting should necessarily propagate to the mart.

This prevents the analytical model from becoming a copy of every source field.

---

# 96. Example End-to-End Lineage

For:

```text
Gross Margin %
```

lineage is:

```text
PostgreSQL invoice_lines
        +
Finance margin adjustments
        ↓
mis_staging.stg_invoice_lines
        +
mis_staging.stg_margin_adjustments
        ↓
mis_transform.tr_invoice_line
        ↓
mis_mart.fact_sales
        ↓
gross_margin
gross_margin_pct
        ↓
Looker Studio
```

---

# 97. Example Receivable Lineage

```text
PostgreSQL invoices
        +
PostgreSQL payments
        ↓
staging
        ↓
transformed invoice balance
        ↓
daily receivable snapshot
        ↓
mis_mart.fact_receivables_snapshot
        ↓
Looker Studio
```

---

# 98. Model Validation Questions

Before any mart table is accepted, we should be able to answer:

```text
What is the grain?

What is the business process?

What is the primary business key?

Which dimensions apply?

Which source owns each measure?

Can this table double count?

How does it refresh?

How is it reconciled?

How is it partitioned?

Who consumes it?
```

If those questions are unclear, the model is not ready.

---

# 99. Definition of Done — Data Mart

The BigQuery modelling layer is complete when:

1. all accepted source data exists in staging
2. transformations standardize source values
3. every fact has a documented grain
4. dimensions are reusable across reporting domains
5. recognized revenue comes from invoice lines
6. booked sales remains separate from recognized revenue
7. gross margin uses transaction-level cost
8. receivable snapshots support historical balance reporting
9. inventory snapshots support historical stock reporting
10. sales targets and budgets integrate cleanly
11. logistics metrics use standardized shipment logic
12. critical facts reconcile with sources
13. large facts are partitioned appropriately
14. Looker Studio can report from `mis_mart` without source-level joins

---

# 100. Final Mart Architecture

```text
                         DIMENSIONS

     dim_date       dim_branch       dim_customer
         │              │                 │
         │              │                 │
         └──────────┬───┴─────┬───────────┘
                    │         │
                    ▼         ▼
               fact_sales   fact_orders
                    │
          ┌─────────┼─────────────┐
          │         │             │
          ▼         ▼             ▼
   fact_payments  receivables  sales_target


dim_product ───── inventory_snapshot ───── dim_warehouse


dim_branch ───── branch_budget
     │
     └────────── operating_expense


sales_orders ───── fact_shipments
```

The mart provides one governed analytical layer across all management reporting domains.

---

# 101. Data Mart Outcome

The BigQuery architecture converts multiple operational sources into:

```text
consistent dimensions
+
clearly-grained fact tables
+
documented business measures
+
historical snapshots
+
reconciled financial metrics
```

This enables the reporting platform to answer management questions without repeatedly rebuilding business logic in spreadsheets or dashboards.

The most important design rule remains:

> The dashboard should consume trusted business-ready data, not reconstruct the data warehouse itself.

---

## Next Step

The next project document should be:

**`09-data-quality-framework.md`**

It will define:

- technical validation rules
- business validation rules
- referential checks
- uniqueness checks
- reconciliation
- freshness checks
- severity levels
- blocking vs non-blocking failures
- quality result storage
- failed-record handling
- monitoring metrics
- test scenarios
- data-quality reporting

After that, the reporting mart will have both the analytical structure and the control framework required before dashboard implementation.