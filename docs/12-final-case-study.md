# 12 — Final Case Study

## Project Title

# Multi-Source MIS & Management Reporting Automation

A consulting-style end-to-end data solution for a fictional mid-sized B2B industrial distributor that currently prepares management MIS manually from PostgreSQL, Excel/CSV files, and an external logistics API.

---

# 1. Executive Summary

Apex Industrial Components Pvt. Ltd. relied on a highly manual reporting process to prepare weekly and monthly management MIS.

Operational data lived in PostgreSQL, Finance and Sales maintained separate Excel files, and delivery information came from an external logistics provider.

The MIS team manually:

- exported data
- cleaned files
- performed spreadsheet lookups
- reconciled totals
- refreshed pivot tables
- prepared management reports

This process consumed approximately:

```text
8–12 analyst hours
per reporting cycle
```

and created several risks:

- inconsistent KPI definitions
- spreadsheet errors
- delayed reporting
- multiple report versions
- limited historical analysis
- weak traceability
- dependency on individual analysts

The solution replaces this process with an automated management reporting platform built around:

```text
PostgreSQL
+
Excel / CSV
+
REST API
        ↓
Python Ingestion
        ↓
BigQuery
        ↓
Governed Data Mart
        ↓
Looker Studio
```

with data quality, reconciliation, scheduling, and monitoring built into the workflow.

---

# 2. Client Scenario

**Client:** Apex Industrial Components Pvt. Ltd.

**Industry:** Industrial components distribution

**Business Model:** B2B distribution to manufacturers, contractors, and maintenance companies.

Approximate scale:

| Metric | Scale |
|---|---:|
| Annual Revenue | ₹120–150 Crore |
| Employees | ~220 |
| Branches | 6 |
| Warehouses | 3 |
| Sales Representatives | ~45 |
| Active Customers | ~2,500 |
| Products / SKUs | ~3,000 |
| Orders per Month | 4,000–6,000 |

The company and datasets are fictional, but the business processes and architecture are designed to reflect a realistic mid-sized enterprise reporting environment.

---

# 3. Business Problem

The management team lacked a single, trusted source of reporting data.

The existing process looked approximately like:

```text
PostgreSQL ERP
      |
      | Export
      v
Operational Files ------+
                        |
Finance Excel ----------+----> MIS Analyst
                        |
Sales Targets ----------+
                        |
Logistics Data ---------+
                             |
                             v
                      Manual Cleaning
                             |
                      Excel Lookups
                             |
                       Reconciliation
                             |
                       Pivot Tables
                             |
                             v
                     Management MIS
```

Key problems included:

### High Manual Effort

Reporting required repetitive extraction, cleaning, and spreadsheet work.

### Inconsistent Metrics

Different teams sometimes used different definitions for the same KPI.

For example:

```text
Sales:
Order Value

Finance:
Posted Invoice Revenue
```

### Delayed Visibility

Management received weekly or monthly reporting rather than consistently refreshed information.

### Spreadsheet Risk

Manual reporting introduced risks such as:

- incorrect formulas
- duplicate data
- broken lookups
- missing files
- inconsistent versions

### Limited History

Older MIS reports existed as separate files rather than as a queryable analytical history.

---

# 4. Consulting Objective

The engagement objective was to design and implement a centralized management reporting solution that would:

1. integrate multiple source systems
2. standardize KPI definitions
3. automate data ingestion
4. centralize reporting data in BigQuery
5. create business-ready data marts
6. implement data-quality controls
7. provide management dashboards
8. automate refresh and monitoring
9. reduce manual reporting effort
10. improve reporting trust

---

# 5. Solution Architecture

The final target architecture is:

```text
                         SOURCE SYSTEMS
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
   PostgreSQL ERP        Excel / CSV        Logistics API
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                              ▼
                     Python Ingestion
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
         Extract           Validate          Audit
            │                 │                 │
            └─────────────────┼─────────────────┘
                              │
                              ▼
                       BigQuery Staging
                              │
                              ▼
                      Transformations
                              │
                              ▼
                    Management Data Mart
                              │
                              ▼
                      Looker Studio
                              │
                              ▼
                         Management
```

Supporting controls include:

```text
watermarks
data quality
reconciliation
pipeline logging
freshness monitoring
failure handling
```

---

# 6. Source Systems

The platform integrates three source patterns.

## PostgreSQL ERP

Contains:

- branches
- warehouses
- customers
- products
- sales representatives
- sales orders
- order lines
- invoices
- invoice lines
- payments
- inventory

Transactional tables are ingested incrementally using `updated_at`.

---

## Excel / CSV

Business-managed files include:

```text
sales_targets_2026.xlsx
branch_budget_2026.xlsx
monthly_operating_expenses.xlsx
product_margin_adjustments.csv
```

Files are validated before ingestion and tracked using metadata and file hashes.

---

## Logistics REST API

Shipment information includes:

- shipment ID
- order ID
- dispatch date
- expected delivery date
- actual delivery date
- status
- carrier
- destination

The API pipeline supports:

- pagination
- retries
- incremental extraction
- schema validation
- duplicate handling

---

# 7. Key Reporting Requirements

The reporting layer was designed around management questions rather than source tables.

Primary reporting domains:

```text
Sales
Profitability
Receivables
Inventory
Logistics
```

Representative KPIs include:

- Recognized Revenue
- Booked Sales
- Revenue Growth %
- Sales Target Achievement %
- Gross Margin
- Gross Margin %
- Outstanding Receivables
- Overdue Receivables
- Receivable Aging
- Inventory Value
- Low-Stock SKU Count
- Slow-Moving SKU Count
- On-Time Delivery %
- Average Delivery Time
- Delayed Shipment Count

---

# 8. KPI Governance

A key design decision was to establish authoritative sources for each KPI.

Examples:

| KPI | Source of Truth |
|---|---|
| Recognized Revenue | ERP Invoices / Invoice Lines |
| Booked Sales | ERP Sales Orders |
| Product Revenue | ERP Invoice Lines |
| Gross Margin | Invoice Lines + Finance Adjustments |
| Sales Target | Sales Target Excel |
| Outstanding Receivables | Invoices + Payments |
| Inventory | ERP Inventory |
| Delivery Performance | Logistics API |

This prevents different teams from calculating the same KPI differently.

---

# 9. Python Ingestion

Python provides a common ingestion framework across all source types.

The ingestion layer handles:

```text
connection
extraction
technical validation
load
watermarks
audit metadata
retries
logging
```

Business KPI calculations remain in BigQuery.

---

# 10. Incremental Loading

Transactional PostgreSQL pipelines use timestamp watermarks.

Example:

```sql
SELECT *
FROM sales_orders
WHERE updated_at > :watermark_from
  AND updated_at <= :watermark_to;
```

The watermark is advanced only after a successful load.

This prevents records from being missed after failed pipeline executions.

---

# 11. Idempotency

Pipelines are designed to be safely rerunnable.

Incremental records are loaded to a temporary batch and merged into BigQuery staging.

Conceptually:

```text
Extract Changes
      ↓
Load Batch
      ↓
MERGE
      ↓
Persistent Staging
```

This avoids duplicate business records when a pipeline is reprocessed.

---

# 12. BigQuery Architecture

The warehouse is divided into four logical datasets.

```text
mis_control
mis_staging
mis_transform
mis_mart
```

## `mis_control`

Operational metadata:

- pipeline runs
- watermarks
- file history
- data-quality results
- reconciliation results

## `mis_staging`

Source-oriented records with minimal transformation.

## `mis_transform`

Cleansed and standardized business data.

## `mis_mart`

Reporting dimensions, fact tables, and serving views.

---

# 13. Analytical Data Model

Core dimensions:

```text
dim_date
dim_branch
dim_warehouse
dim_sales_rep
dim_customer
dim_product
```

Core facts:

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

---

# 14. Important Modelling Decisions

## Revenue Uses Invoice Lines

Recognized Revenue is based on invoice transactions rather than order value.

This separates:

```text
Booked Sales
```

from:

```text
Recognized Revenue
```

---

## Transaction-Level Cost

Historical margin uses the transaction-level cost stored on invoice/order lines rather than current product master cost.

---

## Receivables Are Snapshots

Receivable balances are captured as:

```text
snapshot date
+
invoice
```

This supports historical questions such as:

> What was outstanding at the end of June?

---

## Inventory Is Also Snapshot-Based

Current ERP inventory is converted into daily analytical snapshots.

This makes historical stock trends possible without redesigning the operational ERP.

---

# 15. Data Quality Framework

Data quality is built into the reporting process.

Checks cover:

```text
availability
schema
completeness
uniqueness
referential integrity
business rules
reconciliation
freshness
```

---

# 16. Data Quality Examples

Examples include:

```text
duplicate invoice IDs
missing required fields
unknown product references
duplicate sales-target rows
invalid invoice dates
invalid payment amounts
inventory quantity mismatches
invalid shipment dates
```

Quality results are stored in:

```text
mis_control.data_quality_results
```

---

# 17. Reconciliation

Important financial metrics are reconciled across the platform.

Example:

```text
Source Invoice Revenue
       ↓
Staging
       ↓
Transformation
       ↓
fact_sales
       ↓
Dashboard
```

Unexplained differences can block publication of the affected reporting domain.

---

# 18. Business Exception vs Data Quality

The design distinguishes operational problems from corrupted data.

Examples:

```text
120-day overdue invoice
```

is a valid business exception.

```text
missing invoice due date
```

is a data-quality issue.

Likewise:

```text
late shipment
```

is business performance data.

```text
delivery date before dispatch date
```

is a data-quality problem.

---

# 19. Dashboard Design

The Looker Studio reporting solution contains five management pages.

```text
1. Executive Overview
2. Sales & Profitability
3. Receivables
4. Inventory
5. Logistics
```

---

# 20. Executive Overview

Primary metrics:

- Recognized Revenue
- Revenue Growth %
- Target Achievement %
- Gross Margin %
- Outstanding Receivables
- Overdue Receivables
- Inventory Value
- On-Time Delivery %

Supporting visuals include:

- revenue trend
- actual vs target
- branch performance
- receivable aging
- logistics performance

---

# 21. Sales & Profitability

Designed to answer:

```text
Which branches are performing?

Which salespeople are meeting target?

Which customers drive revenue?

Which products drive margin?
```

Key views include:

- branch performance
- salesperson performance
- top customers
- product-category performance
- customer concentration
- gross margin analysis

---

# 22. Receivables

Designed to monitor:

- outstanding value
- overdue value
- aging
- 90+ day exposure
- collections
- highest-risk customers

---

# 23. Inventory

Designed to monitor:

- inventory value
- stock by warehouse
- low-stock products
- slow-moving products
- inventory trends
- capital tied up in stock

---

# 24. Logistics

Designed to monitor:

- shipment volume
- open shipments
- delayed shipments
- on-time delivery %
- delivery time
- carrier performance
- delayed shipment details

---

# 25. Scheduling

The platform uses a lightweight scheduling approach.

Initial deployment:

```text
cron
+
Dockerized Python pipeline
```

Main PostgreSQL reporting pipelines run daily.

The logistics API runs approximately every four hours.

---

# 26. Daily Operating Flow

Example:

```text
05:30
ERP ingestion

05:50
Inventory snapshot

06:00
File ingestion

06:10
Data quality

06:20
Transformations

06:35
Mart refresh

06:45
Reconciliation

06:50
Publication evaluation

07:00
Dashboard ready
```

---

# 27. Monitoring

The platform tracks:

- pipeline status
- last successful run
- runtime
- row counts
- rejected rows
- source freshness
- data-quality failures
- reconciliation failures

Reporting domains can be classified as:

```text
READY
STALE
BLOCKED
```

---

# 28. Failure Recovery

The design supports:

- rerunning failed pipelines
- watermark recovery
- file replacement
- date-range reprocessing
- partition rebuilding
- domain-level publication

The preferred recovery model is:

```text
fix issue
↓
rerun pipeline
↓
validate
↓
republish
```

rather than manually editing warehouse data.

---

# 29. Cost-Conscious Design

The solution intentionally avoids unnecessary infrastructure.

Not used in the initial design:

```text
Kafka
Spark
Kubernetes
large Airflow deployment
real-time CDC platform
```

The use case does not require them.

This is a deliberate architecture decision.

---

# 30. Cost Controls

BigQuery cost is managed through:

- partitioning
- appropriate clustering
- incremental transformations
- selected-column queries
- prepared reporting models
- limited full-history scans
- controlled dashboard access

Looker Studio connects primarily to curated mart tables and views.

---

# 31. Synthetic Data Strategy

The project uses realistic synthetic data rather than real company information.

The synthetic environment contains approximately:

```text
2,500 customers
3,000 products
45 sales representatives
6 branches
3 warehouses
150k–200k orders
600k–1m order lines
```

over approximately three years.

---

# 32. Synthetic Business Patterns

The data intentionally contains realistic patterns such as:

- branch performance differences
- customer concentration
- seasonal revenue
- salesperson performance differences
- overdue receivables
- slow-moving inventory
- delayed shipments
- carrier-performance differences

This allows the reporting layer to support meaningful analysis.

---

# 33. Example Management Insights

The final dashboard may surface findings such as:

> Revenue is growing year over year, but overdue receivables have increased in one high-performing branch.

> Automation Components are driving strong revenue growth but require increased inventory availability.

> Several older product lines are tying up capital in slow-moving stock.

> One logistics carrier has materially weaker on-time performance than the others.

The final conclusions should be derived from the generated dataset.

---

# 34. Expected Business Outcome

The target improvement is:

## Before

```text
Multiple Sources
      ↓
Manual Extraction
      ↓
Spreadsheet Consolidation
      ↓
Lookups
      ↓
Reconciliation
      ↓
MIS Workbook
```

Manual effort:

```text
8–12 analyst hours
per reporting cycle
```

## After

```text
Multiple Sources
      ↓
Automated Ingestion
      ↓
Validated BigQuery Data
      ↓
Governed Data Mart
      ↓
Management Dashboard
      ↓
Exception Review
```

Target manual intervention:

```text
< 1 hour
per reporting cycle
```

primarily for review and exceptions.

---

# 35. Business Value

The solution is intended to improve:

### Efficiency

Less analyst time spent on repetitive data preparation.

### Consistency

One agreed definition for major management KPIs.

### Timeliness

Regular automated reporting rather than manual report preparation.

### Trust

Quality checks and reconciliation provide confidence in reported figures.

### Historical Analysis

BigQuery provides a centralized reporting history.

### Management Visibility

Business exceptions become easier to identify and investigate.

---

# 36. Skills Demonstrated

This project demonstrates the ability to work across the complete reporting lifecycle.

## Consulting / Analysis

- business problem discovery
- source assessment
- reporting requirements
- KPI definition
- architecture decisions
- scope management

## Data Engineering

- PostgreSQL
- Python ingestion
- incremental loading
- APIs
- file ingestion
- watermarks
- idempotency
- BigQuery

## Data Modelling

- dimensional modelling
- fact grain
- conformed dimensions
- snapshot facts
- incremental transformations

## Data Quality

- structural validation
- business rules
- reconciliation
- freshness
- publication controls

## Analytics

- KPI modelling
- management reporting
- Looker Studio
- business analysis

## Operations

- scheduling
- monitoring
- logging
- failure recovery
- runbooks

---

# 37. Repository Structure

Recommended final repository:

```text
management-reporting-data-platform/
│
├── README.md
│
├── .env.example
├── .gitignore
├── pyproject.toml
│
├── docs/
│   ├── 01-client-scenario.md
│   ├── 02-source-system-assessment.md
│   ├── 03-reporting-requirements.md
│   ├── 04-solution-architecture.md
│   ├── 05-postgresql-source-schema.md
│   ├── 06-synthetic-data-generation.md
│   ├── 07-python-ingestion.md
│   ├── 08-bigquery-data-model.md
│   ├── 09-data-quality-framework.md
│   ├── 10-looker-studio-dashboard.md
│   ├── 11-scheduling-monitoring.md
│   └── 12-final-case-study.md
│
├── database/
│   ├── schema/
│   ├── seed/
│   └── queries/
│
├── ingestion/
│   ├── common/
│   ├── postgres/
│   ├── files/
│   └── api/
│
├── bigquery/
│   ├── staging/
│   ├── transformations/
│   ├── marts/
│   └── quality/
│
├── orchestration/
│
├── monitoring/
│
├── dashboards/
│   ├── screenshots/
│   └── validation/
│
├── diagrams/
│
└── tests/
```

---

# 38. GitHub Presentation

The main README should remain concise.

It should show:

```text
Project Problem
      ↓
Architecture
      ↓
Technology Stack
      ↓
Expected Outcome
      ↓
Current Status
```

Detailed design belongs in `docs/`.

---

# 39. README Visuals

The final README should contain at least:

```text
1 architecture diagram
```

and:

```text
1 executive dashboard screenshot
```

These give visitors immediate visual context.

---

# 40. Architecture Diagram

The final diagram should clearly show:

```text
PostgreSQL
Excel / CSV
REST API
     ↓
Python
     ↓
BigQuery
     ↓
Staging
Transformation
Mart
     ↓
Looker Studio
```

with supporting:

```text
Data Quality
Monitoring
Scheduling
```

---

# 41. Dashboard Screenshots

Recommended final screenshots:

```text
executive-overview.png
sales-profitability.png
receivables.png
inventory.png
logistics.png
```

The executive dashboard screenshot should be the main portfolio visual.

---

# 42. Implementation Evidence

A strong portfolio should also contain evidence that the system works.

Examples:

- pipeline logs
- BigQuery table screenshots
- DQ results
- reconciliation output
- pipeline run status
- monitoring dashboard
- dashboard screenshots

The project should not rely only on architecture diagrams.

---

# 43. Important GitHub Principle

The repository should demonstrate:

```text
thinking
+
implementation
+
results
```

not only:

```text
documentation
```

The final portfolio value comes from showing that the designed architecture was actually implemented.

---

# 44. Limitations

The case study should explicitly acknowledge scope limitations.

Examples:

- synthetic data only
- no real client deployment
- simplified ERP
- one-invoice-per-order pattern used frequently
- simplified payment allocation
- no full SCD Type 2 implementation
- no real-time processing
- lightweight orchestration
- limited enterprise IAM implementation

This makes the case study more credible, not weaker.

---

# 45. Future Enhancements

Potential future improvements include:

```text
Cloud Run Jobs deployment
Cloud Scheduler
Secret Manager
SCD Type 2 dimensions
dbt transformations
advanced alerting
forecasting
demand planning
customer risk scoring
```

These should be presented as future options, not missing requirements.

---

# 46. Architecture Decision — No Streaming

A key design decision was not to use real-time streaming.

Reason:

```text
daily management reporting
+
4-hour logistics refresh
```

does not justify streaming infrastructure.

This demonstrates fit-for-purpose architecture rather than technology accumulation.

---

# 47. Architecture Decision — BigQuery

BigQuery was selected because it provides:

- managed analytical storage
- scalable SQL
- partitioning
- simple Looker Studio integration
- low operational overhead
- cost control for the project scale

---

# 48. Architecture Decision — Python

Python provides one flexible integration layer across:

```text
database
files
API
```

while keeping business transformation in BigQuery SQL.

---

# 49. Architecture Decision — Looker Studio

Looker Studio provides a practical portfolio reporting layer without requiring expensive BI licensing.

The project demonstrates dashboard design and management reporting while keeping the solution accessible.

---

# 50. Architecture Decision — Lightweight Orchestration

Cron / scheduled Python execution is sufficient for the current workload.

A larger orchestration platform would add complexity without solving a current business requirement.

---

# 51. Consulting Narrative

The case study should be positioned as:

> A management reporting transformation engagement.

Not:

> I created a Python ETL project.

The difference is important.

The project begins with:

```text
business problem
```

not:

```text
technology
```

---

# 52. Portfolio Summary — Long Version

A suitable detailed portfolio description:

> Designed and implemented a consulting-style management reporting automation solution for a fictional mid-sized B2B distributor that relied on manual MIS preparation across PostgreSQL, Excel/CSV, and external logistics data.
>
> The solution consolidates multi-source operational data through Python ingestion pipelines into BigQuery, where staging, transformation, and dimensional data-mart layers standardize KPIs across sales, profitability, receivables, inventory, and logistics.
>
> The design includes incremental ingestion using watermarks, idempotent BigQuery loads, financial reconciliation, data-quality controls, historical receivable and inventory snapshots, scheduling, monitoring, and Looker Studio management dashboards.
>
> The objective is to reduce repetitive MIS preparation from approximately 8–12 analyst hours per reporting cycle to an automated workflow requiring mainly exception review.

---

# 53. Malt / Upwork Portfolio Version

A shorter client-facing version:

> **Multi-Source MIS & Management Reporting Automation**
>
> Designed an end-to-end reporting automation solution for a fictional mid-sized distribution business whose management reports were manually assembled from ERP data, spreadsheets, and logistics information.
>
> Built the solution around Python, PostgreSQL, BigQuery, REST APIs, Excel/CSV, and Looker Studio.
>
> The project covers source assessment, automated ingestion, incremental loading, KPI standardization, data modelling, data-quality checks, financial reconciliation, dashboarding, scheduling, and monitoring.
>
> The target outcome was to replace 8–12 hours of repetitive MIS preparation with a trusted, regularly refreshed management reporting workflow.

---

# 54. Very Short Portfolio Version

For platforms with limited space:

> End-to-end MIS automation solution integrating PostgreSQL, Excel/CSV, and REST API data through Python into BigQuery and Looker Studio, with incremental pipelines, data-quality controls, reconciliation, reporting marts, scheduling, and monitoring.

---

# 55. Recruiter-Friendly Version

For GitHub / LinkedIn:

> Built a realistic multi-source management reporting platform covering PostgreSQL and file/API ingestion, Python pipelines, BigQuery staging and dimensional modelling, data-quality controls, reconciliation, historical snapshots, Looker Studio dashboards, and scheduled monitoring.

---

# 56. Interview Talking Points

If asked:

> Tell me about this project.

A good explanation is:

> I started with a reporting problem rather than a technology stack. The fictional client had an MIS analyst manually combining PostgreSQL exports, Finance spreadsheets, Sales targets, and logistics data every reporting cycle.
>
> I first defined the reporting requirements and KPI ownership, then designed a lightweight batch architecture using Python and BigQuery. PostgreSQL transactions are loaded incrementally using watermarks, files are validated and version-tracked, and the API pipeline supports pagination and retries.
>
> In BigQuery I separated staging, transformation, and reporting marts. Revenue is based on invoice lines, while orders remain a separate demand metric. Receivables and inventory are modelled as snapshots because management needs historical as-of reporting.
>
> I also added reconciliation and data-quality gates before the data is exposed to Looker Studio. The goal was to demonstrate an end-to-end reporting solution rather than just a dashboard.

---

# 57. What Makes the Project Different

Many portfolio projects demonstrate:

```text
dataset
→ dashboard
```

This project demonstrates:

```text
business problem
        ↓
source assessment
        ↓
requirements
        ↓
architecture
        ↓
source modelling
        ↓
data generation
        ↓
ingestion
        ↓
warehouse
        ↓
data quality
        ↓
reporting
        ↓
operations
        ↓
business outcome
```

That is the main value of the case study.

---

# 58. Final Portfolio Message

The portfolio should communicate:

> I can understand how a business currently works, identify where reporting is inefficient or unreliable, design an appropriate data architecture, integrate multiple source systems, standardize reporting logic, and deliver a maintainable management reporting solution.

The project should therefore be presented as evidence of:

```text
solution ownership
```

rather than only:

```text
tool proficiency
```

---

# 59. Final Success Criteria

The complete portfolio project is finished when:

1. PostgreSQL source environment is running.
2. Synthetic business data is generated.
3. Excel / CSV sources are generated.
4. Mock logistics API is working.
5. Python ingestion pipelines are operational.
6. Initial and incremental loads work.
7. BigQuery staging is populated.
8. Transformation models are implemented.
9. Dimensions and facts are built.
10. Data-quality checks are running.
11. Reconciliation is passing.
12. Looker Studio dashboards are complete.
13. Scheduled execution is working.
14. Monitoring shows pipeline status and freshness.
15. Architecture diagram is published.
16. Dashboard screenshots are added.
17. README reflects the implemented solution.
18. Final case study reflects actual outcomes rather than planned functionality.

---

# 60. Final Outcome

The completed project should demonstrate the transition:

```text
Manual MIS
    ↓
Fragmented Data
    ↓
Automated Integration
    ↓
Governed Data Model
    ↓
Trusted KPIs
    ↓
Management Dashboard
    ↓
Operational Monitoring
```

The final message is simple:

> This project demonstrates the ability to take ownership of a reporting problem from requirements and source assessment through data engineering, modelling, quality, dashboarding, automation, and operational handover.