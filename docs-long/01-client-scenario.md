# 01 — Client Scenario

## Client Overview

**Client:** Apex Industrial Components Pvt. Ltd.  
**Industry:** Industrial components distribution  
**Business Model:** B2B distribution to manufacturers, contractors, and maintenance companies  
**Primary Market:** India

### Business Profile

| Metric | Approximate Scale |
|---|---:|
| Annual Revenue | ₹120–150 Crore |
| Employees | ~220 |
| Branches | 6 |
| Warehouses | 3 |
| Sales Representatives | ~45 |
| Active Customers | ~2,500 |
| Products / SKUs | ~3,000 |
| Orders per Month | 4,000–6,000 |

Apex has grown into a multi-branch business, but its management reporting process is still largely dependent on manual exports and spreadsheets.

---

## Current Reporting Environment

Operational and management data is spread across several independent sources.

### PostgreSQL ERP

The core operational system stores:

- customers
- products
- sales orders
- order lines
- invoices
- payments
- inventory
- branches
- warehouses
- sales representatives

This system is the main source for transactional business data.

### Excel and CSV Files

Some important planning and finance information is maintained outside the ERP.

Typical files include:

```text
sales_targets_2026.xlsx
branch_budget_2026.xlsx
monthly_operating_expenses.xlsx
product_margin_adjustments.csv
```

These files are maintained by different business teams and shared with the MIS analyst periodically.

### Logistics Provider API

Shipment and delivery information is maintained by an external logistics provider.

Typical API resources include:

```text
GET /shipments
GET /shipments/{shipment_id}
GET /shipment-events
```

The API provides fields such as:

```text
shipment_id
order_id
dispatch_date
expected_delivery_date
actual_delivery_date
shipment_status
carrier
delivery_city
```

---

## Current MIS Process

Management reporting is prepared weekly and monthly.

The existing process is largely manual:

```text
PostgreSQL ERP
      |
      | SQL / CSV Export
      v
Operational Extracts ----+
                         |
Finance Files -----------+----> MIS Analyst
                         |
Sales Target Files ------+
                         |
Logistics Data ----------+
                              |
                              v
                         Data Cleaning
                              |
                         Excel Lookups
                              |
                         Reconciliation
                              |
                         Pivot Tables
                              |
                              v
                         MIS Workbook
                              |
                              v
                          Management
```

The analyst typically spends around **8–12 hours per reporting cycle** assembling, checking, and publishing the reports.

Month-end reporting can require additional effort due to reconciliation and corrections.

---

## Key Business Problems

### 1. High Manual Effort

The MIS process includes repetitive tasks such as:

- exporting data
- copying data between files
- correcting data formats
- applying spreadsheet formulas
- performing lookups
- refreshing pivot tables
- reconciling totals
- updating report layouts

A significant amount of analyst time is spent preparing data rather than analysing it.

---

### 2. Inconsistent KPI Definitions

Different teams sometimes calculate the same metric differently.

For example:

**Sales Revenue**

```text
Sales Team:
Order value

Finance Team:
Posted invoice value - cancellations - adjustments
```

This can result in different numbers being reported for the same reporting period.

The lack of standard KPI definitions reduces trust in the MIS.

---

### 3. Delayed Reporting

Most reporting is prepared weekly or monthly.

Management cannot easily obtain an up-to-date view of:

- revenue performance
- sales target achievement
- branch performance
- customer receivables
- product profitability
- inventory position
- delivery performance

Operational issues may therefore be identified only after the reporting cycle has completed.

---

### 4. Spreadsheet Risk

The reporting process is vulnerable to common spreadsheet issues such as:

- broken formulas
- incorrect lookup ranges
- duplicate records
- missing files
- incorrect filters
- inconsistent date formats
- manual entry errors
- multiple versions of the same report

These problems create additional reconciliation work and reduce confidence in reported numbers.

---

### 5. Fragmented Reporting History

Historical reports are stored as individual spreadsheet files.

For example:

```text
MIS_Jan.xlsx
MIS_Feb.xlsx
MIS_Mar_Final.xlsx
MIS_Mar_Final_v2.xlsx
MIS_Mar_Updated.xlsx
```

There is no centralized analytical history that can be queried consistently across reporting periods.

---

## Management Requirements

Management wants a more reliable and repeatable reporting process.

The future solution should provide visibility into areas such as:

### Sales

- total revenue
- monthly revenue trend
- sales by branch
- sales by salesperson
- sales target achievement
- customer contribution
- product-category performance

### Profitability

- gross sales
- cost of goods sold
- gross margin
- gross margin percentage
- profitability by branch
- profitability by customer
- profitability by product

### Receivables

- invoiced amount
- received amount
- outstanding balance
- overdue receivables
- receivable aging

### Inventory

- stock quantity
- stock value
- inventory by warehouse
- low-stock items
- slow-moving inventory

### Logistics

- dispatched shipments
- delivered shipments
- pending shipments
- late deliveries
- average delivery time
- on-time delivery percentage

---

## Consulting Engagement Objective

The objective of the engagement is to design and implement an automated management reporting solution that consolidates data from operational, financial, planning, and external systems.

The target solution will:

1. ingest data from PostgreSQL, files, and APIs
2. centralize reporting data in BigQuery
3. standardize business KPI definitions
4. create a management reporting data mart
5. implement automated data-quality checks
6. provide management dashboards in Looker Studio
7. automate refresh and pipeline execution
8. provide monitoring and operational documentation

---

## Target State

The target reporting process is:

```text
PostgreSQL ──────┐
                 │
Excel / CSV ─────┼──> Automated Ingestion
                 │
REST API ────────┘
                        |
                        v
                   BigQuery
                        |
                        v
               Validated Data Mart
                        |
                        v
                Looker Studio
                        |
                        v
                   Management
```

Manual work should shift from preparing reports to reviewing exceptions and analysing business performance.

---

## Target Business Outcomes

### Reduced Reporting Effort

Current:

```text
8–12 analyst hours per reporting cycle
```

Target:

```text
Less than 1 hour of manual intervention,
primarily for review and exception handling.
```

### Consistent Management KPIs

Important business metrics will have documented and reusable definitions.

### Improved Data Freshness

Management reporting will move from mainly weekly/monthly preparation toward automated daily refresh.

### Better Data Reliability

Data-quality checks will identify issues before they reach the dashboard.

### Centralized Historical Reporting

BigQuery will provide a consistent analytical history across reporting periods.

---

## Solution Principles

### Business-Driven Design

The data model will be designed around management questions and KPIs rather than simply copying source-system tables.

### Fit-for-Purpose Architecture

The use case does not require real-time streaming.

Batch and incremental processing will be preferred where appropriate.

### Cost Consciousness

The architecture should be practical for a mid-sized business and avoid unnecessary infrastructure.

### Data Quality by Design

Validation, reconciliation, and freshness checks will be part of the reporting pipeline.

### Maintainability

The solution should be understandable and supportable by a small data or analytics team.

### Traceability

Important management KPIs should be traceable from dashboard values back to the analytical model and source data.

---

## Scope of This Portfolio Engagement

This project will cover:

- client scenario and problem definition
- source-system assessment
- reporting and KPI requirements
- architecture design
- PostgreSQL source modelling
- realistic synthetic data generation
- Python ingestion pipelines
- BigQuery staging
- transformations
- management data mart
- data-quality controls
- Looker Studio reporting
- scheduling
- monitoring
- documentation
- final consulting case study

The company and data are fictional, but the solution is designed to reflect a realistic mid-sized enterprise MIS automation engagement.