# Solution architecture

**Reading time: 4–5 minutes. Status: proposed architecture; no runtime is deployed.**

## End-to-end design

```text
PostgreSQL       Excel / CSV       Logistics REST API
     └───────────────┼───────────────────┘
                     ↓
             Python ingestion
                     ↓
             mis_staging
                     ↓
         mis_transform + quality checks
                     ↓
       candidate mis_mart + reconciliation
                     ↓
      approved reporting views → Looker Studio

mis_control: run history, watermarks, file versions,
             quality results, publication status
```

The design uses batch processing because daily financial reporting and several-hour logistics updates meet the scenario’s needs. Python handles source connections, extraction, validation, and loading. BigQuery SQL holds business calculations. Looker Studio consumes curated reporting outputs.

Four datasets separate responsibilities: `mis_staging` preserves source-shaped records and ingestion metadata; `mis_transform` cleans and integrates them; `mis_mart` holds facts, dimensions, and reporting views; `mis_control` records operational state. This keeps a dashboard discrepancy traceable to a load and source record.

The reporting layer should expose only validated results. The exact mechanism for building candidate outputs and promoting them without exposing a partial refresh must be implemented and tested; a status flag alone cannot protect readers of tables updated in place.

## Source-specific loading

| Source | Initial approach | Subsequent approach |
|---|---|---|
| Small ERP masters | Full load | Full refresh where inexpensive |
| Mutable masters and transactions | Historical load in batches | `updated_at` window, deduplication, keyed `MERGE` |
| Inventory | Current full extract | Daily snapshot, keyed by date/warehouse/product |
| Files | Validate approved version | Hash/version tracking; replace affected business scope |
| Shipments | Available history | Paginated updates or recent-window re-fetch; keyed upsert |

Shared Python components should cover configuration, authentication, logging, BigQuery loading, and control records. Source adapters keep their own parsing and failure behavior. Revenue, margin, aging, and target calculations belong in SQL rather than ingestion code.

## Incremental loading and recovery

For each transactional table, capture a fixed upper bound at the start of the run and read the previous successful watermark. The basic extraction window is:

```sql
WHERE updated_at > :watermark_from
  AND updated_at <= :watermark_to
```

Load the extract into a run-specific BigQuery batch table, validate it, deduplicate by source key and update timestamp, and merge it into persistent staging. Advance the ingestion watermark only after successful target loading and required ingestion checks. A failed load leaves the previous watermark available for retry.

Use a unique run ID and record source, window, row counts, target job, duration, and error details. Stable source keys make replays safe. Snapshot reruns replace or merge the same date’s grain, and approved file replays do not duplicate the same version.

The watermark pattern depends on reliable source timestamps. It does not automatically capture hard deletes or transactions committed later with an earlier timestamp. Before claiming complete extraction, define source update behavior and test an overlap/re-read or consistent extraction approach. The existing design’s recovery rule addresses failed loads; source concurrency needs separate validation.

Ingestion success and mart publication have different checkpoints. If staging succeeds but a downstream model fails, retain the staged data and rerun the affected transformations from their last successful state. Advancing ingestion must not cause those staged changes to be skipped by the mart.

## Files and APIs

File adapters will validate schema and business keys, normalize dates/codes/numbers, and retain rejection reasons. Content hashes identify repeat submissions. A new approved version updates its relevant period and scope; absent replacement files should retain existing approved values only where the reporting policy permits it.

The shipment adapter will follow every page, enforce timeouts, and retry temporary failures or rate limits with bounded backoff. Authentication failures and incompatible schemas need intervention. Advance its checkpoint only after the complete extraction is loaded successfully. Retain unmatched order references as visible exceptions rather than silently dropping them.

Money should remain fixed precision through extraction and BigQuery loading. Technical timestamps are UTC; Indian business dates and reporting periods must use an explicit timezone. Empty incremental extracts can be valid; unexpectedly empty full snapshots require investigation.

## Scheduling, deployment, and access

Start with a local, reproducible source environment and a containerized Python runner. A lightweight scheduler is sufficient: cron on a Linux/container host, or a simple supported scheduler on the chosen environment. Airflow is deferred until dependency complexity justifies it. Docker files and runnable commands are planned artifacts, not existing setup instructions.

The daily workflow loads reference data, transactions, files, and inventory; validates staging; builds transformations and candidate marts; reconciles totals; then publishes. Logistics will refresh independently every four hours, including its downstream model and publication checks. [Operations](05-data-quality-operations.md) describes proposed readiness and recovery rules.

Use environment-specific configuration and keep credentials out of Git. Local authentication should use a supported Google development flow; hosted execution should use a service identity with only the required access. Dashboard users should read curated outputs rather than raw staging. Detailed production IAM remains outside the first release.

## Cost and first delivery

Limit source extracts to necessary columns and update windows. Batch loads, partition large facts by relevant dates, filter reporting queries, and clean up temporary load tables. Measure query scans and runtime before adding further infrastructure.

The first engineering milestone is one ERP transaction pipeline: extract, load, validate, merge, record the run, and prove safe retry. Extend that foundation to invoices and lines, payments, files, API data, and daily snapshots before presenting the platform as end to end.

Detailed references: [original architecture](04-solution-architecture.md) and [Python ingestion](07-python-ingestion.md). Continue with [BigQuery model](04-bigquery-data-model.md).
