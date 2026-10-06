# 03 — Reporting & KPI Requirements

## Purpose

This document defines the management reporting requirements for the Multi-Source MIS & Management Reporting Automation project.

The objective is to ensure that the data platform is designed around agreed business questions and KPI definitions rather than simply replicating source-system data.

This document establishes:

- management reporting questions
- KPI definitions
- source-of-truth rules
- calculation logic
- reporting dimensions
- reporting periods
- dashboard requirements
- refresh expectations
- reconciliation requirements
- exception conditions

---

# 1. Reporting Objectives

The reporting solution should help management answer five broad questions.

## Sales

- How much revenue are we generating?
- Are we meeting our sales targets?
- Which branches and sales representatives are performing well?
- Which customers and products contribute the most revenue?

## Profitability

- Are sales generating acceptable margins?
- Which branches, customers, and products are most profitable?
- Where are margins declining?

## Receivables

- How much money is outstanding from customers?
- How much is overdue?
- Which customers have the largest overdue balances?
- How are collections trending?

## Inventory

- What stock is currently available?
- What is the value of inventory held?
- Which products are below reorder level?
- Which products appear to be slow-moving?

## Logistics

- Are shipments being delivered on time?
- Which shipments are delayed?
- How long does delivery typically take?
- Which carriers or regions have weaker delivery performance?

---

# 2. KPI Governance Principle

Every KPI shown in the management dashboard must have:

1. a defined business meaning
2. a documented formula
3. a designated source of truth
4. known exclusions
5. a refresh frequency
6. a reconciliation method where applicable

This prevents different teams from using different definitions for the same metric.

---

# 3. KPI Catalogue

The initial reporting scope will include the following KPI groups.

| KPI Group | KPI |
|---|---|
| Sales | Recognized Revenue |
| Sales | Booked Sales |
| Sales | Revenue Growth % |
| Sales | Sales Target Achievement % |
| Sales | Average Order Value |
| Profitability | Gross Margin |
| Profitability | Gross Margin % |
| Receivables | Outstanding Receivables |
| Receivables | Overdue Receivables |
| Receivables | Collection Rate % |
| Inventory | Inventory Value |
| Inventory | Low-Stock SKU Count |
| Inventory | Slow-Moving SKU Count |
| Logistics | On-Time Delivery % |
| Logistics | Average Delivery Time |
| Logistics | Open / Delayed Shipment Count |

---

# 4. Sales KPIs

## 4.1 Recognized Revenue

### Business Definition

Recognized Revenue represents invoiced sales that are valid for financial reporting.

It is the primary management revenue KPI.

### Source of Truth

```text
PostgreSQL ERP → invoices
```

### Formula

```text
Recognized Revenue
=
SUM(invoice_amount - cancelled_amount)
```

Only financially valid invoices should be included.

### Exclusions

Exclude:

- cancelled invoices
- void invoices
- test transactions
- invalid or rejected invoice records

### Reporting Dimensions

Revenue should be available by:

- date
- month
- quarter
- year
- branch
- region
- customer
- customer segment
- salesperson
- product category where allocation is possible

### Refresh

Daily.

### Reconciliation

Daily and monthly totals should reconcile to valid invoice totals in the source ERP.

---

## 4.2 Booked Sales

### Business Definition

Booked Sales represents the value of valid customer orders received during the reporting period.

This KPI measures commercial demand rather than accounting revenue.

### Source of Truth

```text
PostgreSQL ERP → sales_orders
```

### Formula

```text
Booked Sales
=
SUM(total_amount)
```

for orders with valid business statuses.

### Included Status Examples

```text
CONFIRMED
PROCESSING
DISPATCHED
COMPLETED
```

### Excluded Status Examples

```text
CANCELLED
REJECTED
DRAFT
```

### Business Use

Booked Sales helps management compare:

```text
customer demand
vs
financially recognized revenue
```

This distinction is intentional.

---

## 4.3 Revenue Growth %

### Business Definition

Measures percentage change in recognized revenue compared with a previous comparable period.

### Formula

```text
Revenue Growth %
=
(Current Period Revenue - Previous Period Revenue)
/
Previous Period Revenue
× 100
```

### Supported Comparisons

Initial dashboard:

- Month-over-Month
- Year-over-Year

### Example

```text
Current Month Revenue = ₹12.0 Cr
Previous Month Revenue = ₹10.0 Cr

Growth = 20%
```

### Exception Handling

If previous-period revenue is zero, the percentage should not be calculated as a normal numeric percentage.

---

# 5. Sales Target KPIs

## 5.1 Sales Target

### Source of Truth

```text
sales_targets_2026.xlsx
```

Targets are assigned primarily at salesperson/month level.

Example grain:

```text
month
sales_rep_code
revenue_target
```

---

## 5.2 Sales Target Achievement %

### Business Definition

Measures recognized revenue against the assigned revenue target.

### Formula

```text
Target Achievement %
=
Recognized Revenue
/
Revenue Target
× 100
```

### Example

```text
Recognized Revenue = ₹24 lakh
Target             = ₹20 lakh

Achievement = 120%
```

### Reporting Levels

- salesperson
- branch
- month
- year-to-date

Branch targets will be derived by aggregating salesperson targets unless a specific branch-level target is provided.

### Exception Handling

Records without a valid target should be classified separately rather than interpreted as 0% achievement.

---

# 6. Average Order Value

## 6.1 Definition

Measures the average value of valid customer orders.

### Formula

```text
Average Order Value
=
Booked Sales
/
Count of Valid Orders
```

### Source of Truth

```text
sales_orders
```

### Reporting Dimensions

- month
- branch
- customer segment
- salesperson

---

# 7. Profitability KPIs

## 7.1 Gross Margin

### Business Definition

Gross Margin measures revenue remaining after direct product cost and applicable margin adjustments.

### Initial Formula

```text
Gross Margin
=
Recognized Product Revenue
-
Cost of Goods Sold
-
Net Cost Adjustments
```

### Source Inputs

```text
Invoices / invoice allocation
Sales order lines
Products
Product margin adjustments
```

### Important Design Requirement

Revenue and cost must be compared at a compatible grain.

The transformation layer must avoid comparing:

```text
invoice-level revenue
```

directly with:

```text
order-line-level cost
```

without an allocation or mapping rule.

This logic will be finalized during data-model design.

---

## 7.2 Gross Margin %

### Formula

```text
Gross Margin %
=
Gross Margin
/
Recognized Revenue
× 100
```

### Reporting Dimensions

- month
- branch
- customer
- customer segment
- product
- product category
- salesperson where appropriate

### Exception Handling

If revenue is zero, Gross Margin % should return null rather than an invalid percentage.

---

# 8. Receivables KPIs

## 8.1 Outstanding Receivables

### Business Definition

Outstanding Receivables represents valid invoiced amounts that have not yet been collected.

### Formula

```text
Outstanding Receivables
=
Valid Invoice Amount
-
Successful Payments Applied
```

### Source of Truth

```text
invoices
+
payments
```

### Exclusions

Exclude:

- cancelled invoices
- void invoices
- failed payments
- reversed payments

---

## 8.2 Overdue Receivables

### Business Definition

Outstanding invoice value where the payment due date has passed.

### Formula

```text
Overdue Receivables
=
Outstanding Balance
where
due_date < reporting_date
```

### Reporting Dimensions

- customer
- customer segment
- branch
- salesperson
- aging bucket

---

## 8.3 Receivable Aging

Outstanding receivables will initially be classified into:

```text
Not Due
1–30 Days
31–60 Days
61–90 Days
90+ Days
```

Aging should be calculated using:

```text
reporting_date - due_date
```

for outstanding invoices.

---

## 8.4 Collection Rate %

### Business Definition

Measures cash collected relative to collectible invoice value during the selected reporting period.

### Initial Formula

```text
Collection Rate %
=
Payments Received
/
Collectible Invoice Amount
× 100
```

The exact period attribution rule will be documented during transformation design.

For example, reporting may compare:

```text
payments received during month
vs
invoice value becoming collectible during month
```

This KPI will therefore require careful validation before final dashboard publication.

---

# 9. Inventory KPIs

## 9.1 Inventory Value

### Business Definition

Estimated value of inventory currently held.

### Formula

```text
Inventory Value
=
Quantity On Hand
× Standard Cost
```

### Source Inputs

```text
inventory
+
products.standard_cost
```

### Reporting Dimensions

- warehouse
- branch
- product
- product category
- region

### Historical Reporting

Daily inventory snapshots will be retained in BigQuery to support historical inventory trends.

---

## 9.2 Available Inventory

### Formula

```text
Available Quantity
=
Quantity On Hand
-
Quantity Reserved
```

If the source already provides `quantity_available`, the transformation layer should validate it against this formula.

---

## 9.3 Low-Stock SKU Count

### Business Definition

Number of active products where available stock is at or below the defined reorder level.

### Formula

```text
Low Stock
=
Available Quantity <= Reorder Level
```

### KPI

```text
Low-Stock SKU Count
=
COUNT(DISTINCT product_id)
where low_stock = true
```

### Reporting Dimensions

- warehouse
- branch
- product category

---

## 9.4 Slow-Moving SKU Count

### Business Definition

Products with stock on hand but limited or no recent sales activity.

### Initial Rule

A product may be classified as slow-moving when:

```text
quantity_on_hand > 0
AND
no sales in previous 90 days
```

The threshold will be configurable.

### Initial Threshold

```text
90 days
```

### Future Enhancement

Additional classifications could include:

```text
No sale for 60 days
No sale for 90 days
No sale for 180 days
```

---

# 10. Logistics KPIs

## 10.1 On-Time Delivery %

### Business Definition

Percentage of delivered shipments that arrived on or before the expected delivery date.

### Source of Truth

```text
Logistics REST API
```

### Formula

```text
On-Time Delivery %
=
Shipments Delivered On Time
/
Total Delivered Shipments
× 100
```

### On-Time Condition

```text
actual_delivery_date
<=
expected_delivery_date
```

### Exclusions

Exclude:

- cancelled shipments
- records without a valid delivery date
- shipments still in transit

### Reporting Dimensions

- carrier
- branch
- destination city
- destination region
- month

---

## 10.2 Average Delivery Time

### Business Definition

Average elapsed time between dispatch and successful delivery.

### Formula

```text
Average Delivery Time
=
AVG(actual_delivery_date - dispatch_date)
```

### Unit

Initial dashboard:

```text
days
```

---

## 10.3 Open Shipment Count

### Definition

Number of shipments that have been dispatched but not yet delivered or cancelled.

Typical statuses may include:

```text
DISPATCHED
IN_TRANSIT
OUT_FOR_DELIVERY
```

---

## 10.4 Delayed Shipment Count

### Definition

Open shipments where the expected delivery date has already passed.

### Rule

```text
actual_delivery_date IS NULL
AND
expected_delivery_date < current_date
```

---

# 11. Reporting Dimensions

The data mart should support analysis across the following dimensions.

## Date

Required attributes:

```text
date
day
week
month
month_name
quarter
year
financial_month
financial_quarter
financial_year
```

The project will initially assume the Indian financial year:

```text
April → March
```

---

## Branch

Attributes:

```text
branch_id
branch_code
branch_name
city
state
region
```

---

## Customer

Attributes:

```text
customer_id
customer_code
customer_name
customer_segment
industry
city
state
region
```

---

## Product

Attributes:

```text
product_id
sku
product_name
category
subcategory
brand
unit_of_measure
```

---

## Sales Representative

Attributes:

```text
sales_rep_id
employee_code
sales_rep_name
branch
manager
status
```

---

## Warehouse

Attributes:

```text
warehouse_id
warehouse_code
warehouse_name
branch
city
state
```

---

## Carrier

Attributes:

```text
carrier_name
```

A dedicated carrier dimension may be introduced if carrier information becomes sufficiently complex.

---

# 12. Required Reporting Periods

Management should be able to analyse performance using:

- Today
- Yesterday
- Current Month
- Previous Month
- Month-to-Date
- Previous Month-to-Date
- Current Quarter
- Quarter-to-Date
- Current Financial Year
- Financial Year-to-Date
- Previous Financial Year
- Custom date range

Initial dashboard implementation may use a reduced subset where required.

---

# 13. Core Management Comparisons

The reporting layer should support several common management comparisons.

## Actual vs Target

```text
Recognized Revenue
vs
Sales Target
```

## Actual vs Budget

```text
Revenue
vs
Revenue Budget

Operating Expense
vs
Expense Budget
```

## Current vs Previous Period

Examples:

```text
Current Month Revenue
vs
Previous Month Revenue
```

```text
Current Month Margin %
vs
Previous Month Margin %
```

## Year-over-Year

Example:

```text
October 2026 Revenue
vs
October 2025 Revenue
```

---

# 14. Dashboard Requirements

The Looker Studio solution will initially contain five logical reporting pages.

## Page 1 — Executive Overview

Purpose:

Provide management with a high-level view of overall business performance.

### Suggested KPI Cards

- Recognized Revenue
- Revenue Growth %
- Sales Target Achievement %
- Gross Margin %
- Outstanding Receivables
- Overdue Receivables
- Inventory Value
- On-Time Delivery %

### Suggested Visuals

- monthly revenue trend
- actual vs target
- branch performance
- gross margin trend
- overdue receivables
- delivery performance

---

## Page 2 — Sales & Profitability

### Required Views

- revenue by branch
- revenue by salesperson
- revenue by customer
- revenue by product category
- sales target achievement
- gross margin by branch
- gross margin by customer
- gross margin by product category

### Drill-Down

Expected drill path:

```text
Branch
→ Salesperson
→ Customer
```

and:

```text
Category
→ Subcategory
→ Product
```

---

## Page 3 — Receivables

### Required Views

- outstanding receivables
- overdue receivables
- aging distribution
- largest overdue customers
- collections trend
- receivables by branch
- receivables by salesperson

---

## Page 4 — Inventory

### Required Views

- total inventory value
- inventory by warehouse
- inventory by product category
- low-stock products
- slow-moving products
- inventory-value trend

---

## Page 5 — Logistics

### Required Views

- shipments dispatched
- shipments delivered
- open shipments
- delayed shipments
- on-time delivery %
- average delivery time
- carrier performance
- delayed shipment details

---

# 15. Dashboard Filters

Common dashboard filters should include:

- reporting date range
- financial year
- branch
- region
- customer segment
- salesperson
- product category
- warehouse
- carrier

Not every filter needs to appear on every dashboard page.

The dashboard should avoid excessive controls that make the report difficult to use.

---

# 16. Reporting Grain

Different facts will operate at different grains.

| Reporting Area | Intended Grain |
|---|---|
| Sales Orders | Order |
| Product Sales | Order Line |
| Revenue | Invoice / allocated line |
| Payments | Payment |
| Inventory | Product + Warehouse + Snapshot Date |
| Sales Targets | Month + Salesperson |
| Branch Budget | Month + Branch |
| Expenses | Month + Branch + Expense Category |
| Logistics | Shipment |

These grains must remain explicit during data-model design to prevent double counting.

---

# 17. Source-of-Truth Matrix

Where multiple systems contain similar information, a single authoritative source must be identified.

| Business Metric | Authoritative Source |
|---|---|
| Recognized Revenue | ERP Invoices |
| Booked Sales | ERP Sales Orders |
| Product Quantity Sold | ERP Order Lines |
| Product Cost | ERP Product / Order Line Cost |
| Sales Target | Sales Target Excel |
| Revenue Budget | Branch Budget Excel |
| Operating Expense | Finance Expense Excel |
| Customer | ERP Customer Master |
| Product | ERP Product Master |
| Outstanding Receivable | ERP Invoices + Payments |
| Inventory Quantity | ERP Inventory |
| Shipment Status | Logistics API |
| Delivery Date | Logistics API |

This prevents one dashboard metric from switching between different sources depending on convenience.

---

# 18. KPI Reconciliation Requirements

Financial and operational KPIs must be reconcilable.

## Revenue Reconciliation

For each reporting date:

```text
Dashboard Recognized Revenue
=
Valid Revenue in BigQuery Data Mart
=
Valid ERP Invoice Revenue
```

Differences must be explainable.

---

## Order Reconciliation

Validate:

```text
Source Order Count
vs
Staging Order Count
```

and:

```text
Source Booked Sales
vs
Staging Booked Sales
```

---

## Payment Reconciliation

Validate:

```text
Source Successful Payment Amount
vs
Staging Successful Payment Amount
```

---

## Spreadsheet Reconciliation

For each ingested file:

```text
source row count
loaded row count
rejected row count
```

must be recorded.

---

# 19. Data Freshness Requirements

Initial target SLA:

| Reporting Domain | Maximum Expected Age |
|---|---|
| Sales / Revenue | 24 hours |
| Receivables | 24 hours |
| Inventory | 24 hours |
| Targets / Budget | Latest approved file |
| Logistics | 4–6 hours |

If the target freshness is breached, monitoring should flag the dataset as stale.

---

# 20. Data Quality Requirements

Dashboard publication should depend on critical data-quality checks.

Examples include:

### Sales

- no duplicate invoice IDs
- valid customer references
- valid invoice status
- invoice date present

### Receivables

- payment amount is non-negative
- payment references valid invoice
- cancelled invoice excluded

### Inventory

- no duplicate warehouse/product combination
- quantity values are valid
- products exist in master data

### Logistics

- shipment ID unique
- known order reference
- expected delivery date valid
- actual delivery date not before dispatch date

---

# 21. Exception Reporting

Not all data-quality problems should stop the complete pipeline.

Issues should be classified.

## Critical

Examples:

```text
Revenue reconciliation failure
Missing source table
Pipeline load failure
Major schema change
```

Critical issues may prevent publication of affected dashboard data.

## Warning

Examples:

```text
Unknown product category
Small number of unmatched shipment records
Missing optional customer attribute
```

Warnings should be logged but may not block reporting.

---

# 22. Reporting Security

The portfolio project uses synthetic data, but the design will assume normal enterprise access controls.

Management users should primarily access curated reporting outputs rather than raw source or staging datasets.

Conceptually:

```text
Raw / Staging
    → Data Engineering

Transformation
    → Data Engineering / Analytics

Data Mart
    → Analytics / Reporting

Dashboard
    → Management Users
```

Detailed IAM implementation is outside the initial project scope but may be documented later.

---

# 23. Out of Scope for Initial Release

To keep the solution practical, the following are not required in the first implementation:

- real-time streaming analytics
- machine-learning forecasting
- customer churn prediction
- demand forecasting
- automated pricing optimization
- advanced supply-chain optimization
- mobile application development
- enterprise master-data-management platform

These can be future enhancements but are not required to solve the current MIS problem.

---

# 24. Success Criteria

The initial reporting solution will be considered successful when:

1. PostgreSQL, spreadsheet, and API data are integrated automatically.
2. Recognized Revenue has a single documented definition.
3. Management KPIs are generated from the governed data mart.
4. Revenue and key financial totals reconcile with source data.
5. Data-quality failures are visible and traceable.
6. Dashboard information is refreshed within the agreed SLA.
7. Management can analyse performance without manually combining spreadsheets.
8. The reporting process requires less than one hour of routine manual intervention per reporting cycle.

---

# 25. Requirements Outcome

The management reporting requirement can now be summarized as:

```text
Trusted Sources
      ↓
Defined Business Rules
      ↓
Standardized KPIs
      ↓
Governed Data Mart
      ↓
Management Reporting
```

The project should not optimize for the largest possible technology stack.

It should optimize for:

```text
accuracy
+
consistency
+
maintainability
+
traceability
+
cost efficiency
```

---

## Next Step

The next document will be:

**`04-solution-architecture.md`**

It will translate these reporting requirements into the technical design, including:

- source connectivity
- ingestion patterns
- BigQuery dataset structure
- raw / staging / transformed / mart layers
- incremental loading
- orchestration
- data-quality execution
- monitoring
- logging
- failure handling
- cost controls
- environment and deployment approach