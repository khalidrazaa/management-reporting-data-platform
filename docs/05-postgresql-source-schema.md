# 05 — PostgreSQL Source Schema

## Purpose

This document defines the PostgreSQL source schema for the fictional Apex Industrial Components ERP / order-management system.

The database represents an operational OLTP system used to run day-to-day business processes.

It is intentionally designed as a transactional source rather than an analytical model.

The analytical platform will extract data from this schema and transform it into reporting-friendly structures in BigQuery.

---

# 1. Design Principles

The PostgreSQL source database should reflect several realistic operational characteristics.

## Transaction-Oriented

The schema is optimized primarily for:

- order processing
- invoicing
- collections
- customer management
- product management
- inventory operations

It is not designed specifically for management reporting.

---

## Normalized Structure

Operational entities are stored separately to reduce duplication.

For example:

```text
sales_orders
     |
     +---- customers
     |
     +---- branches
     |
     +---- sales_representatives
```

Product-level details are stored separately:

```text
sales_orders
     |
     v
sales_order_lines
     |
     v
products
```

---

## Stable Keys

Each table should contain a stable surrogate primary key.

Examples:

```text
customer_id
product_id
order_id
invoice_id
```

Business-facing identifiers should also be unique where appropriate.

Examples:

```text
customer_code
sku
order_number
invoice_number
employee_code
```

---

## Change Tracking

Tables used for incremental ingestion should contain:

```text
created_at
updated_at
```

The `updated_at` column will be particularly important for BigQuery ingestion watermarks.

---

# 2. Schema Overview

The initial ERP schema contains the following tables:

```text
branches
warehouses
sales_representatives
customers
products

sales_orders
sales_order_lines

invoices
invoice_lines
payments

inventory
```

The tables can be grouped into three areas.

### Master Data

```text
branches
warehouses
sales_representatives
customers
products
```

### Transaction Data

```text
sales_orders
sales_order_lines
invoices
invoice_lines
payments
```

### Operational State

```text
inventory
```

---

# 3. Entity Relationships

High-level relationship:

```text
branches
   │
   ├──────────── warehouses
   │
   ├──────────── sales_representatives
   │
   ├──────────── customers
   │
   └──────────── sales_orders
                       │
                       ├──────── sales_order_lines
                       │                │
                       │                └──── products
                       │
                       └──────── invoices
                                   │
                                   ├──── invoice_lines
                                   │          │
                                   │          └──── products
                                   │
                                   └──── payments


warehouses
    │
    └──────── inventory
                  │
                  └──── products
```

---

# 4. Naming Conventions

The PostgreSQL schema will use:

- lowercase table names
- snake_case column names
- singular ID column names
- UTC timestamps where practical

Examples:

```text
sales_orders
sales_order_lines
customer_id
created_at
updated_at
```

---

# 5. `branches`

Stores company branch master data.

## Grain

One row per branch.

## Columns

| Column | PostgreSQL Type | Null | Description |
|---|---|---|---|
| branch_id | BIGSERIAL | No | Primary key |
| branch_code | VARCHAR(10) | No | Business branch code |
| branch_name | VARCHAR(100) | No | Branch name |
| city | VARCHAR(80) | No | City |
| state | VARCHAR(80) | No | State |
| region | VARCHAR(30) | No | Reporting region |
| opened_date | DATE | Yes | Branch opening date |
| status | VARCHAR(20) | No | ACTIVE / INACTIVE |
| created_at | TIMESTAMPTZ | No | Record creation timestamp |
| updated_at | TIMESTAMPTZ | No | Last modification timestamp |

## Constraints

```text
PRIMARY KEY (branch_id)

UNIQUE (branch_code)

status IN (
    'ACTIVE',
    'INACTIVE'
)
```

## Example Codes

```text
MUM
DEL
BLR
CHN
PUN
AHM
```

---

# 6. `warehouses`

Stores warehouse master data.

## Grain

One row per warehouse.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| warehouse_id | BIGSERIAL | No | Primary key |
| warehouse_code | VARCHAR(10) | No | Warehouse business code |
| warehouse_name | VARCHAR(100) | No | Warehouse name |
| branch_id | BIGINT | No | Owning branch |
| city | VARCHAR(80) | No | Warehouse city |
| state | VARCHAR(80) | No | State |
| status | VARCHAR(20) | No | ACTIVE / INACTIVE |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Relationships

```text
warehouses.branch_id
→ branches.branch_id
```

## Constraints

```text
UNIQUE (warehouse_code)

status IN (
    'ACTIVE',
    'INACTIVE'
)
```

---

# 7. `sales_representatives`

Stores sales employees.

## Grain

One row per salesperson.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| sales_rep_id | BIGSERIAL | No | Primary key |
| employee_code | VARCHAR(20) | No | Employee business identifier |
| sales_rep_name | VARCHAR(120) | No | Employee name |
| branch_id | BIGINT | No | Current branch |
| manager_id | BIGINT | Yes | Sales manager |
| joining_date | DATE | No | Joining date |
| status | VARCHAR(20) | No | ACTIVE / INACTIVE |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Relationships

```text
branch_id
→ branches.branch_id

manager_id
→ sales_representatives.sales_rep_id
```

`manager_id` is a self-referencing foreign key.

## Constraints

```text
UNIQUE (employee_code)
```

---

# 8. `customers`

Stores B2B customer master data.

## Grain

One row per customer account.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| customer_id | BIGSERIAL | No | Primary key |
| customer_code | VARCHAR(20) | No | Business customer code |
| customer_name | VARCHAR(150) | No | Customer name |
| customer_segment | VARCHAR(30) | No | Customer segment |
| industry | VARCHAR(80) | Yes | Industry classification |
| city | VARCHAR(80) | No | City |
| state | VARCHAR(80) | No | State |
| region | VARCHAR(30) | No | Sales region |
| credit_limit | NUMERIC(15,2) | No | Approved credit limit |
| payment_terms_days | INTEGER | No | Standard payment terms |
| sales_rep_id | BIGINT | No | Assigned salesperson |
| status | VARCHAR(20) | No | ACTIVE / INACTIVE / ON_HOLD |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Possible Segments

```text
ENTERPRISE
MID_MARKET
SME
DEALER
```

## Constraints

```text
UNIQUE (customer_code)

credit_limit >= 0

payment_terms_days >= 0
```

## Relationships

```text
sales_rep_id
→ sales_representatives.sales_rep_id
```

---

# 9. `products`

Stores product master information.

## Grain

One row per SKU.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| product_id | BIGSERIAL | No | Primary key |
| sku | VARCHAR(30) | No | Stock keeping unit |
| product_name | VARCHAR(150) | No | Product name |
| category | VARCHAR(80) | No | Product category |
| subcategory | VARCHAR(80) | Yes | Product subcategory |
| brand | VARCHAR(80) | Yes | Brand |
| unit_of_measure | VARCHAR(20) | No | EA / BOX / SET etc. |
| standard_cost | NUMERIC(15,2) | No | Current standard cost |
| list_price | NUMERIC(15,2) | No | Standard selling price |
| reorder_level | NUMERIC(12,2) | No | Reorder threshold |
| status | VARCHAR(20) | No | ACTIVE / INACTIVE |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Constraints

```text
UNIQUE (sku)

standard_cost >= 0

list_price >= 0

reorder_level >= 0
```

---

# 10. `sales_orders`

Stores customer sales orders.

## Grain

One row per sales order.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| order_id | BIGSERIAL | No | Primary key |
| order_number | VARCHAR(30) | No | Business order number |
| order_date | DATE | No | Order date |
| customer_id | BIGINT | No | Customer |
| sales_rep_id | BIGINT | No | Responsible salesperson |
| branch_id | BIGINT | No | Booking branch |
| order_status | VARCHAR(20) | No | Current order status |
| payment_terms_days | INTEGER | No | Terms copied at time of order |
| subtotal_amount | NUMERIC(15,2) | No | Value before tax |
| discount_amount | NUMERIC(15,2) | No | Order-level discount |
| tax_amount | NUMERIC(15,2) | No | Tax amount |
| total_amount | NUMERIC(15,2) | No | Final order value |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Status Values

```text
DRAFT
CONFIRMED
PROCESSING
DISPATCHED
COMPLETED
CANCELLED
REJECTED
```

## Relationships

```text
customer_id
→ customers.customer_id

sales_rep_id
→ sales_representatives.sales_rep_id

branch_id
→ branches.branch_id
```

## Constraints

```text
UNIQUE (order_number)

subtotal_amount >= 0

discount_amount >= 0

tax_amount >= 0

total_amount >= 0
```

---

# 11. `sales_order_lines`

Stores product-level order details.

## Grain

One row per product line in an order.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| order_line_id | BIGSERIAL | No | Primary key |
| order_id | BIGINT | No | Parent order |
| line_number | INTEGER | No | Line sequence |
| product_id | BIGINT | No | Product |
| quantity | NUMERIC(12,2) | No | Ordered quantity |
| unit_price | NUMERIC(15,2) | No | Selling price per unit |
| unit_cost | NUMERIC(15,2) | No | Cost captured at transaction time |
| discount_amount | NUMERIC(15,2) | No | Line discount |
| net_amount | NUMERIC(15,2) | No | Net line sales amount |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Relationships

```text
order_id
→ sales_orders.order_id

product_id
→ products.product_id
```

## Constraints

```text
UNIQUE (
    order_id,
    line_number
)

quantity > 0

unit_price >= 0

unit_cost >= 0

discount_amount >= 0

net_amount >= 0
```

---

# 12. Why `unit_cost` Exists on Order Lines

Although the product table contains:

```text
standard_cost
```

the order line also stores:

```text
unit_cost
```

This is deliberate.

Product standard cost may change later.

The transaction-level cost captures the cost basis applicable when the sale occurred.

This allows historical profitability analysis without incorrectly applying today's product cost to older transactions.

---

# 13. `invoices`

Stores invoice-header records.

## Grain

One row per invoice.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| invoice_id | BIGSERIAL | No | Primary key |
| invoice_number | VARCHAR(30) | No | Business invoice number |
| order_id | BIGINT | No | Related sales order |
| customer_id | BIGINT | No | Customer |
| branch_id | BIGINT | No | Invoice branch |
| invoice_date | DATE | No | Invoice date |
| due_date | DATE | No | Payment due date |
| invoice_status | VARCHAR(20) | No | Invoice status |
| subtotal_amount | NUMERIC(15,2) | No | Pre-tax amount |
| tax_amount | NUMERIC(15,2) | No | Tax |
| invoice_amount | NUMERIC(15,2) | No | Gross invoice value |
| cancelled_amount | NUMERIC(15,2) | No | Cancelled / voided value |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Status Values

```text
POSTED
PARTIALLY_PAID
PAID
CANCELLED
VOID
```

## Relationships

```text
order_id
→ sales_orders.order_id

customer_id
→ customers.customer_id

branch_id
→ branches.branch_id
```

## Constraints

```text
UNIQUE (invoice_number)

due_date >= invoice_date

invoice_amount >= 0

cancelled_amount >= 0
```

---

# 14. `invoice_lines`

Stores product-level invoice details.

This table is included because management reporting requires product-level recognized revenue and gross margin.

Without invoice-line data, recognized product revenue would need to be approximated from sales-order lines.

## Grain

One row per product line per invoice.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| invoice_line_id | BIGSERIAL | No | Primary key |
| invoice_id | BIGINT | No | Parent invoice |
| order_line_id | BIGINT | Yes | Original order line |
| line_number | INTEGER | No | Invoice-line sequence |
| product_id | BIGINT | No | Product |
| quantity | NUMERIC(12,2) | No | Invoiced quantity |
| unit_price | NUMERIC(15,2) | No | Selling price |
| unit_cost | NUMERIC(15,2) | No | Transaction cost |
| discount_amount | NUMERIC(15,2) | No | Line discount |
| net_amount | NUMERIC(15,2) | No | Recognized line revenue before tax |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Relationships

```text
invoice_id
→ invoices.invoice_id

order_line_id
→ sales_order_lines.order_line_id

product_id
→ products.product_id
```

## Constraints

```text
UNIQUE (
    invoice_id,
    line_number
)

quantity > 0

unit_price >= 0

unit_cost >= 0

net_amount >= 0
```

---

# 15. Why Invoice Lines Matter

Management revenue is defined using posted invoices rather than order value.

For product profitability, we therefore need:

```text
invoice-line revenue
-
invoice-line cost
```

rather than:

```text
order-line revenue
-
current product cost
```

This gives the reporting platform a more reliable basis for:

- product revenue
- category revenue
- product gross margin
- customer-product profitability

---

# 16. `payments`

Stores customer payments received against invoices.

For this portfolio scope, a payment applies to one invoice.

A more complex ERP might support one payment being allocated across multiple invoices through a separate allocation table, but that is intentionally outside the initial scope.

## Grain

One row per payment transaction.

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| payment_id | BIGSERIAL | No | Primary key |
| payment_reference | VARCHAR(40) | No | Business payment reference |
| invoice_id | BIGINT | No | Related invoice |
| customer_id | BIGINT | No | Customer |
| payment_date | DATE | No | Payment date |
| payment_amount | NUMERIC(15,2) | No | Amount received |
| payment_method | VARCHAR(20) | No | Payment method |
| payment_status | VARCHAR(20) | No | Payment status |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Payment Methods

Examples:

```text
BANK_TRANSFER
NEFT
RTGS
UPI
CHEQUE
```

## Payment Status

```text
SUCCESS
PENDING
FAILED
REVERSED
```

## Relationships

```text
invoice_id
→ invoices.invoice_id

customer_id
→ customers.customer_id
```

## Constraints

```text
UNIQUE (payment_reference)

payment_amount > 0
```

---

# 17. `inventory`

Stores current product inventory by warehouse.

This table represents the **current operational state** rather than historical stock levels.

Historical inventory will be created by daily snapshots in BigQuery.

## Grain

One row per:

```text
warehouse
+
product
```

## Columns

| Column | Type | Null | Description |
|---|---|---|---|
| inventory_id | BIGSERIAL | No | Primary key |
| warehouse_id | BIGINT | No | Warehouse |
| product_id | BIGINT | No | Product |
| quantity_on_hand | NUMERIC(12,2) | No | Physical stock |
| quantity_reserved | NUMERIC(12,2) | No | Reserved stock |
| quantity_available | NUMERIC(12,2) | No | Available stock |
| last_stock_update | TIMESTAMPTZ | No | Last operational stock update |
| created_at | TIMESTAMPTZ | No | Created timestamp |
| updated_at | TIMESTAMPTZ | No | Updated timestamp |

## Relationships

```text
warehouse_id
→ warehouses.warehouse_id

product_id
→ products.product_id
```

## Constraints

```text
UNIQUE (
    warehouse_id,
    product_id
)

quantity_on_hand >= 0

quantity_reserved >= 0

quantity_available >= 0
```

Expected relationship:

```text
quantity_available
=
quantity_on_hand
-
quantity_reserved
```

This will also be validated by the data-quality layer.

---

# 18. Table Summary

| Table | Approximate Volume | Change Pattern | BigQuery Load |
|---|---:|---|---|
| branches | 6 | Rare | Full |
| warehouses | 3 | Rare | Full |
| sales_representatives | ~45 | Low | Full / Incremental |
| customers | ~2,500 | Moderate | Incremental |
| products | ~3,000 | Moderate | Incremental |
| sales_orders | 150k–200k / 3 yrs | High | Incremental |
| sales_order_lines | 600k–1m / 3 yrs | High | Incremental |
| invoices | 150k–200k / 3 yrs | High | Incremental |
| invoice_lines | 500k–900k / 3 yrs | High | Incremental |
| payments | 100k–180k / 3 yrs | High | Incremental |
| inventory | ~9,000 current rows | Frequent | Daily Snapshot |

The synthetic implementation may initially use lower volumes during development and scale up for the final portfolio dataset.

---

# 19. Incremental Extraction Columns

Tables loaded incrementally should have an index on:

```text
updated_at
```

because the ingestion pattern will use queries such as:

```sql
SELECT
    ...
FROM sales_orders
WHERE updated_at > :watermark_from
  AND updated_at <= :watermark_to;
```

Tables requiring this pattern include:

```text
sales_representatives
customers
products
sales_orders
sales_order_lines
invoices
invoice_lines
payments
```

---

# 20. Index Strategy

Indexes should support operational application access and incremental extraction.

The project should avoid adding indexes purely because a reporting query needs them.

Analytics should primarily happen in BigQuery.

Recommended indexes include:

```text
branches(branch_code)

warehouses(warehouse_code)
warehouses(branch_id)

sales_representatives(employee_code)
sales_representatives(branch_id)

customers(customer_code)
customers(sales_rep_id)
customers(updated_at)

products(sku)
products(updated_at)

sales_orders(order_number)
sales_orders(customer_id)
sales_orders(order_date)
sales_orders(updated_at)

sales_order_lines(order_id)
sales_order_lines(product_id)
sales_order_lines(updated_at)

invoices(invoice_number)
invoices(order_id)
invoices(customer_id)
invoices(invoice_date)
invoices(updated_at)

invoice_lines(invoice_id)
invoice_lines(product_id)
invoice_lines(updated_at)

payments(invoice_id)
payments(customer_id)
payments(payment_date)
payments(updated_at)

inventory(warehouse_id, product_id)
inventory(updated_at)
```

---

# 21. Avoiding Over-Indexing

Because this is an OLTP source system, excessive indexes would increase the cost of:

```text
INSERT
UPDATE
DELETE
```

operations.

The objective is therefore:

```text
support operational access
+
support incremental extraction
```

rather than optimizing PostgreSQL for every analytical question.

---

# 22. Foreign-Key Relationships

The source system should enforce normal relational integrity.

Key relationships include:

```text
warehouses.branch_id
→ branches.branch_id

sales_representatives.branch_id
→ branches.branch_id

customers.sales_rep_id
→ sales_representatives.sales_rep_id

sales_orders.customer_id
→ customers.customer_id

sales_orders.sales_rep_id
→ sales_representatives.sales_rep_id

sales_orders.branch_id
→ branches.branch_id

sales_order_lines.order_id
→ sales_orders.order_id

sales_order_lines.product_id
→ products.product_id

invoices.order_id
→ sales_orders.order_id

invoices.customer_id
→ customers.customer_id

invoice_lines.invoice_id
→ invoices.invoice_id

invoice_lines.order_line_id
→ sales_order_lines.order_line_id

invoice_lines.product_id
→ products.product_id

payments.invoice_id
→ invoices.invoice_id

payments.customer_id
→ customers.customer_id

inventory.warehouse_id
→ warehouses.warehouse_id

inventory.product_id
→ products.product_id
```

---

# 23. Delete Behavior

Operational transactional records should generally not be physically deleted when referenced by downstream transactions.

For example:

```text
customer with orders
product with invoice lines
branch with invoices
```

should not be deleted casually.

Preferred business behavior is usually to change:

```text
status = 'INACTIVE'
```

rather than deleting the record.

Foreign keys should therefore normally use restrictive delete behavior.

---

# 24. Timestamp Strategy

Most tables should have:

```text
created_at
updated_at
```

using:

```text
TIMESTAMPTZ
```

Recommended defaults:

```text
created_at
DEFAULT CURRENT_TIMESTAMP

updated_at
DEFAULT CURRENT_TIMESTAMP
```

The application or a database trigger must update `updated_at` whenever a record changes.

---

# 25. Why `updated_at` Is Critical

Incremental ingestion depends on it.

Example:

```text
Invoice created:
2026-10-01 10:00

Invoice updated after payment:
2026-10-12 14:15
```

A pipeline based only on:

```text
invoice_date
```

could miss the later change.

A pipeline based on:

```text
updated_at
```

will extract the modified record again.

---

# 26. Monetary Data Types

Financial values should use:

```text
NUMERIC
```

rather than floating-point types.

Recommended pattern:

```text
NUMERIC(15,2)
```

Examples:

```text
invoice_amount
payment_amount
unit_price
unit_cost
credit_limit
```

This avoids floating-point precision problems in financial calculations.

---

# 27. Date vs Timestamp

Use `DATE` when the business meaning is calendar-based.

Examples:

```text
order_date
invoice_date
due_date
payment_date
```

Use `TIMESTAMPTZ` when exact event timing matters.

Examples:

```text
created_at
updated_at
last_stock_update
```

---

# 28. Transaction Status Philosophy

Status values should reflect operational lifecycle states.

For example:

```text
sales order

DRAFT
   ↓
CONFIRMED
   ↓
PROCESSING
   ↓
DISPATCHED
   ↓
COMPLETED
```

Alternative terminal states:

```text
CANCELLED
REJECTED
```

The analytics layer should not blindly include every status.

Business rules determine which statuses contribute to KPIs.

---

# 29. Source System vs Reporting Logic

The PostgreSQL database should store operational facts.

It should not contain analytical classifications simply for dashboard convenience.

For example, PostgreSQL stores:

```text
invoice_date
due_date
payment_amount
```

BigQuery calculates:

```text
outstanding_balance
days_overdue
aging_bucket
```

Similarly, PostgreSQL stores:

```text
dispatch and delivery information
```

from the external API separately rather than forcing logistics reporting logic into the ERP schema.

---

# 30. Revenue Source

The project distinguishes:

```text
Booked Sales
```

from:

```text
Recognized Revenue
```

Booked Sales comes from:

```text
sales_orders
sales_order_lines
```

Recognized Revenue comes from:

```text
invoices
invoice_lines
```

This allows management to compare:

```text
commercial demand
vs
financial realization
```

without mixing the two concepts.

---

# 31. Margin Calculation Basis

Product-level gross margin will primarily use:

```text
invoice_lines.net_amount
-
(invoice_lines.quantity × invoice_lines.unit_cost)
```

before applicable Finance margin adjustments.

Conceptually:

```text
Recognized Product Revenue
            -
      Transaction Cost
            -
    Margin Adjustments
            =
        Gross Margin
```

This calculation will be implemented in the BigQuery transformation layer rather than the source database.

---

# 32. Receivables Basis

Receivables will derive from:

```text
invoices
+
successful payments
```

Conceptually:

```text
Outstanding Balance
=
Valid Invoice Amount
-
Successful Payments Applied
```

The mart will calculate:

- outstanding amount
- overdue amount
- days overdue
- aging bucket

---

# 33. Inventory History

The PostgreSQL source contains only the current state:

```text
inventory
```

The ingestion pipeline will capture daily snapshots.

Example analytical history:

```text
snapshot_date | warehouse | product | quantity
------------------------------------------------
2026-10-05    | WH01      | SKU101  | 120
2026-10-06    | WH01      | SKU101  | 108
2026-10-07    | WH01      | SKU101  | 94
```

This is an example of the analytical platform adding historical capability without redesigning the operational application.

---

# 34. Deliberate Simplifications

The schema is designed to be realistic without becoming a complete ERP implementation.

The initial project will not include:

- purchase orders
- suppliers
- goods-receipt transactions
- detailed inventory movements
- customer returns
- credit notes
- GST accounting ledgers
- multiple currencies
- payment allocations across multiple invoices
- general ledger
- manufacturing
- procurement workflows

These features may exist in a real enterprise ERP but are not required to solve the reporting problem defined for this project.

---

# 35. Synthetic Data Requirements

The generated dataset should not look uniformly random.

It should contain realistic patterns such as:

- larger and smaller customers
- branch performance differences
- salesperson performance differences
- seasonal revenue variation
- popular and low-volume products
- partial payments
- overdue invoices
- cancelled orders
- delayed deliveries
- slow-moving inventory
- products reaching reorder levels

The synthetic data should create meaningful management insights when the dashboards are built.

---

# 36. Controlled Data-Quality Issues

A small number of deliberate issues may be introduced for testing the data-quality framework.

Examples:

```text
duplicate spreadsheet record
unknown SKU in margin-adjustment file
shipment referencing unknown order
missing optional customer classification
late-arriving invoice update
```

However, deliberately corrupting core PostgreSQL foreign-key relationships is unnecessary because the source database itself should enforce those constraints.

The most realistic quality problems should appear at integration boundaries.

---

# 37. Expected Source Data Timeline

The final portfolio dataset should contain approximately three years of history.

For example:

```text
2024
2025
2026
```

The exact synthetic period can be controlled through the data-generation configuration.

This allows the dashboard to demonstrate:

- monthly trends
- year-over-year comparison
- financial-year reporting
- customer history
- target achievement
- receivable aging

---

# 38. Source Schema Diagram

Conceptually:

```text
                         branches
                       /    |     \
                      /     |      \
                     ▼      ▼       ▼
              warehouses  sales_   sales_orders
                          reps          │
                           │            │
                           ▼            ▼
                       customers   sales_order_lines
                           │            │
                           │            ▼
                           │         products
                           │            ▲
                           ▼            │
                       invoices ── invoice_lines
                           │
                           ▼
                        payments


warehouses ───── inventory ───── products
```

---

# 39. Implementation Order

The PostgreSQL schema should be created in dependency order.

Recommended sequence:

```text
1. branches

2. warehouses

3. sales_representatives

4. customers

5. products

6. sales_orders

7. sales_order_lines

8. invoices

9. invoice_lines

10. payments

11. inventory
```

This makes foreign-key creation and synthetic data generation easier.

---

# 40. Repository Structure

The PostgreSQL implementation can be organized as:

```text
database/
│
├── schema/
│   ├── 001_branches.sql
│   ├── 002_warehouses.sql
│   ├── 003_sales_representatives.sql
│   ├── 004_customers.sql
│   ├── 005_products.sql
│   ├── 006_sales_orders.sql
│   ├── 007_sales_order_lines.sql
│   ├── 008_invoices.sql
│   ├── 009_invoice_lines.sql
│   ├── 010_payments.sql
│   ├── 011_inventory.sql
│   └── 012_indexes.sql
│
├── seed/
│
└── queries/
```

Alternatively, an initial combined schema file may be used during early development.

---

# 41. Schema Outcome

The PostgreSQL source now provides enough operational complexity to demonstrate:

```text
relational modelling
+
foreign keys
+
transactional data
+
incremental extraction
+
late updates
+
financial transactions
+
inventory snapshots
+
source-to-target reconciliation
```

without attempting to recreate an entire ERP product.

The source database exists to support the consulting scenario:

```text
Operational System
        ↓
Data Integration
        ↓
Analytical Platform
        ↓
Management Reporting
```

---

## Next Step

The next implementation step should be:

**PostgreSQL DDL and local source environment setup**

This will include:

- Dockerized PostgreSQL
- database initialization
- actual `CREATE TABLE` scripts
- foreign keys
- constraints
- indexes
- `.env.example`
- source database connection configuration

After the database is running and the schema is validated, the next major step will be:

**realistic synthetic data generation**

The data generator should create business behavior rather than simply populate tables with random values.