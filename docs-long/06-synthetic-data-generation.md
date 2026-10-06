# 06 — Synthetic Data Generation

## Purpose

This document defines the strategy for generating realistic synthetic source data for the Multi-Source MIS & Management Reporting Automation project.

The goal is not to generate random rows.

The synthetic dataset should behave like a real mid-sized B2B distribution business and support meaningful analysis across:

- sales
- profitability
- receivables
- inventory
- branch performance
- salesperson performance
- customer concentration
- logistics
- targets and budgets

The generated data should also contain controlled operational and integration issues so the data-quality framework can be tested realistically.

---

# 1. Synthetic Data Objectives

The generated dataset should satisfy five objectives.

## Realistic Business Behaviour

The data should contain patterns that look commercially plausible.

Examples:

- some branches consistently outperform others
- some sales representatives exceed targets
- a small group of customers contributes a large share of revenue
- some products sell frequently while others move slowly
- revenue varies by season
- some invoices are paid late
- some shipments are delayed
- inventory levels change over time

---

## Relational Consistency

The PostgreSQL source database should respect:

- primary keys
- foreign keys
- status rules
- valid date sequences
- valid monetary values

The database itself should remain structurally reliable.

---

## Reproducibility

The same configuration should be able to regenerate the same dataset.

A fixed random seed will be used.

Example:

```text
RANDOM_SEED=20261006
```

This helps with:

- debugging
- testing
- documentation
- repeatable portfolio demonstrations

---

## Configurability

Generation volume and historical period should be configurable.

For example:

```text
START_DATE=2024-01-01
END_DATE=2026-09-30

CUSTOMER_COUNT=2500
PRODUCT_COUNT=3000
SALES_REP_COUNT=45

MIN_MONTHLY_ORDERS=4000
MAX_MONTHLY_ORDERS=6000
```

This allows a smaller dataset during development and a larger dataset for the final demonstration.

---

## Controlled Imperfections

The dataset should contain realistic business exceptions without destroying referential integrity.

Examples:

- overdue invoices
- cancelled orders
- low-stock products
- delayed shipments
- target underperformance
- unknown SKU in one external file
- duplicate spreadsheet row
- late-arriving shipment update

---

# 2. Data Generation Scope

Synthetic data will be generated for:

## PostgreSQL

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

## Files

```text
sales_targets_2026.xlsx
branch_budget_2026.xlsx
monthly_operating_expenses.xlsx
product_margin_adjustments.csv
```

## Logistics API

Synthetic shipment data will be generated separately and exposed through the simulated API.

---

# 3. Historical Period

The initial dataset should cover approximately three years.

Recommended period:

```text
2024-01-01
to
2026-09-30
```

This provides enough history for:

- monthly trends
- quarter comparisons
- year-over-year analysis
- financial-year reporting
- receivable aging
- salesperson trends
- branch trends

The exact end date should be configurable.

---

# 4. Development vs Final Dataset

Two generation profiles should be supported.

## Development Profile

Used for fast local testing.

Example:

```text
Customers:        300
Products:         500
Sales Reps:       15
Orders / Month:   300–500
History:          6 months
```

## Portfolio Profile

Used for the final project demonstration.

Example:

```text
Customers:        ~2,500
Products:         ~3,000
Sales Reps:       ~45
Orders / Month:   4,000–6,000
History:          ~3 years
```

This prevents development from becoming unnecessarily slow.

---

# 5. Recommended Generator Structure

The generator should be written in Python.

Suggested repository structure:

```text
database/
└── seed/
    ├── generate.py
    ├── config.py
    ├── random_utils.py
    │
    ├── generators/
    │   ├── branches.py
    │   ├── warehouses.py
    │   ├── sales_reps.py
    │   ├── customers.py
    │   ├── products.py
    │   ├── orders.py
    │   ├── invoices.py
    │   ├── payments.py
    │   └── inventory.py
    │
    └── scenarios/
        ├── sales_patterns.py
        ├── payment_patterns.py
        ├── inventory_patterns.py
        └── business_events.py
```

External source generation can be separated:

```text
source-data/
├── excel/
├── csv/
└── logistics/
```

---

# 6. Generation Sequence

Data must be created in dependency order.

Recommended sequence:

```text
1. Branches
2. Warehouses
3. Sales Representatives
4. Customers
5. Products
6. Sales Orders
7. Sales Order Lines
8. Invoices
9. Invoice Lines
10. Payments
11. Inventory
12. Excel / CSV Sources
13. Logistics Shipments
```

---

# 7. Branch Generation

The project will initially use six branches.

Suggested locations:

| Code | Branch | Region |
|---|---|---|
| MUM | Mumbai | West |
| DEL | Delhi | North |
| BLR | Bengaluru | South |
| CHN | Chennai | South |
| PUN | Pune | West |
| AHM | Ahmedabad | West |

Branches should not have identical commercial performance.

Suggested relative revenue weights:

```text
MUM  1.25
DEL  1.15
BLR  1.10
PUN  0.95
CHN  0.90
AHM  0.75
```

These are business simulation weights, not fixed revenue percentages.

They create meaningful branch comparisons later.

---

# 8. Warehouse Generation

Three warehouses can support the initial scenario.

Example:

| Warehouse | Primary Region |
|---|---|
| WH-MUM | West |
| WH-DEL | North |
| WH-BLR | South |

Warehouses may support multiple branches.

The source schema currently associates each warehouse with a branch, so the selected warehouse locations should map to the appropriate primary branch.

---

# 9. Sales Representative Generation

Approximately 45 sales representatives should be distributed unevenly across branches.

Example:

```text
Mumbai       10
Delhi         9
Bengaluru     8
Pune          7
Chennai       6
Ahmedabad     5
```

Each salesperson should receive a performance factor.

Example distribution:

```text
Top Performers          15%
Above Average           25%
Average                 40%
Below Average           15%
Low Performer            5%
```

This factor should influence:

- order frequency
- order value
- target achievement
- customer activity

This will create realistic salesperson rankings.

---

# 10. Customer Generation

Approximately 2,500 customers should be created.

## Customer Segments

Suggested distribution:

```text
ENTERPRISE       10%
MID_MARKET       25%
SME              50%
DEALER           15%
```

---

## Customer Revenue Behaviour

Revenue contribution should not be uniform.

A Pareto-style distribution should be used.

For example:

```text
~20% of customers
generate
~65–75% of revenue
```

This produces realistic customer concentration.

---

## Customer Activity Classes

Customers can be classified internally during generation as:

```text
HIGH_ACTIVITY
MEDIUM_ACTIVITY
LOW_ACTIVITY
DORMANT
```

These generation-only classifications do not need to exist in the ERP schema.

They influence:

- order frequency
- order value
- payment behavior

---

# 11. Customer Credit Profile

Customer payment behavior should vary.

Example synthetic profiles:

```text
LOW_RISK
STANDARD
SLOW_PAYER
HIGH_RISK
```

Suggested distribution:

```text
LOW_RISK       25%
STANDARD       50%
SLOW_PAYER     20%
HIGH_RISK       5%
```

These profiles influence payment timing later.

---

# 12. Credit Limits

Credit limits should depend partly on customer size.

Example ranges:

```text
ENTERPRISE:
₹25 lakh – ₹2 crore

MID_MARKET:
₹10 lakh – ₹60 lakh

SME:
₹2 lakh – ₹20 lakh

DEALER:
₹5 lakh – ₹30 lakh
```

Values should vary rather than being fixed.

---

# 13. Payment Terms

Example distribution:

```text
15 days     15%
30 days     45%
45 days     25%
60 days     15%
```

Larger customers may be more likely to receive longer terms.

---

# 14. Product Generation

Approximately 3,000 products should be generated.

Suggested categories:

```text
Bearings
Electrical Components
Industrial Tools
Safety Equipment
Pumps & Valves
Fasteners
Automation Components
Maintenance Supplies
```

Each category should contain multiple subcategories.

---

# 15. Product Sales Behaviour

Product demand should follow a long-tail distribution.

For example:

```text
Top 10% of products
→ high sales frequency

Next 30%
→ moderate demand

Remaining 60%
→ low or occasional demand
```

This allows the final dashboard to show:

- top products
- slow movers
- category concentration
- stock imbalance

---

# 16. Product Pricing

Product values should differ by category.

Example price ranges:

```text
Fasteners
₹50 – ₹2,000

Safety Equipment
₹300 – ₹15,000

Industrial Tools
₹2,000 – ₹75,000

Automation Components
₹5,000 – ₹2,00,000

Pumps & Valves
₹3,000 – ₹1,50,000
```

These should be generated from distributions rather than uniformly across the range.

---

# 17. Product Margin Profiles

Different categories should have different typical margins.

Example:

| Category | Typical Gross Margin |
|---|---:|
| Fasteners | 18–25% |
| Bearings | 15–22% |
| Industrial Tools | 20–30% |
| Safety Equipment | 25–35% |
| Pumps & Valves | 15–25% |
| Automation Components | 18–28% |

This makes category profitability analysis meaningful.

---

# 18. Seasonality

Monthly revenue should not be flat.

Suggested seasonal index:

| Month | Factor |
|---|---:|
| Jan | 0.95 |
| Feb | 0.98 |
| Mar | 1.15 |
| Apr | 0.90 |
| May | 0.95 |
| Jun | 1.00 |
| Jul | 0.98 |
| Aug | 1.02 |
| Sep | 1.08 |
| Oct | 1.10 |
| Nov | 1.05 |
| Dec | 0.94 |

March can be stronger due to financial-year closing activity.

The values are simulation assumptions rather than claims about all real industrial businesses.

---

# 19. Annual Growth

The company should show moderate growth over the synthetic period.

Example:

```text
2024 baseline

2025:
+8% to +12%

2026:
+10% to +15%
```

Growth should come from a combination of:

- higher customer activity
- new customers
- pricing changes
- branch growth

---

# 20. Order Generation

Monthly order volume should vary.

Portfolio target:

```text
4,000–6,000 orders per month
```

The number of orders should depend on:

```text
month seasonality
×
branch performance factor
×
salesperson performance factor
×
customer activity
```

---

# 21. Order Status Distribution

Suggested completed lifecycle distribution:

```text
COMPLETED       75%
DISPATCHED       7%
PROCESSING       5%
CONFIRMED        4%
CANCELLED        6%
REJECTED         2%
DRAFT            1%
```

Historical months should contain a higher percentage of completed records.

Recent periods may contain more open orders.

---

# 22. Order Line Generation

Each order should contain approximately:

```text
2–8 lines
```

with a distribution centered around 3–4 lines.

High-value enterprise customers may order more lines on average.

Products should not be selected uniformly.

High-demand products should appear significantly more often.

---

# 23. Order Pricing

Actual selling price can differ from list price.

Example:

```text
unit_price
=
list_price
× customer discount factor
× small random variation
```

Enterprise customers may receive larger discounts.

Example discount ranges:

```text
ENTERPRISE      5–15%
MID_MARKET      3–10%
SME             0–7%
DEALER          8–18%
```

---

# 24. Transaction Cost

Order and invoice lines should capture `unit_cost`.

At transaction time:

```text
unit_cost
≈ product standard cost
```

with small variations to simulate:

- freight
- procurement changes
- temporary supplier cost differences

Historical costs should gradually change over time.

---

# 25. Invoice Generation

Not every order should immediately become an invoice.

Eligible statuses:

```text
DISPATCHED
COMPLETED
```

Most valid fulfilled orders should generate invoices.

Some orders may be:

- partially fulfilled
- invoiced later
- cancelled before invoicing

For simplicity, the initial implementation may use one invoice per order for most transactions.

---

# 26. Invoice Timing

Invoice date should normally be:

```text
order_date
+
1 to 10 days
```

depending on fulfillment time.

Enterprise or large orders may occasionally take longer.

---

# 27. Invoice Lines

Invoice lines should normally originate from order lines.

Relationships should preserve:

```text
invoice_line.order_line_id
→ sales_order_lines.order_line_id
```

Invoice quantity may occasionally be less than ordered quantity to represent partial fulfillment.

The initial version can keep partial invoicing relatively uncommon.

---

# 28. Payment Generation

Payment timing should depend on customer payment profile.

## LOW_RISK

Typical payment behavior:

```text
Before due date
or
0–5 days after due date
```

## STANDARD

Typical:

```text
-5 to +15 days around due date
```

## SLOW_PAYER

Typical:

```text
15–45 days after due date
```

## HIGH_RISK

Possible:

```text
30–90+ days late
or
still unpaid
```

---

# 29. Partial Payments

Some invoices should receive multiple payments.

Suggested:

```text
Fully paid in one payment       65%
Multiple / partial payments     20%
Outstanding balance             15%
```

These values can vary based on customer risk profile.

---

# 30. Payment Status Distribution

Most generated payments should be successful.

Example:

```text
SUCCESS     95%
PENDING      1%
FAILED       2%
REVERSED     2%
```

Only successful payments should reduce receivables.

---

# 31. Receivable Aging Patterns

The final dataset should contain meaningful values in all aging buckets.

Example target distribution of outstanding value:

```text
Not Due         45%
1–30 Days       25%
31–60 Days      15%
61–90 Days       8%
90+ Days         7%
```

The generator should produce the pattern naturally through invoice dates, due dates, and payment timing rather than writing aging buckets directly.

---

# 32. Inventory Generation

Initial inventory should depend on product popularity.

High-demand products should generally have higher stock.

Example:

```text
high-demand products
→ larger stock and reorder level

slow-moving products
→ lower demand but potentially excess stock
```

---

# 33. Inventory Availability

For each warehouse/product:

```text
quantity_available
=
quantity_on_hand
-
quantity_reserved
```

This must remain valid in PostgreSQL.

---

# 34. Low-Stock Scenarios

Approximately:

```text
5–10%
```

of active warehouse/product combinations may be at or below reorder level.

This creates meaningful low-stock reporting.

---

# 35. Slow-Moving Inventory

Some products should retain inventory despite having no recent sales.

Example:

```text
8–15% of stocked SKUs
```

could have no sale in the previous 90 days.

This allows the slow-moving inventory KPI to produce useful output.

---

# 36. Inventory Movement Simulation

The source database contains only current inventory.

Therefore, for initial generation:

1. create historical transactions
2. derive approximate consumption
3. generate final current stock
4. allow the ingestion process to create future daily snapshots

We do not need to generate three years of inventory snapshots directly in PostgreSQL.

If historical inventory trend demonstration is required later, synthetic snapshot history can be seeded directly into BigQuery.

---

# 37. Sales Target Generation

The file:

```text
sales_targets_2026.xlsx
```

should contain monthly targets by salesperson.

Targets should be based partly on historical performance.

Example:

```text
Target
=
Previous Period Baseline
×
Growth Expectation
```

with a small planning adjustment.

---

# 38. Target Achievement Patterns

Do not make everyone achieve exactly 100%.

Suggested behavior:

```text
Top performers:
105–130%

Above average:
95–115%

Average:
85–105%

Below average:
65–90%

Low performer:
45–75%
```

Monthly variation should still exist.

---

# 39. Branch Budget Generation

The file:

```text
branch_budget_2026.xlsx
```

should contain:

```text
month
branch_code
revenue_budget
expense_budget
```

Revenue budgets should approximately follow:

```text
previous performance
+
planned growth
```

Expenses should reflect branch size.

---

# 40. Operating Expense Generation

The file:

```text
monthly_operating_expenses.xlsx
```

should contain realistic expense categories.

Examples:

```text
Rent
Payroll Support
Utilities
Travel
Marketing
Warehouse Operations
Vehicle Expense
Administration
IT Services
```

Some categories should be relatively fixed.

Examples:

```text
Rent
IT Services
```

Others should vary with business activity.

Examples:

```text
Travel
Warehouse Operations
Vehicle Expense
```

---

# 41. Product Margin Adjustments

The CSV file:

```text
product_margin_adjustments.csv
```

should contain occasional finance adjustments.

Examples:

```text
FREIGHT_ADJUSTMENT
SUPPLIER_REBATE
COST_CORRECTION
SPECIAL_DISCOUNT
```

Adjustments should affect only a small proportion of products.

Example:

```text
1–3% of product/month combinations
```

---

# 42. Logistics Shipment Generation

A shipment should normally be created for dispatched or completed orders.

Suggested relationship:

```text
sales_order
→ shipment
```

Most orders can have one shipment.

A small number may have multiple shipments later if we want to simulate split fulfillment, but this is not required initially.

---

# 43. Logistics Carrier Simulation

Create several fictional carriers.

Example:

```text
FastRoute Logistics
BlueLine Transport
SwiftCargo
MetroFreight
National Express Logistics
```

Carriers should have different delivery performance.

Example:

| Carrier Profile | On-Time Rate |
|---|---:|
| Strong | 94–97% |
| Good | 90–94% |
| Average | 84–90% |
| Weak | 75–85% |

This allows meaningful carrier comparisons.

---

# 44. Shipment Timing

Typical delivery time may depend on destination.

Example:

```text
same region:
1–3 days

adjacent region:
2–5 days

long distance:
3–7 days
```

Expected delivery dates should use this logic.

Actual delivery dates should vary around the expectation.

---

# 45. Shipment Status Distribution

For historical shipments:

```text
DELIVERED
```

should dominate.

Recent records may include:

```text
DISPATCHED
IN_TRANSIT
OUT_FOR_DELIVERY
DELAYED
DELIVERED
CANCELLED
```

---

# 46. On-Time Delivery

Target overall performance:

```text
approximately 88–93%
```

This leaves enough late deliveries for meaningful analysis without making the logistics provider look unrealistic.

Performance should vary by:

- carrier
- region
- month

---

# 47. Business Events

The dataset should contain a few deliberate business events so time-series charts tell a story.

Examples:

## Event A — Mumbai Growth

From mid-2025:

```text
Mumbai branch sales
↑ approximately 15%
```

Possible explanation in the fictional case:

> Expansion of the regional sales team.

---

## Event B — Product Category Growth

Automation Components begin performing strongly during 2026.

This should create:

- revenue growth
- higher inventory demand
- improved category contribution

---

## Event C — Delhi Receivables Issue

During one quarter:

```text
Delhi overdue receivables
↑ materially
```

Caused by a small group of high-value slow-paying customers.

---

## Event D — Logistics Performance Drop

One carrier experiences poorer delivery performance for approximately two months.

This will become visible on the logistics dashboard.

---

## Event E — Slow-Moving Inventory

A subset of products retains inventory after demand declines.

This provides a realistic inventory-management issue.

---

# 48. Why Business Events Matter

Without deliberate business behavior, dashboards often show:

```text
random-looking charts
+
no meaningful story
```

A consulting portfolio is stronger when the dashboard can support observations such as:

> West-region revenue is growing, but a portion of that improvement is accompanied by higher overdue receivables.

or:

> Automation Components are driving growth, while several older product lines are accumulating slow-moving stock.

The data should make management analysis possible.

---

# 49. Synthetic Data Quality Strategy

There are two distinct categories.

## Valid Source Data

PostgreSQL data should generally satisfy:

- foreign keys
- primary keys
- database constraints
- valid types

## Integration / Business Issues

Problems should primarily appear where realistic systems often experience them.

Examples:

```text
spreadsheet duplicate
unknown SKU in adjustment file
shipment referencing unknown order
late-arriving API update
missing optional field
unexpected spreadsheet category
```

This is more realistic than intentionally breaking every source table.

---

# 50. Controlled Data Quality Scenarios

The final dataset should include a small number of known test cases.

Example catalogue:

| ID | Scenario | Source | Expected Severity |
|---|---|---|---|
| DQ-001 | Duplicate target row | Excel | Critical |
| DQ-002 | Unknown SKU adjustment | CSV | Warning |
| DQ-003 | Unknown order shipment | API | Warning |
| DQ-004 | Missing customer industry | PostgreSQL | Warning |
| DQ-005 | Late invoice update | PostgreSQL | Normal incremental case |
| DQ-006 | Duplicate API shipment received | API | Handled by idempotency |
| DQ-007 | Delayed shipment | API | Business condition |
| DQ-008 | Outstanding invoice >90 days | PostgreSQL | Business condition |

Known test cases should be documented so pipeline behavior can be verified.

---

# 51. Important Distinction

A business exception is not necessarily a data-quality failure.

For example:

```text
invoice unpaid for 120 days
```

is valid business data.

It represents a:

```text
business risk
```

not corrupted data.

Similarly:

```text
shipment delivered late
```

is a business-performance issue, not necessarily a technical data-quality issue.

This distinction should remain clear throughout the project.

---

# 52. Data Generation Configuration

Suggested configuration file:

```text
config/
└── synthetic_data.yaml
```

Example:

```yaml
seed: 20261006

period:
  start_date: "2024-01-01"
  end_date: "2026-09-30"

volume:
  customers: 2500
  products: 3000
  sales_reps: 45
  monthly_orders_min: 4000
  monthly_orders_max: 6000

payments:
  full_payment_rate: 0.65
  partial_payment_rate: 0.20
  outstanding_rate: 0.15

inventory:
  low_stock_rate: 0.08
  slow_moving_rate: 0.12

logistics:
  target_on_time_rate: 0.90
```

Actual implementation may use Python configuration rather than YAML if simpler.

---

# 53. Random Seed

A fixed seed should initialize all random generators.

For example:

```python
random.seed(20261006)
```

and where NumPy is used:

```python
np.random.seed(20261006)
```

This allows identical datasets to be reproduced.

---

# 54. Data Generation Libraries

Possible Python libraries include:

```text
Faker
random
NumPy
pandas
SQLAlchemy
psycopg
openpyxl
```

Use only libraries that materially simplify generation.

The generator should remain understandable.

---

# 55. Faker Usage

Faker can generate:

- names
- company names
- cities
- addresses

However, Faker should not determine important business behavior.

For example:

```text
customer revenue
payment performance
branch growth
product demand
```

should come from our simulation rules rather than pure Faker output.

---

# 56. Batch Inserts

Large datasets should not be inserted one row at a time.

Preferred approaches include:

```text
executemany
bulk insert
COPY
SQLAlchemy bulk operations
```

For high-volume fact data, PostgreSQL `COPY` may be used for faster initial seeding.

---

# 57. Transaction Management

Generation should occur in controlled transactions.

For example:

```text
Generate master table
       ↓
Bulk insert
       ↓
Commit
       ↓
Generate dependent table
```

If a generation stage fails, the error should be clear and recoverable.

---

# 58. Generator Logging

The data generator should log:

```text
generation step
start time
end time
rows generated
rows inserted
elapsed time
status
```

Example:

```text
[INFO] Generating customers...
[INFO] Customers generated: 2,500

[INFO] Generating orders for 2025-03...
[INFO] Orders generated: 5,421
[INFO] Order lines generated: 18,905
```

---

# 59. Validation After Generation

Generation is not considered complete simply because records were inserted.

Post-generation checks should verify:

## Row Counts

```text
customers > 0
products > 0
orders > 0
invoices > 0
```

## Referential Integrity

Examples:

```text
all order customer IDs valid
all order-line product IDs valid
all invoice order IDs valid
```

## Financial Logic

Examples:

```text
order totals approximately equal line totals + tax

invoice totals reconcile to invoice lines

successful payments do not create invalid negative balances
```

## Inventory Logic

```text
quantity_available
=
quantity_on_hand
-
quantity_reserved
```

---

# 60. Distribution Validation

Business distributions should also be tested.

Examples:

```text
cancelled order rate ≈ expected range

overall delivery performance ≈ 90%

revenue concentration exists

all branches have transactions

all major product categories have sales

some customers have overdue balances

some SKUs are slow moving
```

These are important because technically valid data can still be unrealistic.

---

# 61. Generated File Validation

After Excel and CSV files are created, validate:

```text
expected columns
expected file names
row counts
business keys
date ranges
```

Deliberate DQ scenarios should be added only after the clean base dataset is generated.

This makes test-case behavior easier to understand.

---

# 62. API Dataset

Shipment records should be stored in a separate source dataset used by the simulated logistics API.

Possible format:

```text
source-data/logistics/shipments.json
```

or a lightweight database behind the mock API.

The pipeline should consume the API rather than reading this source file directly.

This preserves the integration scenario.

---

# 63. API Update Simulation

Shipment records should include:

```text
created_at
updated_at
```

to support incremental extraction.

Some shipments should change status over time.

Example:

```text
DISPATCHED
    ↓
IN_TRANSIT
    ↓
OUT_FOR_DELIVERY
    ↓
DELIVERED
```

The API simulation should allow updated records to be returned again through `updated_since`.

---

# 64. Late-Arriving Data

The generator should include some records where:

```text
business event date
<
record updated_at
```

Example:

```text
Invoice date:
2026-08-15

Invoice cancellation recorded:
2026-08-25
```

This demonstrates why incremental ingestion uses:

```text
updated_at
```

rather than only transaction dates.

---

# 65. Data Volume Expectations

Approximate final portfolio volume:

| Dataset | Approximate Volume |
|---|---:|
| Branches | 6 |
| Warehouses | 3 |
| Sales Representatives | 45 |
| Customers | 2,500 |
| Products | 3,000 |
| Sales Orders | 150k–200k |
| Sales Order Lines | 600k–1m |
| Invoices | 140k–190k |
| Invoice Lines | 500k–900k |
| Payments | 120k–180k |
| Current Inventory | ~9,000 |
| Sales Targets | ~540/year |
| Shipments | 140k–190k |

Exact numbers should vary naturally with configuration.

---

# 66. Runtime Consideration

Generating approximately one million transactional rows is enough to demonstrate:

- realistic ingestion
- incremental loading
- BigQuery transformations
- partitioning
- reconciliation
- dashboard performance

There is no portfolio benefit in generating tens or hundreds of millions of rows purely for scale.

The project should remain cost-conscious and runnable.

---

# 67. Rebuild Strategy

The generated environment should be disposable.

A developer should eventually be able to run something similar to:

```text
reset database
       ↓
create schema
       ↓
generate synthetic data
       ↓
create external files
       ↓
initialize API dataset
```

This provides a repeatable source environment.

---

# 68. Generator Idempotency

Running the generator accidentally against an already populated database should not silently duplicate the entire dataset.

Possible strategies:

```text
require empty target tables
```

or:

```text
explicit --reset flag
```

Example:

```bash
python generate.py --profile portfolio --reset
```

The exact CLI will be defined during implementation.

---

# 69. Suggested Generator Commands

Conceptually:

```bash
python -m database.seed.generate --profile development
```

and:

```bash
python -m database.seed.generate --profile portfolio
```

Optional:

```bash
python -m database.seed.generate \
    --profile portfolio \
    --seed 20261006
```

---

# 70. Security and Privacy

All generated information must be synthetic.

The generator should not use:

- real customer records
- actual company financial data
- personal employee data
- production source exports

Names may resemble realistic companies or individuals, but they should be generated purely for this project.

---

# 71. Expected Analytical Stories

After generation, the dataset should support findings such as:

### Sales

> Mumbai and Bengaluru are driving most year-over-year growth.

### Target Performance

> A small group of sales representatives consistently exceed targets, while several remain below 80%.

### Customer Concentration

> The top 20% of customers contribute the majority of recognized revenue.

### Profitability

> Safety Equipment has strong margins while some high-revenue categories generate weaker margins.

### Receivables

> Delhi has increased revenue but also carries a disproportionate amount of 60+ day receivables.

### Inventory

> Several low-demand SKUs continue to consume working capital.

### Logistics

> One carrier has materially weaker on-time delivery performance during a specific period.

The exact observations will be derived from the generated data rather than hardcoded into the dashboard.

---

# 72. Generation Outcome

The synthetic data layer should create a realistic fictional operating environment:

```text
Business Rules
      ↓
Synthetic Transactions
      ↓
PostgreSQL ERP
      +
Excel / CSV
      +
Logistics API
      ↓
Multi-Source Reporting Pipeline
```

The quality of this synthetic source environment is important because every later portfolio artifact depends on it.

Good synthetic data enables us to demonstrate:

```text
engineering
+
data quality
+
business analysis
+
management reporting
```

rather than merely demonstrating that code executes.

---

## Next Step

After the PostgreSQL schema and synthetic source environment are implemented, the next major design document should be:

**`07-ingestion-pipeline-design.md`**

It will define the actual ingestion implementation for:

- PostgreSQL → BigQuery
- Excel / CSV → BigQuery
- Logistics API → BigQuery

including:

- connection management
- extraction patterns
- batch sizes
- watermarks
- BigQuery load strategy
- `MERGE` handling
- file validation
- retries
- pipeline audit records
- error handling
- configuration
- local execution
- Docker execution