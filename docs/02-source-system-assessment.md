# 02 — Source System Assessment

## Purpose

This document assesses the source systems that will feed the management reporting platform.

The objective is to identify:

- what data exists
- where it is stored
- who owns it
- how frequently it changes
- how it can be extracted
- expected data volumes
- business keys
- likely data-quality issues
- ingestion approach
- reporting relevance

The assessment will guide the design of the ingestion pipelines and BigQuery staging layer.

---

# 1. Source Landscape

The target reporting solution will integrate data from three primary source types:

1. PostgreSQL ERP
2. Excel / CSV files
3. External logistics REST API

At a high level:

```text
PostgreSQL ERP
     |
     |
Excel / CSV --------+
                    |
REST API -----------+
                    |
                    v
              Python Ingestion
                    |
                    v
              BigQuery Staging
```

Each source has different extraction, refresh, validation, and failure-handling requirements.

---

# 2. Source System Inventory

| Source | Type | Primary Owner | Data Domain | Expected Refresh |
|---|---|---|---|---|
| ERP Database | PostgreSQL | Operations / IT | Sales, Customers, Products, Inventory, Finance | Daily |
| Sales Targets | Excel | Sales Operations | Targets and Sales Planning | Monthly / As Updated |
| Branch Budget | Excel | Finance | Budget and Planning | Monthly / Annual |
| Operating Expenses | Excel | Finance | Expenses | Monthly |
| Margin Adjustments | CSV | Finance | Product Margin Adjustments | Monthly / As Required |
| Logistics Platform | REST API | Logistics Partner | Shipment and Delivery | Every Few Hours |

---

# 3. PostgreSQL ERP

## 3.1 Overview

The ERP database represents the main operational source system.

It contains transactional and master data used across sales, inventory, invoicing, and collections.

For the project, PostgreSQL will act as a realistic transactional source rather than an analytical database.

The reporting platform should avoid running unnecessarily heavy analytical queries directly against the ERP.

---

## 3.2 Expected Database Characteristics

| Attribute | Assumption |
|---|---|
| Database Engine | PostgreSQL |
| System Type | Transactional / OLTP |
| Approximate Data History | 3 Years |
| Active Customers | ~2,500 |
| Active Products | ~3,000 |
| Orders per Month | 4,000–6,000 |
| Order Lines per Order | 2–8 |
| Invoices per Month | 4,000–6,000 |
| Payments per Month | 3,000–5,000 |
| Inventory Snapshots | Daily |
| Primary Extraction Method | SQL via Python |
| Initial Load | Full historical extraction |
| Ongoing Load | Incremental |

---

# 4. ERP Table Assessment

## 4.1 `branches`

Stores branch master data.

### Expected Fields

```text
branch_id
branch_code
branch_name
city
state
region
opened_date
status
created_at
updated_at
```

### Expected Volume

Approximately:

```text
6–10 rows
```

### Business Key

```text
branch_id
```

### Change Pattern

Low-change master data.

### Reporting Usage

Used for:

- branch revenue
- branch profitability
- budget comparison
- inventory reporting
- regional reporting

### Likely Data-Quality Risks

- inconsistent branch naming
- inactive branches still referenced
- missing region mapping

### Ingestion Strategy

Full refresh is acceptable because of the very small dataset.

---

## 4.2 `warehouses`

Stores warehouse master information.

### Expected Fields

```text
warehouse_id
warehouse_code
warehouse_name
branch_id
city
state
status
created_at
updated_at
```

### Expected Volume

Approximately:

```text
3–6 rows
```

### Business Key

```text
warehouse_id
```

### Relationships

```text
warehouses.branch_id
    → branches.branch_id
```

### Reporting Usage

Used for:

- inventory by warehouse
- stock availability
- regional inventory analysis

### Ingestion Strategy

Full refresh.

---

## 4.3 `sales_representatives`

Stores sales employee information.

### Expected Fields

```text
sales_rep_id
employee_code
sales_rep_name
branch_id
manager_id
joining_date
status
created_at
updated_at
```

### Expected Volume

Approximately:

```text
45–60 rows
```

### Business Key

```text
sales_rep_id
```

### Reporting Usage

Used for:

- salesperson revenue
- target achievement
- customer ownership
- sales-team performance

### Data-Quality Risks

- reassignment between branches
- inactive sales representatives
- missing manager hierarchy

### Ingestion Strategy

Full refresh or incremental using `updated_at`.

---

# 5. Customer Master

## 5.1 `customers`

Stores customer master information.

### Expected Fields

```text
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
created_at
updated_at
```

### Expected Volume

Approximately:

```text
2,500–3,500 customers
```

### Business Key

```text
customer_id
```

Alternative natural key:

```text
customer_code
```

### Reporting Usage

Used for:

- customer revenue
- customer profitability
- receivables
- customer concentration
- segment analysis
- regional analysis

### Likely Data-Quality Risks

- duplicate customers
- inconsistent customer names
- missing customer segment
- missing region
- incorrect sales representative assignment
- inactive customers appearing in current reporting

### Ingestion Strategy

Initial full load followed by incremental ingestion using:

```text
updated_at
```

---

# 6. Product Master

## 6.1 `products`

Stores product and pricing master information.

### Expected Fields

```text
product_id
sku
product_name
category
subcategory
brand
unit_of_measure
standard_cost
list_price
reorder_level
status
created_at
updated_at
```

### Expected Volume

Approximately:

```text
3,000 products
```

### Business Key

```text
product_id
```

Natural key:

```text
sku
```

### Reporting Usage

Used for:

- product revenue
- category performance
- gross margin
- inventory analysis
- slow-moving inventory
- low-stock reporting

### Likely Data-Quality Risks

- duplicate SKUs
- missing category
- negative or zero cost
- inactive products in active inventory
- unexpected unit-of-measure changes

### Ingestion Strategy

Initial full load followed by incremental extraction using:

```text
updated_at
```

---

# 7. Sales Orders

## 7.1 `sales_orders`

Stores order-level transaction data.

### Expected Fields

```text
order_id
order_number
order_date
customer_id
sales_rep_id
branch_id
order_status
payment_terms_days
subtotal_amount
discount_amount
tax_amount
total_amount
created_at
updated_at
```

### Expected Volume

Approximately:

```text
4,000–6,000 orders per month
50,000–70,000 orders per year
```

For three years of history:

```text
150,000–200,000 orders
```

### Business Key

```text
order_id
```

Natural key:

```text
order_number
```

### Reporting Usage

Used for:

- booked sales
- order trends
- branch performance
- customer demand
- order pipeline analysis

### Data-Quality Risks

- cancelled orders included in sales totals
- duplicate order numbers
- invalid order dates
- missing customer references
- mismatched order totals

### Ingestion Strategy

Initial historical load followed by incremental ingestion.

Incremental extraction candidate:

```text
updated_at > last_successful_watermark
```

Using `updated_at` instead of only `order_date` is important because historical orders may change status after creation.

---

# 8. Order Lines

## 8.1 `sales_order_lines`

Stores product-level order details.

### Expected Fields

```text
order_line_id
order_id
product_id
quantity
unit_price
discount_amount
net_amount
unit_cost
created_at
updated_at
```

### Expected Volume

Average:

```text
2–8 lines per order
```

Estimated:

```text
20,000–35,000 rows per month
250,000–400,000 rows per year
```

Historical volume may approach:

```text
1 million rows
```

### Business Key

```text
order_line_id
```

### Relationships

```text
sales_order_lines.order_id
    → sales_orders.order_id

sales_order_lines.product_id
    → products.product_id
```

### Reporting Usage

Critical for:

- product revenue
- product margin
- product mix
- quantity sold
- customer-product analysis

### Data-Quality Risks

- zero or negative quantity
- zero selling price
- missing products
- incorrect unit cost
- line totals not matching calculated values

### Ingestion Strategy

Incremental using:

```text
updated_at
```

---

# 9. Invoices

## 9.1 `invoices`

Represents financially recognized sales.

This table will be particularly important because management revenue should ultimately be aligned with the agreed financial definition rather than simply order value.

### Expected Fields

```text
invoice_id
invoice_number
order_id
customer_id
invoice_date
due_date
invoice_status
subtotal_amount
tax_amount
invoice_amount
cancelled_amount
created_at
updated_at
```

### Expected Volume

Approximately:

```text
4,000–6,000 invoices per month
```

### Business Key

```text
invoice_id
```

Natural key:

```text
invoice_number
```

### Reporting Usage

Primary source for:

- recognized revenue
- receivables
- overdue invoices
- branch revenue
- customer revenue

### Data-Quality Risks

- duplicate invoice number
- cancelled invoices included in revenue
- due date before invoice date
- missing order reference
- invoice amount mismatch

### Ingestion Strategy

Incremental based on:

```text
updated_at
```

This captures later payment, cancellation, or adjustment-related changes.

---

# 10. Payments

## 10.1 `payments`

Stores receipts against customer invoices.

### Expected Fields

```text
payment_id
payment_reference
invoice_id
customer_id
payment_date
payment_amount
payment_method
payment_status
created_at
updated_at
```

### Expected Volume

Approximately:

```text
3,000–5,000 payments per month
```

### Business Key

```text
payment_id
```

### Reporting Usage

Used for:

- collected amount
- outstanding receivables
- customer payment behavior
- receivable aging

### Data-Quality Risks

- duplicate payment reference
- payments without matching invoice
- payment exceeding invoice value
- failed payments included as successful receipts

### Ingestion Strategy

Incremental load using:

```text
updated_at
```

---

# 11. Inventory

## 11.1 `inventory`

Represents the current inventory balance by warehouse and product.

### Expected Fields

```text
inventory_id
warehouse_id
product_id
quantity_on_hand
quantity_reserved
quantity_available
last_stock_update
updated_at
```

### Expected Volume

Potentially:

```text
3 warehouses × 3,000 products
≈ 9,000 active rows
```

### Business Key

Composite business key:

```text
warehouse_id + product_id
```

### Reporting Usage

Used for:

- available inventory
- stock value
- low-stock products
- stock by warehouse

### Important Design Consideration

A current-state inventory table alone cannot provide historical inventory trends.

Therefore the analytical platform may create periodic inventory snapshots.

For example:

```text
daily_inventory_snapshot
```

This allows historical analysis without modifying the ERP design.

### Ingestion Strategy

Full snapshot each day.

The BigQuery layer will preserve daily inventory history.

---

# 12. Excel / CSV Sources

Spreadsheet sources behave differently from transactional databases.

They are manually maintained and therefore require stronger validation before ingestion.

---

# 13. Sales Targets

## 13.1 `sales_targets_2026.xlsx`

Maintained by Sales Operations.

### Example Structure

| month | sales_rep_code | branch_code | revenue_target |
|---|---|---|---:|
| 2026-01 | SR001 | MUM | 2500000 |
| 2026-01 | SR002 | MUM | 2200000 |
| 2026-01 | SR003 | DEL | 2000000 |

### Expected Volume

Approximately:

```text
45 sales reps × 12 months
≈ 540 rows per year
```

### Business Key

```text
month + sales_rep_code
```

### Reporting Usage

Used for:

- sales target achievement
- salesperson performance
- branch target performance

### Data-Quality Risks

- duplicate targets
- invalid employee codes
- missing months
- numeric values stored as text
- changed spreadsheet column names

### Ingestion Strategy

Full-file ingestion whenever a new version is received.

File-level metadata should also be captured:

```text
source_file_name
file_modified_time
load_timestamp
```

---

# 14. Branch Budget

## 14.1 `branch_budget_2026.xlsx`

Maintained by Finance.

### Example Structure

| month | branch_code | revenue_budget | expense_budget |
|---|---|---:|---:|
| 2026-01 | MUM | 20000000 | 3500000 |
| 2026-01 | DEL | 18000000 | 3000000 |

### Business Key

```text
month + branch_code
```

### Reporting Usage

Used for:

- actual vs budget
- branch performance
- expense comparison

### Data-Quality Risks

- duplicate month/branch rows
- missing branch
- wrong financial period
- numbers stored with commas or symbols

### Ingestion Strategy

Full-file replacement with version tracking.

---

# 15. Operating Expenses

## 15.1 `monthly_operating_expenses.xlsx`

Maintained by Finance.

### Example Structure

| month | branch_code | expense_category | amount |
|---|---|---|---:|
| 2026-01 | MUM | Rent | 450000 |
| 2026-01 | MUM | Utilities | 125000 |
| 2026-01 | MUM | Travel | 185000 |

### Business Key

Potential composite key:

```text
month + branch_code + expense_category
```

### Reporting Usage

Used for:

- branch profitability
- operating-cost trends
- budget vs actual analysis

### Data-Quality Risks

- inconsistent category names
- duplicate rows
- missing branch mappings
- negative values
- inconsistent periods

### Ingestion Strategy

Full-file ingestion with validation and version metadata.

---

# 16. Product Margin Adjustments

## 16.1 `product_margin_adjustments.csv`

Used by Finance for exceptional product-level cost or margin adjustments that are not immediately reflected in the ERP.

### Example Structure

```text
effective_month,sku,adjustment_type,adjustment_amount,reason
2026-01,SKU1001,COST_ADJUSTMENT,2500,Freight correction
2026-01,SKU2042,REBATE,-1800,Supplier rebate
```

### Business Key

Potential composite key:

```text
effective_month + sku + adjustment_type
```

### Reporting Usage

Used in adjusted profitability calculations.

### Data-Quality Risks

- unknown SKU
- duplicate adjustment
- missing reason
- invalid sign
- inconsistent adjustment type

### Ingestion Strategy

Full-file ingestion.

---

# 17. Logistics REST API

## 17.1 Overview

Shipment and delivery data is provided by an external logistics service.

The API introduces a different integration pattern because the data must be fetched using HTTP rather than database queries or file processing.

---

## 17.2 Shipment Endpoint

Example:

```http
GET /shipments
```

Potential response:

```json
{
  "shipment_id": "SHP-100245",
  "order_id": "ORD-50512",
  "dispatch_date": "2026-01-12T10:15:00Z",
  "expected_delivery_date": "2026-01-15",
  "actual_delivery_date": "2026-01-14",
  "status": "DELIVERED",
  "carrier": "FastRoute Logistics",
  "delivery_city": "Pune"
}
```

---

## 17.3 Expected API Volume

Approximately:

```text
4,000–6,000 shipments per month
```

Historical API availability may be limited.

For the project, the API will support controlled synthetic historical records.

---

## 17.4 Business Key

```text
shipment_id
```

Order relationship:

```text
shipment.order_id
    → ERP sales_orders.order_number
```

---

## 17.5 Reporting Usage

Used for:

- delivery performance
- late shipments
- pending shipments
- dispatch-to-delivery time
- on-time delivery rate
- carrier performance

---

## 17.6 API Risks

Potential operational risks include:

- HTTP failures
- authentication errors
- API rate limits
- duplicate records
- pagination
- missing shipments
- delayed status updates
- schema changes
- temporary service unavailability

---

## 17.7 Ingestion Strategy

The API will be polled several times per day.

The pipeline should support:

- pagination
- retries
- timeout handling
- authentication
- incremental extraction
- duplicate prevention
- response logging

Possible incremental parameter:

```text
updated_since
```

Example:

```http
GET /shipments?updated_since=2026-01-15T06:00:00Z
```

If an API does not support incremental filtering, the pipeline can re-fetch a recent time window and perform idempotent upserts in the target.

---

# 18. Source-to-Reporting Mapping

| Reporting Area | Primary Source |
|---|---|
| Revenue | Invoices |
| Booked Sales | Sales Orders |
| Product Sales | Sales Order Lines |
| Sales Targets | Excel |
| Gross Margin | Order Lines + Products + Adjustments |
| Customer Analysis | Customers |
| Salesperson Performance | Sales Representatives + Targets |
| Branch Analysis | Branches |
| Receivables | Invoices + Payments |
| Inventory | Inventory + Products + Warehouses |
| Budget Comparison | Branch Budget Excel |
| Operating Expenses | Expense Excel |
| Delivery Performance | Logistics API |

---

# 19. Initial Load vs Incremental Load

## Initial Load

The first pipeline execution will load the available historical data.

Expected scope:

```text
3 years of ERP transactions
current master data
historical spreadsheet planning data
available logistics history
```

---

## Incremental Loads

After the initial load, the pipeline should process only newly created or changed records where possible.

Example pattern:

```sql
SELECT *
FROM sales_orders
WHERE updated_at > :last_successful_watermark
  AND updated_at <= :current_run_watermark;
```

The pipeline should store its processing watermark after a successful run.

---

# 20. Watermark Strategy

A control table can track the state of incremental pipelines.

Example:

```text
pipeline_name
source_name
last_successful_watermark
last_run_started_at
last_run_completed_at
last_run_status
rows_extracted
rows_loaded
```

Example record:

```text
postgres_sales_orders
erp_postgres
2026-01-15 06:00:00
2026-01-16 06:00:01
2026-01-16 06:02:41
SUCCESS
172
172
```

This will later form part of the monitoring framework.

---

# 21. Source Audit Metadata

Every staging dataset should include technical metadata where appropriate.

Recommended fields include:

```text
_source_system
_source_table
_ingestion_timestamp
_pipeline_run_id
_source_file_name
_source_updated_at
```

These fields improve traceability and troubleshooting.

Not every field applies to every source.

For example:

```text
_source_file_name
```

is relevant to Excel/CSV ingestion but not PostgreSQL tables.

---

# 22. Data Classification

The project will use synthetic data only.

However, the architecture should still reflect realistic data-governance principles.

Potential sensitive fields include:

- customer names
- customer addresses
- employee names
- sales values
- receivable balances

The reporting platform should avoid unnecessarily exposing sensitive operational attributes.

Only data required for reporting should be included in the analytical model.

---

# 23. Extraction Design Summary

| Source | Extraction Method | Load Type |
|---|---|---|
| Branches | PostgreSQL SQL | Full |
| Warehouses | PostgreSQL SQL | Full |
| Sales Representatives | PostgreSQL SQL | Full / Incremental |
| Customers | PostgreSQL SQL | Incremental |
| Products | PostgreSQL SQL | Incremental |
| Sales Orders | PostgreSQL SQL | Incremental |
| Sales Order Lines | PostgreSQL SQL | Incremental |
| Invoices | PostgreSQL SQL | Incremental |
| Payments | PostgreSQL SQL | Incremental |
| Inventory | PostgreSQL SQL | Daily Snapshot |
| Sales Targets | Excel Parser | Full File |
| Branch Budget | Excel Parser | Full File |
| Operating Expenses | Excel Parser | Full File |
| Margin Adjustments | CSV Parser | Full File |
| Shipments | REST API | Incremental |

---

# 24. Key Data-Quality Risks

Based on the source assessment, the highest-risk areas are expected to be:

### Referential Integrity

Examples:

```text
invoice → missing customer
order line → missing product
warehouse → missing branch
shipment → unknown order
```

### Duplicate Records

Particularly relevant for:

- spreadsheets
- API responses
- manual file replacements

### Financial Reconciliation

Revenue and receivable calculations must reconcile with source-system totals.

### Late-Arriving Data

Examples include:

- payments posted after invoice creation
- updated shipment status
- cancelled invoices
- revised Excel targets

### Master Data Changes

Customer ownership, product categories, and employee assignments may change over time.

Some of these changes may eventually require historical dimension handling.

---

# 25. Assumptions

The current assessment makes the following assumptions:

1. PostgreSQL provides reliable `created_at` and `updated_at` timestamps.
2. Source tables contain stable primary keys.
3. Spreadsheet formats are reasonably consistent but may contain data-quality issues.
4. The logistics API supports either updated timestamps or controlled re-fetching.
5. Management reporting does not require real-time data.
6. Daily ERP ingestion is sufficient for most reporting requirements.
7. Historical ERP data is available for approximately three years.
8. The portfolio environment uses entirely synthetic business data.

These assumptions will be validated as the technical implementation progresses.

---

# 26. Assessment Outcome

The source landscape supports a straightforward but realistic multi-source reporting architecture.

The key engineering patterns required are:

```text
PostgreSQL
    → incremental database ingestion

Excel / CSV
    → file validation and controlled loading

REST API
    → paginated API ingestion with retry handling

Inventory
    → periodic snapshots

All Sources
    → standardized BigQuery staging
    → validation
    → transformation
    → management data mart
```

No individual source presents unusual scale requirements.

The main challenge is therefore not volume, but reliable integration, consistent business definitions, data quality, and operational automation.

---

## Next Step

The next project document will define:

**`03-reporting-requirements.md`**

It will establish:

- management questions
- KPI definitions
- calculation rules
- reporting dimensions
- dashboard requirements
- refresh expectations
- reconciliation rules

This ensures that the data platform is designed around actual reporting requirements rather than simply moving source data into BigQuery.