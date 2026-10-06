# Case study: management reporting automation

**Reading time: 3–4 minutes. Status: design-stage case study; implementation and outcomes remain to be demonstrated.**

## Engagement context

This portfolio project models a consulting engagement for Apex Industrial Components Pvt. Ltd., a fictional Indian B2B distributor. The scenario includes six branches, three warehouses, and separate operational, planning, and logistics sources. All data will be synthetic.

The assumed reporting process requires an analyst to combine ERP exports, Finance and Sales spreadsheets, and delivery records into weekly/monthly MIS. Preparation takes an assumed 8–12 hours per cycle. Spreadsheet versions and inconsistent definitions make it difficult for management to know which number to trust.

The target is less than one hour of routine manual intervention, daily financial/stock reporting, and several-hour logistics updates. These are design goals, not achieved savings or service levels.

## Solution and key decisions

The proposed flow is PostgreSQL + Excel/CSV + logistics REST API → Python ingestion → BigQuery staging, transformations, and management mart → Looker Studio. A lightweight scheduler will run the workflows, with control tables recording checkpoints, validation, reconciliation, and reporting readiness.

Five decisions shape the design:

| Decision | Why it matters |
|---|---|
| Separate booked orders from invoiced revenue | Demand and financial sales can differ because of cancellations or partial invoicing. |
| Model sales at invoice-line grain | Product revenue and margin need the actual invoiced items. |
| Preserve transaction-level cost | Later changes to product cost should not rewrite the cost basis of an old sale. |
| Keep daily balance snapshots | Current receivables and stock do not explain how balances changed over time. |
| Validate and reconcile before publication | A successful data load can still produce incorrect management totals. |

Batch processing fits the reporting need and keeps the operating model manageable. Streaming, full Type 2 dimension history, advanced forecasting, and complex payment allocation are deferred. BigQuery supplies the planned analytical storage; Python handles varied source interfaces; Looker Studio supplies the management reporting experience.

## What exists today

Repository inspection found a README, twelve long design documents, and repository metadata/configuration. The useful evidence is the written business scenario, source/schema proposal, architecture, analytical model, and quality/operations design.

There are no executable ingestion pipelines, schema or transformation scripts, generated source datasets, Docker deployment files, automated tests, scheduler outputs, or dashboard screenshots in the inspected repository. The documentation is a design artifact. It does not demonstrate working integration, measured reconciliation, production reliability, or delivery to a real client.

The older material remains available for reference; its completed-project language should not be used as implementation evidence.

## Implementation and evidence plan

| Milestone | Evidence needed before calling it complete |
|---|---|
| Reproducible sources | PostgreSQL DDL, generator configuration/seed, generated file/API examples, integrity checks |
| First ERP pipeline | Runnable extraction/load, control record, source/target comparison, safe retry and replay results |
| Multi-source integration | Approved file version handling, API pagination/retry results, documented order-reference mapping |
| Analytical mart | SQL, fact-grain checks, matching-basis invoice/payment reconciliation, tested snapshots and adjustments |
| Management reporting | Five working pages, screenshots, filter checks, KPI-to-mart comparisons, freshness indicators |
| Scheduled operation | Scheduled run records, stale/blocked behavior, recovery demonstration, measured runtime and costs |

Begin with one complete transaction pipeline, then invoices/payments, files/shipments, snapshots, and dashboard views.

Synthetic data should include realistic business patterns, such as branch growth, customer concentration, slower collections, stock risks, and carrier differences. These are planned simulation inputs. Claims such as “Delhi has the highest overdue balance” must wait until generated data and validated reports actually support them.

## Planned dashboard and evaluation

Five dashboard pages are planned: Executive Overview, Sales & Profitability, Receivables, Inventory, and Logistics. Each will show performance and relevant exceptions, with traceable KPI definitions and visible data freshness.

Measure reporting effort with a defined baseline exercise and the equivalent automated workflow, including exception review. Record source volumes, run duration, freshness, reconciliation results, rejected records, and manual review time. A synthetic exercise can demonstrate portfolio behavior; it cannot establish real client savings or adoption.

When implementation is complete, add actual results and link to the artifacts above. Until then, the outcome is a documented solution design and a clear delivery plan.

## Interview narrative and limitations

A defensible current explanation is: “I designed a reporting platform for a fictional distributor whose MIS depends on ERP exports, spreadsheets, and logistics data. I separated order demand from invoice revenue, chose invoice-line facts for product margin, and planned quality gates and daily balance snapshots. I’m now building the source environment and first ingestion pipeline; I have not yet measured reporting savings.”

Before SQL implementation, resolve the inconsistent gross-versus-net revenue basis in the older docs, partial cancellation treatment, adjustment signs/allocation, and collection-rate attribution. Historical inventory is available only after snapshots begin unless explicitly simulated. Type 1 dimensions limit historical attribute reporting. Production access controls and complete source change capture also require further implementation work.

Detailed references: [dashboard design](10-looker-studio-dashboard-design.md) and [original case study](12-final-case-study.md). Return to the [project overview](../README.md).
