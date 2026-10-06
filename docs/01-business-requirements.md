# Business requirements

**Reading time: 3–4 minutes. Status: planned design for a fictional engagement.**

## Client and reporting problem

Apex Industrial Components Pvt. Ltd. distributes industrial components to manufacturers, contractors, and maintenance businesses in India. The scenario assumes six branches, three warehouses, roughly 45 sales representatives, 2,500 customers, and 3,000 products. Annual revenue of ₹120–150 crore and 4,000–6,000 monthly orders establish the intended scale; they are not verified client figures.

Today’s assumed process combines PostgreSQL exports, departmental spreadsheets, and logistics records into weekly and monthly MIS. An analyst cleans files, performs lookups, reconciles totals, and refreshes reports. The assumed effort is 8–12 hours per cycle. Different teams use different definitions of sales, and corrections create multiple workbook versions.

The project should replace repeated preparation with a daily reporting workflow and an explicit exception review. Management should be able to explain a dashboard total using the underlying records and business rules.

## Management questions and ownership

| Area | Management question | Authoritative input |
|---|---|---|
| Sales | Are revenue and demand growing? Are targets being met? | ERP invoices/orders; Sales Operations targets |
| Profitability | Which products, customers, and branches generate margin? | Invoice lines and their costs; Finance adjustments |
| Receivables | What is outstanding, overdue, or deteriorating? | ERP invoices and successful payments |
| Inventory | Where is stock low or slow-moving? | ERP stock balances and product master |
| Logistics | Which deliveries are late, and which carriers need attention? | Logistics API |
| Planning | How do revenue and expenses compare with budget? | Finance budget and expense files |

Finance should confirm revenue, receivables, and adjustment rules. Sales Operations should confirm approved targets. Operations should confirm stock and delivery definitions. These are proposed responsibilities, not evidence of stakeholder sign-off.

## KPI definitions

The later source schema and mart design refine the early requirements: product revenue will use **invoice lines**, rather than an allocation from order values. The original documents also mix gross invoice value and pre-tax revenue; that difference must be resolved before SQL is published.

| KPI | Intended calculation and exclusions |
|---|---|
| Recognized revenue | Sum valid invoice-line net amounts before tax; exclude cancelled/void invoices. Partial cancellation treatment requires Finance confirmation. |
| Booked sales / average order value | Valid order value; average divides by valid order count. Exclude draft, rejected, and cancelled orders. Confirm the tax basis separately. |
| Revenue growth | Change against a comparable prior period, divided by prior revenue. Return null when the denominator is zero. |
| Target achievement | Recognized revenue / approved revenue target. Missing or zero targets are flagged, not treated as 0% achievement. |
| Gross margin / margin % | Revenue − invoiced quantity × transaction unit cost − allocated net cost adjustment; percentage uses total margin / total revenue. |
| Outstanding / overdue | Valid collectible invoice amount − successful payments applied; overdue means due date before the reporting date. |
| Inventory value / availability | On-hand quantity × standard cost; available quantity = on hand − reserved. |
| Low stock / slow moving | Available quantity at or below reorder level; stocked products with no sales in the preceding 90 days. |
| On-time delivery / delivery time | Delivered shipments on or before expected date / eligible delivered shipments; average elapsed days from dispatch to delivery. |
| Open / delayed shipments | Dispatched but neither delivered nor cancelled; delayed when expected delivery date has passed. |

Receivables aging uses Not Due, 1–30, 31–60, 61–90, and over 90 days, with boundaries that do not overlap. Collection rate remains provisional: the invoice cohort and payment-period attribution must be agreed before displaying a percentage.

## Reporting experience and freshness

Five Looker Studio pages are planned: Executive Overview, Sales & Profitability, Receivables, Inventory, and Logistics. Filters should follow the page’s purpose: date, financial year, branch, region, salesperson, customer segment, product category, warehouse, or carrier. Business reporting follows the Indian April–March financial year.

Sales, receivables, and inventory have a 24-hour freshness target. Logistics has a 4–6-hour target. Planning measures use the latest approved file for the relevant period. Snapshots show balances as of one selected date; transaction measures show activity over a date range.

## Acceptance criteria and boundaries

Acceptance will require automated integration of all three source types, source-to-mart reconciliation, visible quality failures, freshness within the target windows, and consistent KPI definitions. The less-than-one-hour manual effort target must be demonstrated through a timed reporting exercise.

The initial scope excludes streaming, forecasting, pricing optimization, complex payment allocation, and enterprise master-data management. Outstanding decisions include cancellation/tax treatment, adjustment signs, collection-rate attribution, and approved reconciliation tolerances. None should be presented as implemented or approved yet.

Detailed references: [client scenario](01-client-scenario.md) and [reporting requirements](03-reporting-requirements.md). Continue with [source design](02-source-design.md).
