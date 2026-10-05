# 10 — Looker Studio Dashboard Design

## Purpose

This document defines the dashboard design for the Multi-Source MIS & Management Reporting Automation project.

The dashboard is intended for management users who need a concise and trusted view of:

- revenue
- sales performance
- profitability
- receivables
- inventory
- logistics

The dashboard should consume only governed data from:

```text id="d1a4kh"
mis_mart
```

and should not recreate complex business logic inside Looker Studio.

The reporting layer should be:

- simple to navigate
- visually clear
- business-focused
- consistent across pages
- performant
- traceable to documented KPIs

---

# 1. Dashboard Audience

Primary users:

```text id="rlr04r"
Managing Director / CEO
Finance Head
Sales Head
Operations Head
Branch Managers
MIS / Analytics Team
```

Different users may focus on different pages, but the dashboard should remain one consistent management-reporting solution.

---

# 2. Dashboard Goals

The dashboard should help management quickly answer:

```text id="yraw9r"
How are we performing?

Where are we underperforming?

What changed?

Why did it change?

Where should management investigate?
```

The dashboard should not attempt to expose every field available in the data warehouse.

---

# 3. Dashboard Structure

The initial dashboard will contain five pages.

```text id="f22vod"
1. Executive Overview
2. Sales & Profitability
3. Receivables
4. Inventory
5. Logistics
```

An optional sixth page may later be added for:

```text id="p5w4jg"
Data / Pipeline Health
```

but this should remain separate from the management reporting experience.

---

# 4. Navigation

A simple top navigation should allow movement between:

```text id="air4o6"
Executive
Sales
Receivables
Inventory
Logistics
```

The active page should be visually obvious.

Avoid overly complex menus.

---

# 5. Common Dashboard Filters

Common filters should be available where relevant.

Recommended filters:

```text id="1jngoo"
Date Range
Financial Year
Branch
Region
Sales Representative
Customer Segment
Product Category
Warehouse
Carrier
```

Not every filter should appear on every page.

For example:

```text id="mot5di"
Carrier
```

belongs mainly on the Logistics page.

---

# 6. Default Reporting Period

The Executive Overview should default to:

```text id="gimgjj"
Current Financial Year
```

or:

```text id="d4r91m"
Current Month
```

depending on the visual.

The selected period should always be obvious to the user.

---

# 7. Comparison Logic

Management KPIs should include meaningful comparisons.

Examples:

```text id="3n9tcp"
Current Month
vs
Previous Month
```

```text id="khi6vh"
Current Month
vs
Same Month Last Year
```

```text id="8pjcxn"
Actual
vs
Target
```

```text id="wzp3lq"
Actual
vs
Budget
```

Comparisons should not be added simply for visual decoration.

---

# 8. Page 1 — Executive Overview

## Purpose

Provide senior management with a quick view of overall company performance.

A user should understand the business condition within approximately one minute.

---

# 9. Executive KPI Cards

Recommended top-level cards:

```text id="ktdfsp"
Recognized Revenue

Revenue Growth %

Sales Target Achievement %

Gross Margin %

Outstanding Receivables

Overdue Receivables

Inventory Value

On-Time Delivery %
```

These should represent the main reporting domains.

---

# 10. Revenue Card

Display:

```text id="k2cluj"
Recognized Revenue
```

with comparison:

```text id="uyr71c"
vs Previous Period
```

or:

```text id="yfzdhu"
YoY %
```

Example:

```text id="6gjjo8"
₹12.4 Cr

+8.7% YoY
```

---

# 11. Target Achievement Card

Display:

```text id="hh22mv"
Sales Target Achievement
```

Example:

```text id="hxvr47"
96.4%
```

Supporting indicator:

```text id="w12hc3"
Actual Revenue
₹12.4 Cr

Target
₹12.9 Cr
```

---

# 12. Gross Margin Card

Display:

```text id="f34xoc"
Gross Margin %
```

with previous-period comparison.

Example:

```text id="c5wvwh"
23.8%

Previous Month:
22.6%
```

---

# 13. Receivable Cards

Two separate metrics should remain visible.

```text id="u9npvm"
Outstanding Receivables
```

and:

```text id="z1kzj3"
Overdue Receivables
```

This prevents management from confusing total open receivables with genuinely overdue amounts.

---

# 14. Executive Visual — Revenue Trend

Recommended:

```text id="i4wjio"
Line chart
```

showing:

```text id="963p7s"
Month
vs
Recognized Revenue
```

Optional comparison:

```text id="r67uwk"
Previous Year Revenue
```

This should make growth and seasonality visible.

---

# 15. Executive Visual — Actual vs Target

Recommended:

```text id="k9gqd7"
Monthly grouped bars
```

or another simple comparison chart showing:

```text id="d9d792"
Actual Revenue
vs
Target
```

Management should quickly see underperforming months.

---

# 16. Executive Visual — Branch Performance

Recommended table or ranked bar chart.

Metrics:

```text id="z40ve5"
Branch
Recognized Revenue
Target Achievement %
Gross Margin %
Revenue Growth %
```

This should help identify strong and weak branches.

---

# 17. Executive Visual — Receivable Aging

Recommended stacked bar or horizontal bar.

Buckets:

```text id="exuj95"
Not Due
1–30
31–60
61–90
90+
```

Measure:

```text id="f7fac1"
Outstanding Amount
```

---

# 18. Executive Visual — Delivery Performance

Recommended:

```text id="l6bzo7"
On-Time Delivery %
trend
```

by month.

This lets management see whether delivery performance is improving or deteriorating.

---

# 19. Executive Page Layout

Suggested conceptual layout:

```text id="h9tyas"
-------------------------------------------------------
| Revenue | Target | Margin | Overdue | Inventory | OTD |
-------------------------------------------------------

|           Revenue Trend                              |
-------------------------------------------------------

| Actual vs Target       | Branch Performance          |
-------------------------------------------------------

| Receivable Aging       | Delivery Performance        |
-------------------------------------------------------
```

The exact design can change during implementation.

---

# 20. Page 2 — Sales & Profitability

## Purpose

Help Sales and Management understand:

```text id="ethci0"
where revenue comes from
who is performing
which customers drive revenue
which products drive revenue
whether revenue is profitable
```

---

# 21. Sales KPI Cards

Recommended:

```text id="cnjxwq"
Recognized Revenue
Booked Sales
Revenue Growth %
Gross Margin
Gross Margin %
Average Order Value
Target Achievement %
```

---

# 22. Sales Trend

Recommended line chart:

```text id="j3pzxp"
Month
→ Recognized Revenue
```

Optional second line:

```text id="d0dmz0"
Booked Sales
```

This helps illustrate the difference between demand and financial recognition.

---

# 23. Branch Performance

Recommended ranked chart:

```text id="1da6cz"
Revenue by Branch
```

with secondary indicators:

```text id="82bwup"
Target Achievement
Gross Margin %
```

---

# 24. Salesperson Performance

Recommended table:

```text id="0tc9dr"
Sales Representative
Branch
Revenue
Target
Achievement %
Gross Margin %
Customer Count
```

Sort initially by:

```text id="p8cyca"
Recognized Revenue
```

or:

```text id="2uvp1v"
Target Achievement %
```

---

# 25. Salesperson Conditional Formatting

Example logic:

```text id="5t5pna"
Achievement >= 100%
→ positive indicator

80–99%
→ attention

<80%
→ under target
```

The dashboard should not become excessively colorful; use formatting mainly to direct attention.

---

# 26. Customer Analysis

Recommended:

```text id="aqao0j"
Top Customers by Revenue
```

table or bar chart.

Metrics:

```text id="av8gng"
Revenue
Gross Margin
Outstanding Receivables
Overdue Receivables
```

This allows management to see whether high-revenue customers are also high-risk debtors.

---

# 27. Customer Concentration

A useful visual:

```text id="wrnjtk"
Cumulative revenue contribution
```

or:

```text id="j1wicm"
Top 10 / Top 20 customer revenue share
```

This supports the business question:

> How dependent are we on a small number of customers?

---

# 28. Product Category Performance

Recommended:

```text id="pu3owh"
Revenue
+
Gross Margin %
```

by:

```text id="zjkopm"
Product Category
```

A category can have:

```text id="u0b9m8"
high revenue
but
low margin
```

which is an important management insight.

---

# 29. Product Drill-Down

Preferred hierarchy:

```text id="letnb3"
Category
    ↓
Subcategory
    ↓
Product
```

Users should be able to move from broad performance to specific SKU performance.

---

# 30. Branch Drill-Down

Preferred hierarchy:

```text id="cu843v"
Region
  ↓
Branch
  ↓
Sales Representative
  ↓
Customer
```

Not every visual needs the full hierarchy.

---

# 31. Sales Page Layout

Conceptually:

```text id="rr7pb4"
-----------------------------------------------------
| Revenue | Booked | Growth | Margin | Target | AOV |
-----------------------------------------------------

|          Revenue / Booked Sales Trend              |
-----------------------------------------------------

| Branch Performance    | Product Category            |
-----------------------------------------------------

| Salesperson Performance Table                      |
-----------------------------------------------------

| Top Customers / Customer Concentration             |
-----------------------------------------------------
```

---

# 32. Page 3 — Receivables

## Purpose

Help Finance and management monitor:

- open receivables
- overdue exposure
- aging
- customer payment risk
- collections

---

# 33. Receivable KPI Cards

Recommended:

```text id="jwxv6v"
Outstanding Receivables
Overdue Receivables
Overdue %
90+ Day Receivables
Collections This Period
Collection Rate %
```

---

# 34. Overdue Percentage

Formula:

```text id="vn3yde"
Overdue Receivables
/
Outstanding Receivables
```

This gives management an immediate sense of collection risk.

---

# 35. Aging Distribution

Primary visual:

```text id="ykbjcp"
Outstanding Value by Aging Bucket
```

Buckets:

```text id="885900"
Not Due
1–30
31–60
61–90
90+
```

A horizontal or stacked bar works well.

---

# 36. Receivables Trend

Recommended:

```text id="n3ueio"
Month-end Outstanding Receivables
```

and optionally:

```text id="flut5p"
Month-end Overdue Receivables
```

This requires the receivable snapshot fact designed earlier.

---

# 37. Largest Overdue Customers

Recommended table:

```text id="d3ipfn"
Customer
Branch
Outstanding
Overdue
Oldest Due Date
90+ Amount
Sales Representative
```

This should be one of the most actionable views.

---

# 38. Branch Receivable Risk

Recommended chart:

```text id="e73ada"
Overdue Amount by Branch
```

Optional metric:

```text id="au2nym"
Overdue % of Outstanding
```

This prevents the largest branch from appearing worst simply because it has the highest sales volume.

---

# 39. Sales vs Receivable Risk

A useful analytical view may compare:

```text id="4256hm"
Revenue
vs
Overdue Receivables
```

by customer or branch.

This can reveal cases such as:

> Revenue is growing, but collection quality is deteriorating.

---

# 40. Collection Trend

Recommended:

```text id="qpdcap"
Monthly Successful Payments
```

with comparison to:

```text id="t0fmoc"
Invoice Value
```

where the business definition supports it.

---

# 41. Receivables Detail Table

Detailed table:

```text id="36kut7"
Invoice Number
Customer
Invoice Date
Due Date
Outstanding Amount
Days Overdue
Aging Bucket
Branch
Sales Representative
```

This supports operational investigation.

---

# 42. Receivables Page Layout

```text id="ms54v7"
------------------------------------------------------
| Outstanding | Overdue | Overdue % | 90+ | Collections |
------------------------------------------------------

|             Aging Distribution                     |
------------------------------------------------------

| Receivable Trend     | Overdue by Branch           |
------------------------------------------------------

|            Largest Overdue Customers               |
------------------------------------------------------

|                 Invoice Detail                     |
------------------------------------------------------
```

---

# 43. Page 4 — Inventory

## Purpose

Help operations understand:

```text id="22vq9u"
how much stock is held
where it is held
what is running low
what is not moving
```

---

# 44. Inventory KPI Cards

Recommended:

```text id="7xjm77"
Inventory Value
Total Stocked SKUs
Low-Stock SKU Count
Slow-Moving SKU Count
Reserved Quantity
Available Quantity
```

---

# 45. Inventory by Warehouse

Recommended:

```text id="1zhfnb"
Inventory Value by Warehouse
```

or:

```text id="h84aew"
Available Quantity by Warehouse
```

depending on the question.

---

# 46. Inventory by Category

Recommended chart:

```text id="q8do75"
Inventory Value
by
Product Category
```

This can be compared against sales performance.

---

# 47. Low-Stock Products

Recommended table:

```text id="xu7hkj"
SKU
Product
Warehouse
Available Quantity
Reorder Level
Recent Sales
```

Sort by:

```text id="g4ywdm"
available quantity relative to reorder level
```

---

# 48. Slow-Moving Inventory

Recommended table:

```text id="cpke5v"
SKU
Product
Category
Warehouse
Quantity On Hand
Inventory Value
Days Since Last Sale
```

Sort by:

```text id="jr56mz"
Inventory Value
```

to identify capital tied up in slow-moving stock.

---

# 49. Inventory Trend

Recommended:

```text id="1cgtrc"
Inventory Value over Time
```

using daily or month-end snapshots.

This helps detect whether inventory is rising faster than revenue.

---

# 50. Inventory vs Sales

Optional analytical view:

```text id="lx9r6b"
Product Category
Revenue
Inventory Value
```

This may highlight:

```text id="8a5u99"
high inventory
+
low sales
```

categories.

---

# 51. Inventory Page Layout

```text id="i0tdke"
-----------------------------------------------------
| Value | SKUs | Low Stock | Slow Moving | Available |
-----------------------------------------------------

| Inventory Trend       | Inventory by Warehouse     |
-----------------------------------------------------

| Inventory by Category                              |
-----------------------------------------------------

| Low-Stock Products                                 |
-----------------------------------------------------

| Slow-Moving Inventory                              |
-----------------------------------------------------
```

---

# 52. Page 5 — Logistics

## Purpose

Help operations monitor delivery performance.

Key questions:

```text id="jx7owj"
Are shipments arriving on time?

Where are delays occurring?

Which carriers perform well?

Which shipments require attention?
```

---

# 53. Logistics KPI Cards

Recommended:

```text id="n3fw1x"
Total Shipments
Delivered Shipments
Open Shipments
Delayed Shipments
On-Time Delivery %
Average Delivery Time
```

---

# 54. On-Time Delivery Trend

Recommended:

```text id="xhg1u8"
Monthly On-Time Delivery %
```

This should expose deterioration or improvement over time.

---

# 55. Carrier Performance

Recommended table:

```text id="1mdxnw"
Carrier
Shipments
On-Time %
Average Delivery Days
Delayed Shipments
```

Sort initially by:

```text id="ecf7vu"
On-Time %
```

or shipment volume.

---

# 56. Delivery Performance by Region

Recommended:

```text id="yhv5fs"
On-Time %
by
Destination Region
```

This helps identify geographic delivery issues.

---

# 57. Delayed Shipment Detail

Recommended table:

```text id="n51m43"
Shipment ID
Order Number
Customer
Carrier
Dispatch Date
Expected Delivery Date
Days Late
Destination
Shipment Status
```

This is the main operational exception list.

---

# 58. Delivery-Time Distribution

Optional visual:

```text id="le442d"
Number of shipments
by
delivery duration
```

Example buckets:

```text id="z25kv6"
0–2 days
3–4 days
5–7 days
8+ days
```

---

# 59. Logistics Page Layout

```text id="yzre17"
--------------------------------------------------------
| Shipments | Delivered | Open | Delayed | OTD | Avg Days |
--------------------------------------------------------

|          On-Time Delivery Trend                       |
--------------------------------------------------------

| Carrier Performance    | Regional Performance          |
--------------------------------------------------------

|              Delayed Shipment Detail                  |
--------------------------------------------------------
```

---

# 60. Cross-Page Consistency

Common terms should be identical across pages.

For example, always use:

```text id="chzdco"
Recognized Revenue
```

rather than switching between:

```text id="oct7te"
Sales
Revenue
Net Sales
Invoice Sales
```

unless those metrics genuinely mean different things.

---

# 61. KPI Definitions

Dashboard metric names should map directly to the definitions in:

```text id="49yhxy"
03-reporting-requirements.md
```

The dashboard should not introduce alternative definitions.

---

# 62. Serving Views

Looker Studio should primarily connect to curated mart tables or views.

Recommended views:

```text id="g3qk8o"
vw_executive_summary
vw_sales_performance
vw_receivables
vw_inventory
vw_logistics
```

These can simplify the dashboard and reduce repeated query logic.

---

# 63. Executive Summary View Grain

The executive view may use:

```text id="4khc0w"
date
+
branch
```

or monthly equivalents depending on performance requirements.

Metrics should be aggregated from individual facts before being combined.

---

# 64. Avoid Direct Fact-to-Fact Joins

Looker Studio should not directly join:

```text id="a3x10s"
fact_sales
fact_inventory_snapshot
fact_receivables_snapshot
fact_shipments
```

at atomic grain.

This risks:

```text id="g66bfv"
fan-out
+
double counting
```

Prepare aggregated reporting models in BigQuery instead.

---

# 65. Dashboard Calculation Principle

Prefer:

```text id="wsqe0v"
BigQuery calculates business logic
```

and:

```text id="5y9wdv"
Looker Studio visualizes results
```

Examples calculated in BigQuery:

```text id="awbhhi"
gross_margin_pct
target_achievement_pct
aging_bucket
days_overdue
low_stock_flag
slow_moving_flag
on_time_flag
delayed_flag
```

---

# 66. Simple Dashboard Calculations

Looker Studio may still handle simple presentation calculations such as:

```text id="4u9l7q"
percentage formatting
display labels
simple ratios from already governed measures
```

where duplication of business logic is not a concern.

---

# 67. Performance Considerations

To keep dashboard performance acceptable:

- use mart tables, not staging
- partition large tables
- filter dates
- avoid overly complex blended data
- avoid unnecessary high-cardinality visuals
- pre-aggregate expensive calculations
- limit detailed tables to useful row counts

---

# 68. Default Date Filtering

Dashboard queries should always apply a meaningful date range.

Avoid scanning all three years unnecessarily when displaying:

```text id="7gyajo"
Current Month
```

performance.

---

# 69. Dashboard Extracts

If Looker Studio extract data provides a meaningful performance or cost advantage for a specific page, it may be considered.

However, direct BigQuery connection is preferred initially to preserve a simple architecture.

---

# 70. Chart Selection Principles

Use charts based on the management question.

## Trends

Use:

```text id="grqqvi"
line charts
```

## Ranking

Use:

```text id="jgx3tz"
horizontal bar charts
tables
```

## Actual vs Target

Use:

```text id="l7qejb"
grouped bars
variance indicators
```

## Composition

Use carefully selected:

```text id="et5q26"
stacked bars
```

Avoid excessive pie charts.

---

# 71. Tables Matter

Management dashboards should not avoid tables entirely.

Tables are appropriate for:

```text id="k89n3i"
top customers
salesperson performance
overdue invoices
low-stock products
delayed shipments
```

because users need actionable detail.

---

# 72. Avoid Dashboard Clutter

Each page should answer a focused set of questions.

Avoid:

```text id="61k919"
20 KPI cards
15 charts
too many filters
tiny unreadable visuals
```

A smaller number of meaningful visuals is better.

---

# 73. Visual Hierarchy

Recommended order:

```text id="qfmpov"
1. KPI cards
2. primary trend
3. major comparison
4. ranked performance
5. exception detail
```

This follows how management typically consumes information.

---

# 74. Management Exceptions

Important exceptions should be easy to identify.

Examples:

```text id="nr808s"
salesperson < 80% target

customer with 90+ overdue amount

SKU below reorder level

shipment overdue
```

The user should not need to inspect multiple charts to find them.

---

# 75. Conditional Formatting

Use conditional formatting mainly in tables.

Examples:

```text id="2srwth"
Target Achievement

>= 100%
positive

80–99%
attention

<80%
under target
```

```text id="79hg3k"
Receivable Aging

90+
high attention
```

Do not use excessive decorative formatting.

---

# 76. Currency Formatting

Indian management reporting may display:

```text id="0mwxyq"
₹
```

with values shown in:

```text id="xov293"
Lakhs
Crores
```

where appropriate.

Example:

```text id="op33jd"
₹12.4 Cr
```

rather than:

```text id="rdq3o3"
₹124,000,000
```

for executive cards.

Detailed tables may retain full values.

---

# 77. Percentage Formatting

Examples:

```text id="38yh8y"
96.4%
23.8%
91.2%
```

Avoid excessive decimal precision.

Typically:

```text id="7e65e3"
1 decimal place
```

is enough for management metrics.

---

# 78. Date Formatting

Examples:

```text id="vprkpw"
Oct 2026
06 Oct 2026
FY2026-27
```

depending on context.

---

# 79. Drill-Down Strategy

Drill-down should provide useful business progression.

Examples:

```text id="tz6r2d"
Region
→ Branch
→ Sales Representative
```

```text id="91zx24"
Category
→ Subcategory
→ Product
```

Avoid drill paths that simply expose database fields without business meaning.

---

# 80. Filter Interactions

Selecting:

```text id="e3zvyd"
Mumbai
```

should filter relevant visuals on the page.

Cross-filtering can be enabled where it makes analysis easier.

However, interactions should remain predictable.

---

# 81. Page-Level vs Report-Level Filters

Useful report-level filters:

```text id="1v1bp8"
Date
Branch
```

Page-specific filters:

```text id="1om90g"
Product Category
Warehouse
Carrier
Aging Bucket
```

This reduces unnecessary clutter.

---

# 82. Data Freshness Indicator

The dashboard should show:

```text id="mfd34a"
Data refreshed:
06 Oct 2026 07:00
```

or equivalent.

This helps management understand reporting freshness.

---

# 83. Data Status Indicator

If practical, the executive dashboard may also expose:

```text id="msz4bo"
Data Status:
READY
```

If one domain is stale:

```text id="zqhd4m"
Logistics data:
STALE
```

should be visible rather than silently showing outdated data.

---

# 84. Dashboard Trust

Management should be able to trust that:

```text id="4cydlm"
same revenue number
```

appears consistently across:

```text id="z03ga1"
Executive Overview
Sales Dashboard
Branch Analysis
Customer Analysis
```

for the same filters.

This consistency is one of the main reasons business calculations belong in the mart.

---

# 85. Dashboard QA

Before publication, validate:

```text id="4gsd6k"
KPI values
filters
period comparisons
drill-downs
totals
formatting
data freshness
empty-state behavior
```

---

# 86. KPI Reconciliation During QA

For each major dashboard KPI:

```text id="96oyit"
Looker Studio value
=
BigQuery serving model value
```

Examples:

```text id="qj7t78"
Recognized Revenue
Gross Margin
Outstanding Receivables
Inventory Value
On-Time Delivery %
```

---

# 87. Filter QA

Test combinations such as:

```text id="t9ml6h"
Branch = Mumbai
Month = September 2026
```

Then compare dashboard values against direct BigQuery queries.

---

# 88. Empty-State Testing

Some filter combinations may return no records.

The dashboard should not display misleading zeros when the correct meaning is:

```text id="ezda75"
No data
```

Examples:

- salesperson before joining date
- product category not sold in branch
- carrier not used in period

---

# 89. Dashboard Development Order

Recommended sequence:

```text id="k1m5vp"
1. Connect Looker Studio to mart
2. Validate core revenue metric
3. Build Executive Overview
4. Build Sales & Profitability
5. Build Receivables
6. Build Inventory
7. Build Logistics
8. Add common filters
9. Add drill-down
10. Perform reconciliation QA
11. Improve formatting
12. Capture portfolio screenshots
```

---

# 90. First Dashboard Milestone

Do not attempt all five pages immediately.

First create:

```text id="z5knr2"
Executive Overview
```

with only:

```text id="gnigpy"
Recognized Revenue
Gross Margin %
Target Achievement %
Overdue Receivables
Revenue Trend
Branch Performance
```

Validate these completely before expanding.

---

# 91. Portfolio Screenshots

The final GitHub portfolio should include dashboard screenshots.

Suggested screenshots:

```text id="sqbn8u"
Executive Overview

Sales & Profitability

Receivables

Inventory

Logistics
```

Store in:

```text id="zn2q3w"
dashboards/screenshots/
```

---

# 92. README Preview

The main README should eventually include:

```text id="l80klb"
1 architecture diagram
+
1 executive dashboard screenshot
```

This allows a visitor to understand the project quickly.

Detailed dashboard images can remain under:

```text id="8ltwu1"
dashboards/
```

or:

```text id="hx54lq"
docs/
```

---

# 93. Dashboard Documentation

Suggested repository structure:

```text id="2kem3u"
dashboards/
│
├── README.md
│
├── screenshots/
│   ├── executive-overview.png
│   ├── sales-profitability.png
│   ├── receivables.png
│   ├── inventory.png
│   └── logistics.png
│
└── validation/
    └── dashboard-reconciliation.md
```

---

# 94. Dashboard README

The dashboard-specific README can document:

```text id="t7k8fb"
dashboard pages
data sources
filters
KPI definitions
refresh schedule
known limitations
```

without making the main project README too long.

---

# 95. Case Study Value

The dashboard is the visible outcome of the project, but the portfolio should make clear that it sits on top of:

```text id="2spe6b"
source assessment
+
data ingestion
+
warehouse modelling
+
business definitions
+
data quality
+
reconciliation
+
monitoring
```

The dashboard should be presented as the final delivery layer, not the entire solution.

---

# 96. Example Management Story

The final synthetic dashboard should ideally support a narrative such as:

```text id="20fyhr"
Revenue is up 11% YoY.

However:

Delhi overdue receivables have increased materially.

Automation Components are driving sales growth.

Several older product categories now have slow-moving inventory.

One logistics carrier experienced weaker delivery performance during the last two months.
```

This demonstrates that the reporting solution supports actual management decisions.

---

# 97. Avoid Fake Insights

Business conclusions should be derived from generated data.

Do not write a dashboard conclusion first and manually force every number to match it.

The synthetic generation layer provides controlled trends, but the final analysis should still be calculated from the data.

---

# 98. Dashboard Definition of Done

The dashboard phase is complete when:

1. all five pages connect only to governed mart models
2. KPI definitions match the reporting requirements
3. recognized revenue is consistent across pages
4. financial metrics reconcile with BigQuery
5. common filters behave correctly
6. branch and product drill-downs work
7. receivable aging is accurate
8. inventory exceptions are visible
9. logistics exceptions are visible
10. reporting freshness is displayed
11. the dashboard performs acceptably
12. final portfolio screenshots are captured
13. dashboard documentation is added to GitHub

---

# 99. Dashboard Outcome

The dashboard converts the governed analytical platform into a usable management interface:

```text id="17gn7a"
Trusted Data
      ↓
Defined KPIs
      ↓
Management Views
      ↓
Exceptions
      ↓
Investigation
      ↓
Decision
```

The success of the dashboard should not be measured by the number of visuals.

It should be measured by whether a manager can quickly understand:

```text id="bctrxj"
what is happening
where the issue is
and what requires attention
```

---

## Next Step

The next document should be:

**`11-scheduling-monitoring.md`**

It will define:

- pipeline scheduling
- execution dependencies
- orchestration approach
- daily run sequence
- monitoring
- pipeline health
- freshness monitoring
- failure handling
- retry policy
- alerting
- recovery
- operational runbook
- cost-conscious deployment

After that, the main technical design will be complete and the project can move into the final GitHub documentation and consulting case-study layer.