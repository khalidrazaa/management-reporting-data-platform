# Data quality and operations

**Reading time: 3–4 minutes. Status: planned controls and runbook; no scheduled execution evidence yet.**

## Publication rule

A completed load does not prove that reporting data is trustworthy. Each reporting domain should pass structural, business, reconciliation, and freshness checks before new results become available to management.

Use three severities: **INFO** for observations, **WARNING** for limited issues that permit qualified reporting, and **CRITICAL** for failures that block the affected domain. The same issue can escalate when its volume or business impact changes. A late shipment or an unpaid invoice is a business exception; a delivery date before dispatch or a duplicate financial key is a data defect.

## Minimum checks

| Stage | Required controls |
|---|---|
| Extraction | Source reachable, required table/file present, complete API pagination, valid schema |
| Staging | Non-null and unique keys, expected types, accepted/rejected counts, valid references |
| Business rules | Valid statuses, due date ≥ invoice date, delivery date ≥ dispatch date, stock arithmetic |
| Mart | Unique fact grain, compatible joins, valid snapshot cutoff, valid target and adjustment mappings |
| Reconciliation | Source-to-staging counts/amounts; invoice headers to lines; source-to-mart financial totals |
| Reporting | Published values match mart, filters behave correctly, zero denominators and empty results display sensibly |

Reconciliation compares the same period, eligibility rules, currency, and tax basis. Pre-tax line revenue is compared with the corresponding header subtotal; gross invoice balances include their applicable tax and credits. Payment checks use successful receipts, and allocated adjustments must tie to approved file totals.

Record run ID, check ID, expected/actual values, difference, tolerance, severity, and result. File accounting should explain every row as accepted or rejected. Tolerances need documented approval: key duplication should have zero tolerance; monetary rounding may justify a small numeric tolerance. The old design suggests a small unmatched-shipment warning threshold around 0.1%, but this remains a proposed threshold to validate.

## Schedule and freshness

The proposed daily batch window is **05:30–07:00**, with the final scheduler timezone to be configured explicitly for Indian business reporting. These times are targets, not observed runtimes. The sequence is reference data → transactions/files/inventory → staging validation → transformations → candidate mart → reconciliation → publication.

Logistics runs every four hours and refreshes its reporting outputs after ingestion and validation. Sales, receivables, and inventory target a maximum age of 24 hours; logistics targets 4–6 hours. Planning measures require the latest approved file for their reporting period. No new file does not imply zero targets or expenses; distinguish an unchanged approved version from missing required period data.

A lightweight scheduler starts the Python runner. The runner manages dependencies, failure states, and checkpoints. Prevent overlapping runs for the same pipeline or snapshot scope. A logistics failure should not invalidate an independently validated sales domain.

## Monitoring and readiness

Record run start/end, status, row counts, rejection counts, duration, source window, target load job, and errors. Keep file-load history, quality results, reconciliation results, and last successful refresh in `mis_control`.

Each domain has a proposed reporting state:

- **READY:** validated output exists and is within its freshness target.
- **STALE:** the last validated output exists but is older than the target.
- **BLOCKED:** a critical failure prevents publication of new results.

Show last successful publication time and a visible stale/blocked message on relevant dashboard pages. Preserve the last validated output during failures. Implement candidate-build/promotion so an incomplete or invalid refresh cannot leak into live views; merely updating a status table is insufficient.

Monitoring should identify failed or missing runs, unusual counts or durations, reconciliation failures, stale data, and repeated API errors. Logs must exclude credentials. The notification destination and thresholds will be chosen during implementation.

## Recovery steps

1. Identify the failed run, affected source window, and reporting domains. Keep unvalidated output unpublished.
2. Fix the source, configuration, schema, or offending file. Retry temporary network/rate-limit failures with bounded backoff; investigate authentication and contract failures.
3. Rerun ingestion from its last successful checkpoint if loading failed. Advance the watermark only after successful loading and required ingestion checks.
4. If staging already succeeded, rerun affected transformations from their own checkpoint. Include changed headers, lines, payments, and adjustment periods.
5. Replace the same snapshot date or approved file scope safely. Reconcile and re-evaluate freshness before publication.

Backfills need an explicit date range and reason. Inventory history cannot be reconstructed from current stock alone. Historical receivables corrections need sufficient source history; do not regenerate old balances using today’s state without explaining the limitation.

## Evidence required before operation is claimed

Demonstrate duplicate-target rejection, an unknown reference warning, a late invoice cancellation, repeated shipment handling, safe snapshot rerun, failed-load watermark recovery, and blocked publication on revenue mismatch. Confirm that the dashboard still shows the last validated results after failure. Store run outputs and expected-versus-actual results before marking these controls implemented.

Detailed references: [quality framework](09-data-quality-frameework.md) and [scheduling/monitoring](11-scheduleing-monitoring.md). Continue with [case study](06-final-case-study.md).
