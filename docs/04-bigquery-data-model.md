# BigQuery data model

**Reading time: 4–5 minutes. Status: planned model; no SQL or BigQuery deployment evidence yet.**

## Layers and lineage

| Dataset | Purpose | Examples |
|---|---|---|
| `mis_staging` | Source-shaped data, stable identifiers, load metadata | `stg_invoices`, `stg_invoice_lines`, `stg_payments`, `stg_shipments` |
| `mis_transform` | Cleaned records, mappings, eligibility flags, calculations | `tr_invoice`, `tr_invoice_line`, `tr_payment`, `tr_shipment` |
| `mis_mart` | Conformed dimensions, facts, dashboard serving views | `dim_customer`, `fact_sales`, receivables reporting view |
| `mis_control` | Operational audit and reporting readiness | Pipeline runs, watermarks, file history, quality/reconciliation results |

Staging will preserve source IDs, `_source_updated_at`, load timestamp, run ID, and source identity. File records also need version/hash information. Transformations standardize types and statuses, resolve source references, and identify invalid records. Financial exclusions must be explicit and auditable.

Revenue lineage is invoice headers + invoice lines → eligible invoice lines → `fact_sales` → reporting view → dashboard. Finance adjustments join at a compatible product/month grain. Retain identifiers so any aggregate can be traced back to its contributing records.

## Dimensions

Core dimensions are `dim_date`, `dim_branch`, `dim_warehouse`, `dim_sales_rep`, `dim_customer`, and `dim_product`. Carrier and expense-category dimensions are optional if their attributes warrant separate tables. `dim_date` holds calendar periods and the Indian April–March financial year.

Use deterministic dimension keys derived from stable source identifiers, retain the source keys, and include an explicit unknown member where an optional mapping is missing. Missing mandatory financial references should fail a critical check rather than being hidden behind an unknown member.

The first release uses Type 1 dimensions: updated attributes replace older values. Keep available transaction-level branch and salesperson identifiers for historical attribution. Full Type 2 history is deferred; historical customer segments or product categories may therefore appear under their current classification. That limitation must remain visible.

## Facts and grains

| Fact | One row per | Main use |
|---|---|---|
| `fact_sales` | Invoice line | Recognized revenue, quantity, cost, margin |
| `fact_orders` | Sales order | Booked sales and order count |
| `fact_payments` | Payment | Successful collections |
| `fact_receivables_snapshot` | Snapshot date + invoice | Outstanding amount, overdue balance, aging |
| `fact_inventory_snapshot` | Snapshot date + warehouse + product | Stock, value, low/slow-moving flags |
| `fact_sales_target` | Month + salesperson | Approved revenue target |
| `fact_branch_budget` | Month + branch | Revenue and expense budget |
| `fact_operating_expense` | Month + branch + category | Operating costs |
| `fact_shipments` | Shipment | Status, delivery duration, on-time and delayed flags |

Uniqueness checks must use these grains. Separate orders and invoices because booked demand can differ from invoiced revenue. Keep payments separate because multiple receipts can apply to one invoice.

## Revenue, cost, and adjustments

`fact_sales` will retain invoice-line and invoice IDs, order references, dimension keys, invoiced quantity, line net revenue, transaction unit cost, allocated adjustment, and margin.

The intended product revenue basis is valid invoice-line `net_amount` before tax. Cost is `quantity × unit_cost` on that invoice line. Margin subtracts that cost and the applicable net cost adjustment. Aggregate margin percentage is **sum of margin / sum of revenue**, with null for a zero denominator; do not average individual line percentages.

The older documents compare invoice gross amounts with pre-tax line revenue. Reconciliation must compare matching measures: eligible line revenue to eligible header subtotal, and subtotal plus tax to gross invoice amount. Partial cancellation/credit treatment and any allocation to lines need a confirmed rule before implementation. Gross receivables and net sales should not be forced into one measure.

Aggregate Finance adjustments by product/month before joining to sales. The proposed allocation distributes each product/month adjustment in proportion to line revenue. Allocated amounts must reconcile to the original total within a documented rounding tolerance. Adjustment sign conventions, zero-revenue periods, and rounding residual handling are still open decisions. Keep unallocated adjustments visible.

## Receivables and inventory history

Aggregate successful payments by invoice before combining them with invoice headers; otherwise invoice values can multiply across payment rows. Calculate collectible gross balance after valid cancellations/credits and payments, using an explicit snapshot cutoff. Exclude failed or reversed receipts, and flag unexplained negative balances.

Aging uses snapshot date minus due date. Daily snapshots preserve the balance observed at that date. Historical corrections require deliberate reprocessing; current invoice status alone cannot recreate every historical balance or cancellation state.

Inventory snapshots capture available stock, on-hand value, and stock-risk flags each day. Persist the applicable standard cost with the snapshot so later cost changes do not rewrite old valuations. Slow-moving means stock exists but there were no sales in the preceding 90 days, subject to the confirmed sales-activity rule.

Neither balance fact should be summed across snapshot dates. A trend shows one balance per date; a headline balance uses one selected as-of date. Historical inventory before the first extract is unavailable unless a clearly labelled synthetic history is separately generated.

## Safe reporting joins

Aggregate revenue to month/salesperson before joining monthly targets, and aggregate branch results before joining branch budgets. Derive branch targets from salesperson targets unless an approved separate branch target is provided. Never add both sources together.

Build each executive measure at its own compatible grain before combining summaries. Directly joining invoice lines, payments, shipments, and stock balances can multiply rows. Logistics percentages use eligible delivered shipments; missing delivery dates should surface as coverage exceptions.

Looker Studio will read serving views for the five reporting pages. Business definitions belong in reusable SQL; the dashboard supplies filters and presentation. Monthly activity and daily balances must keep separate date semantics.

## Refresh and performance

Partition large transaction facts by their relevant business date and snapshots by snapshot date. Cluster only where repeated filters justify it; small dimensions and target tables need little tuning.

Track affected invoice IDs when either headers or lines change, including old cancellations. Rebuild affected product/month allocations when Finance adjustments change. Update payments and current receivables as collections arrive. A rolling reprocessing window may simplify some models, but must not discard older changes outside that window. Reruns replace the same fact keys or snapshot partition.

Before publication, verify grains, references, header/line totals, payments, target/budget totals, allocations, and dashboard aggregates. Prove the model using synthetic cases before claiming accurate financial reporting.

Detailed reference: [original BigQuery model](08-bigquery-data-model.md). Continue with [quality and operations](05-data-quality-operations.md).
