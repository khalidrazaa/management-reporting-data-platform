# Source design

**Reading time: 4–5 minutes. Status: proposed sources and schema; not yet built.**

## Source landscape

The portfolio will simulate an operational ERP, departmental planning files, and an external logistics service. PostgreSQL will hold normalized business transactions. Python will extract those records without moving heavy management analysis into the source database. BigQuery will provide analytical storage and reporting history.

| Source | Proposed owner | Content | Load pattern |
|---|---|---|---|
| PostgreSQL ERP | Operations / IT | Masters, orders, invoices, payments, stock | Initial full load; incremental transactions |
| Sales targets Excel | Sales Operations | Monthly salesperson targets | New approved file version |
| Budget and expenses Excel | Finance | Branch budgets and monthly expenses | New approved file version |
| Margin adjustments CSV | Finance | Product/month cost adjustments and rebates | New approved file version |
| Mock logistics REST API | Simulated logistics provider | Shipment status and delivery dates | Poll every four hours |

No source environment, generator, or API exists in the current repository. Endpoint names and volumes below are design choices.

## PostgreSQL entities and relationships

| Table | Grain / key | Important relationships or measures |
|---|---|---|
| `branches` | Branch / `branch_id` | Unique branch code; city, state, region |
| `warehouses` | Warehouse / `warehouse_id` | Belongs to a branch |
| `sales_representatives` | Salesperson / `sales_rep_id` | Employee code, branch, optional manager |
| `customers` | Customer / `customer_id` | Customer code, branch, salesperson, segment, credit terms |
| `products` | Product / `product_id` | Unique SKU, category, standard cost, reorder level |
| `sales_orders` | Order / `order_id` | Customer, branch, salesperson, status, date, amounts |
| `sales_order_lines` | Order line / `order_line_id` | Order, product, quantity, price, discount, unit cost |
| `invoices` | Invoice / `invoice_id` | Order, customer, branch, invoice/due dates, status, gross amount |
| `invoice_lines` | Invoice line / `invoice_line_id` | Invoice, product, optional order line; net amount and transaction cost |
| `payments` | Payment / `payment_id` | One invoice per payment; amount, date, status |
| `inventory` | Warehouse + product | Current on-hand, reserved, and available quantities |

Orders can have multiple lines and invoices. An invoice can have multiple lines and payments. Invoice lines are essential: product revenue and margin should reflect what was invoiced, including partial invoicing, rather than assuming that every ordered item became revenue.

`unit_cost` on transaction lines preserves the cost at the time of sale. Using today’s product standard cost for an old sale would distort historical margin. Product standard cost remains the proposed basis for current inventory valuation.

Each entity has a stable primary key; business codes such as SKU, order number, and invoice number should be unique. Invoice-line sequence is unique within an invoice. Foreign keys enforce the relationships above. Monetary fields use fixed-precision numeric types, and technical timestamps use `TIMESTAMPTZ` normalized to UTC.

Source constraints should reject invalid quantities, negative basic amounts, and due dates before invoice dates. Available inventory must equal on hand minus reserved. Financial totals must reconcile between headers and lines with tax handled separately. Exact cancellation rules remain an implementation prerequisite.

## Change capture and source limits

Mutable tables will have `created_at` and `updated_at`, with `updated_at` maintained on every relevant change. Incremental extraction will use the update timestamp, so a later cancellation of an older invoice can be captured. Small branch and warehouse masters can be fully refreshed; customers, products, salespeople, and transactions can use incremental loads.

Inventory is current state, approximately 9,000 warehouse/product combinations at the assumed scale. A full daily extract will create BigQuery snapshots. The source table cannot prove historical stock levels, so the project must not imply that three years of inventory history can be recovered from it.

The initial schema simplifies payments to one invoice per payment. Allocation of one receipt across multiple invoices is deferred. Timestamp extraction also cannot detect hard deletes by itself; deletion handling must be settled before claiming complete change capture.

## File contracts

| File | Business grain | Required identifiers / values |
|---|---|---|
| `sales_targets_2026.xlsx` | Month + salesperson | Employee code, branch code, revenue target |
| `branch_budget_2026.xlsx` | Month + branch | Branch code, revenue budget, expense budget |
| `monthly_operating_expenses.xlsx` | Month + branch + expense category | Branch code, category, amount |
| `product_margin_adjustments.csv` | Month + SKU + adjustment type, provisionally | SKU, type, signed amount, reason |

Validate columns, types, mandatory fields, duplicate keys, periods, and references before accepting a version. Record filename, content hash, receipt/load time, and accepted/rejected counts. Revised files replace the relevant approved scope rather than accumulating duplicate targets. If multiple adjustments of the same type are legitimate, introduce an adjustment ID or documented aggregation rule before enforcing uniqueness.

## Logistics API contract

The mock service will expose shipment records through `/shipments`, with pagination and an `updated_since` filter where supported. Records include shipment ID, ERP order reference, dispatch date, expected/actual delivery dates, status, carrier, destination, and update timestamp.

The early assessment maps the API’s `order_id` to ERP `order_number`; the implementation must make that mapping explicit instead of treating a business code as a database ID. Repeated shipment responses are upserted by shipment ID. A recent-window re-fetch is the fallback if incremental filtering is unavailable.

## Synthetic data plan

Use a fixed seed, configurable volumes, and development/portfolio profiles. The planned history is January 2024–September 2026, with roughly 150,000–200,000 orders and up to about one million order lines. These are generation targets, not existing record counts.

Generate masters before dependent transactions, then files and API data. Include branch/category growth, customer concentration, partial payments, overdue invoices, low stock, and differing carrier performance. Add controlled integration defects after generating a valid base dataset: duplicate targets, unknown SKUs, unmatched shipments, and late updates. Validate relationships, amounts, and distributions. Any analytical findings must come from the resulting data.

Detailed references: [source assessment](02-source-system-assessment.md), [PostgreSQL schema](05-postgresql-source-schema.md), and [synthetic data](06-synthetic-data-generation.md). Continue with [architecture](03-solution-architecture.md).
