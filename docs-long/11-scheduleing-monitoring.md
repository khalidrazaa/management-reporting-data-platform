# 11 — Scheduling & Monitoring

## Purpose

This document defines how the Multi-Source MIS & Management Reporting Automation solution will run, monitor, recover, and surface failures in a production-like environment.

The objective is to ensure that the reporting platform operates as a managed workflow rather than a collection of manually executed scripts.

The operating model should provide:

- scheduled pipeline execution
- clear dependency sequencing
- retry handling
- pipeline status tracking
- data-freshness monitoring
- data-quality monitoring
- reconciliation monitoring
- failure visibility
- recovery procedures
- low operational overhead
- cost-conscious execution

---

# 1. Operating Principle

The solution should answer four operational questions at any time:

```text
What should have run?

What actually ran?

Did it succeed?

Is the resulting data trusted and fresh?
```

A successful scheduled job alone is not enough.

The platform must distinguish between:

```text
job completed
```

and:

```text
reporting data ready
```

---

# 2. Scheduling Architecture

The initial portfolio implementation will use a lightweight scheduling model.

Conceptually:

```text
Scheduler
   │
   ▼
Python Pipeline Runner
   │
   ├── PostgreSQL ingestion
   ├── File ingestion
   ├── API ingestion
   ├── Data quality
   ├── Transformations
   ├── Reconciliation
   └── Mart publication
         │
         ▼
      BigQuery
         │
         ▼
    Looker Studio
```

---

# 3. Initial Scheduling Approach

For the first implementation, use:

```text
cron
+
Dockerized Python pipeline
```

or:

```text
Python scheduler
```

This is sufficient for the size and complexity of the project.

The platform does not require a dedicated Airflow cluster.

---

# 4. Why Lightweight Scheduling

The project contains a manageable number of pipelines.

Using a large orchestration platform would add:

- infrastructure
- maintenance
- cost
- operational complexity

without materially improving the solution.

The design should remain capable of moving to a managed orchestrator later if requirements grow.

---

# 5. Possible Production Evolution

If the client later moves the pipeline runtime into Google Cloud, scheduling could evolve to:

```text
Cloud Scheduler
      ↓
Cloud Run Jobs
      ↓
BigQuery
```

For workflows with more complex dependencies, another orchestration layer could be introduced.

The code should remain portable enough to support this later.

---

# 6. Pipeline Frequency

Recommended initial schedule:

| Pipeline | Frequency |
|---|---|
| PostgreSQL Master Data | Daily |
| Sales Orders | Daily |
| Order Lines | Daily |
| Invoices | Daily |
| Invoice Lines | Daily |
| Payments | Daily |
| Inventory Snapshot | Daily |
| Sales Target File Check | Daily |
| Budget File Check | Daily |
| Expense File Check | Daily |
| Margin Adjustment File Check | Daily |
| Logistics API | Every 4 Hours |
| BigQuery Transformations | After Required Ingestion |
| Data Quality | During / After Each Stage |
| Data Mart Refresh | After Successful Validation |
| Dashboard | Reads latest published mart |

---

# 7. Daily Batch Window

Suggested main reporting batch:

```text
05:30
to
07:00
```

The objective is to have management reporting ready before the normal working day begins.

---

# 8. Example Daily Sequence

```text
05:30
PostgreSQL master ingestion starts

05:35
Transactional PostgreSQL ingestion starts

05:50
Inventory snapshot

06:00
File-source checks

06:10
Staging data-quality checks

06:20
Transformations

06:35
Mart refresh

06:45
Reconciliation checks

06:50
Publication status evaluation

07:00
Management reporting ready
```

These times are indicative and can change during implementation.

---

# 9. Logistics Schedule

The logistics API requires more frequent refresh.

Suggested runs:

```text
02:00
06:00
10:00
14:00
18:00
22:00
```

or every four hours.

This keeps delivery data reasonably current without creating unnecessary API traffic.

---

# 10. Dependency Sequence

The main batch should follow dependency order.

```text
Master Data
    ↓
Transactional Data
    ↓
External Files
    ↓
External API
    ↓
Staging DQ
    ↓
Transformations
    ↓
Mart Build
    ↓
Reconciliation
    ↓
Publication
```

---

# 11. Master Data Dependencies

Reference datasets should be available before dependent transactional validation runs.

Example:

```text
customers
products
branches
sales reps
```

should be available before validating:

```text
orders
invoice lines
targets
```

---

# 12. Independent Pipelines

Not all pipelines need to wait for one another.

For example:

```text
PostgreSQL ingestion
```

and:

```text
file-source checks
```

can run independently.

Likewise, the logistics API can run on its own schedule.

The orchestration should avoid unnecessary serialization.

---

# 13. Domain Dependencies

The platform should treat reporting domains separately where practical.

Example:

```text
Sales
Receivables
Inventory
Logistics
```

A failure in Logistics should not automatically invalidate Revenue reporting.

---

# 14. Pipeline Groups

The CLI can support grouped execution.

Examples:

```text
daily-postgres
```

```text
daily-files
```

```text
logistics
```

```text
transform
```

```text
publish
```

and an umbrella command:

```text
daily
```

---

# 15. Example Command

Conceptually:

```bash
python -m ingestion.cli daily
```

The command would coordinate the main daily workflow.

---

# 16. Scheduler Responsibility

The scheduler should remain simple.

It should primarily:

```text
start pipeline
```

The pipeline framework itself should handle:

- run ID creation
- watermarks
- logging
- retries
- validation
- failure recording

This avoids putting too much business logic into cron configuration.

---

# 17. Cron Example

Conceptual example:

```text
30 5 * * * run-daily-pipeline
```

Logistics:

```text
0 */4 * * * run-logistics-pipeline
```

Actual deployment scripts should wrap Docker or the Python runtime appropriately.

---

# 18. Pipeline State

The operational state should be stored in:

```text
mis_control
```

Primary tables include:

```text
pipeline_runs
pipeline_watermarks
data_quality_results
reconciliation_results
file_load_history
```

This becomes the operational control plane of the reporting solution.

---

# 19. Pipeline Run Monitoring

For each run, capture:

```text
pipeline_run_id
pipeline_name
source_system
target_table

started_at
completed_at
duration

status

rows_extracted
rows_loaded
rows_rejected

watermark_from
watermark_to

error_type
error_message
```

---

# 20. Pipeline Status Values

Recommended:

```text
STARTED
SUCCESS
FAILED
NO_CHANGE
PARTIAL_SUCCESS
```

`PARTIAL_SUCCESS` should be used sparingly.

Most incomplete financial or transactional loads should be considered failed.

---

# 21. Successful Pipeline

A pipeline can be marked:

```text
SUCCESS
```

only when:

- source extraction completes
- required validation passes
- target loading completes
- expected reconciliation passes
- watermark is safely committed

---

# 22. NO_CHANGE

Used when a pipeline runs successfully but finds nothing new.

Examples:

```text
No updated orders
No new approved target file
No changed shipment records
```

This is not an error.

---

# 23. Failed Pipeline

A pipeline should become:

```text
FAILED
```

when the result cannot be safely used.

Examples:

- source unavailable
- authentication failed
- required schema changed
- incomplete API extraction
- BigQuery load failed
- critical quality check failed

---

# 24. Pipeline Duration

Track:

```text
completed_at - started_at
```

for each pipeline.

Duration trends can reveal:

- increasing data volume
- database slowdown
- API slowdown
- warehouse query degradation

---

# 25. Duration Monitoring

Example:

```text
Normal sales-order pipeline:
3–5 minutes

Current execution:
18 minutes
```

This may warrant a warning even if the pipeline eventually succeeds.

---

# 26. Data Freshness Monitoring

Monitoring pipeline execution is not sufficient.

The platform must also monitor the age of actual data.

Example:

```text
pipeline ran successfully
```

but source returned stale information.

Freshness should therefore consider:

```text
MAX(source_updated_at)
```

or the relevant business timestamp.

---

# 27. Freshness SLAs

Initial reporting SLAs:

| Domain | Maximum Age |
|---|---:|
| Revenue | 24 hours |
| Orders | 24 hours |
| Receivables | 24 hours |
| Inventory | 24 hours |
| Logistics | 6 hours |
| Targets | Latest approved file |
| Budget | Latest approved file |

---

# 28. Freshness Status

Recommended states:

```text
FRESH
WARNING
STALE
```

Example:

```text
Revenue age:
18 hours

Status:
FRESH
```

```text
Revenue age:
27 hours

Status:
STALE
```

---

# 29. Freshness Control Table

A reporting-status table may contain:

```text
reporting_domain
last_successful_pipeline
latest_source_timestamp
published_at
expected_frequency_hours
current_age_hours
freshness_status
```

---

# 30. Publication Status

Each domain should also have a publication state.

Recommended values:

```text
READY
STALE
BLOCKED
```

---

# 31. READY

All required conditions are satisfied:

```text
ingestion successful
+
critical DQ passed
+
reconciliation passed
+
freshness within SLA
```

---

# 32. STALE

Data is technically valid but older than required.

Example:

```text
Logistics API unavailable for 10 hours
```

The previous validated data may remain available, but it must be clearly marked stale.

---

# 33. BLOCKED

Data should not be treated as trusted.

Examples:

```text
Revenue reconciliation failed
Duplicate target keys
Incomplete invoice ingestion
```

Affected mart publication should be stopped.

---

# 34. Domain-Level Publication

Example:

```text
Sales         READY
Receivables   READY
Inventory     READY
Logistics     STALE
```

This is preferable to hiding the entire dashboard due to a single unrelated source issue.

---

# 35. Monitoring Categories

Monitoring will cover:

```text
Pipeline Health
Data Freshness
Data Quality
Reconciliation
Source Availability
Volume Anomalies
```

---

# 36. Pipeline Health

Key questions:

```text
Did it run?
Did it finish?
How long did it take?
How many rows moved?
```

---

# 37. Data Quality Monitoring

Key metrics:

```text
Critical checks failed
Warning checks
Rejected rows
Unknown mappings
Quality failure rate
```

---

# 38. Reconciliation Monitoring

Key metrics:

```text
Revenue difference
Payment difference
Target difference
Budget difference
Margin-adjustment difference
```

Financial reconciliation should normally have zero unexplained difference.

---

# 39. Source Availability Monitoring

Track:

```text
PostgreSQL reachable
API reachable
Expected file present
BigQuery reachable
```

---

# 40. Volume Monitoring

Compare current ingestion volume to normal behavior.

Example:

```text
Today's invoices:
182

7-day average:
197
```

Normal.

Another example:

```text
Today's invoices:
14

7-day average:
201
```

Warning.

---

# 41. Baseline Volume

Initial volume anomaly detection can use:

```text
7-day rolling average
```

or:

```text
same weekday historical average
```

The simple rolling baseline is sufficient for the initial portfolio release.

---

# 42. Monitoring Queries

A monitoring layer should expose useful operational queries.

Example:

```sql
SELECT
    pipeline_name,
    status,
    started_at,
    completed_at,
    rows_loaded
FROM `mis_control.pipeline_runs`
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY pipeline_name
    ORDER BY started_at DESC
) = 1;
```

---

# 43. Failed Runs Query

Example:

```sql
SELECT
    pipeline_name,
    pipeline_run_id,
    started_at,
    error_type,
    error_message
FROM `mis_control.pipeline_runs`
WHERE status = 'FAILED'
ORDER BY started_at DESC;
```

---

# 44. Stale Domain Query

Conceptually:

```sql
SELECT
    reporting_domain,
    latest_source_timestamp,
    current_age_hours,
    freshness_status
FROM `mis_control.reporting_status`
WHERE freshness_status = 'STALE';
```

---

# 45. Quality Failure Query

Example:

```sql
SELECT
    pipeline_run_id,
    check_name,
    table_name,
    failed_records,
    severity
FROM `mis_control.data_quality_results`
WHERE status = 'FAIL'
ORDER BY executed_at DESC;
```

---

# 46. Monitoring Dashboard

A lightweight operational dashboard may be created separately from the management dashboard.

Possible tools:

```text
Looker Studio
```

or direct BigQuery inspection during early development.

---

# 47. Operational Dashboard Sections

Suggested:

```text
Pipeline Status
Data Freshness
Data Quality
Reconciliation
Recent Failures
```

---

# 48. Pipeline Health Cards

Possible KPI cards:

```text
Pipelines Run Today

Successful

Failed

Stale Domains

Critical DQ Failures
```

---

# 49. Pipeline Status Table

Suggested columns:

```text
Pipeline
Last Run
Status
Duration
Rows Loaded
Last Successful Run
```

---

# 50. Freshness View

Suggested:

```text
Domain
Latest Data
Age
SLA
Status
```

Example:

```text
Revenue
06 Oct 05:25
1.6 hrs
24 hrs
FRESH
```

---

# 51. Data Quality View

Suggested:

```text
Check
Dataset
Severity
Failed Rows
Status
Executed At
```

---

# 52. Reconciliation View

Suggested:

```text
Metric
Source Value
Target Value
Difference
Status
```

---

# 53. Alerting

The first portfolio version does not require a full enterprise alerting platform.

The architecture should nevertheless support notifications for:

```text
pipeline failure
critical DQ failure
reconciliation failure
data stale
```

---

# 54. Initial Alerting Option

A simple implementation can use:

```text
email notification
```

or structured logs inspected through monitoring.

The core monitoring framework should work even without notifications.

---

# 55. Future Alert Channels

Potential production integrations:

```text
Email
Microsoft Teams
Slack
PagerDuty
```

The notification channel should remain separate from pipeline logic.

---

# 56. Alert Severity

Recommended levels:

```text
INFO
WARNING
CRITICAL
```

---

# 57. Critical Alerts

Examples:

```text
Invoice pipeline failed

Revenue reconciliation failed

Critical data-quality check failed

Primary PostgreSQL source unavailable
```

---

# 58. Warning Alerts

Examples:

```text
Logistics data approaching SLA

Volume unusually low

Small number of shipment references unmatched

Optional file delayed
```

---

# 59. Avoid Alert Fatigue

The system should not send repeated alerts every few minutes for the same unresolved condition.

Possible later strategy:

```text
open incident
↓
suppress duplicate alert
↓
send recovery notification
```

This is not required for the first implementation but should guide the design.

---

# 60. Retry Strategy

Retries are appropriate for temporary technical failures.

Examples:

```text
network timeout
HTTP 503
HTTP 429
temporary PostgreSQL connection issue
```

---

# 61. Retry Backoff

Example:

```text
Attempt 1
↓
2 seconds

Attempt 2
↓
5 seconds

Attempt 3
↓
15 seconds
```

After the configured attempts:

```text
FAILED
```

---

# 62. Do Not Retry Permanent Errors

Examples:

```text
invalid API key
missing required file column
invalid SQL syntax
duplicate target business key
```

Retries would not solve these.

Fail quickly and record the cause.

---

# 63. Scheduled Retry vs Immediate Retry

Two retry patterns exist.

## Immediate Retry

Used for:

```text
temporary network failure
API server error
```

## Scheduled Re-Execution

Used when:

```text
source unavailable for longer period
file has not arrived
```

The next normal scheduler run may retry the source.

---

# 64. Watermark Recovery

Watermarks must only advance after successful ingestion.

Example:

```text
Run A
T1 → T2
SUCCESS

watermark = T2
```

Next run:

```text
Run B
T2 → T3
FAILED
```

Watermark remains:

```text
T2
```

The next successful run processes:

```text
T2 → T4
```

This prevents missing records.

---

# 65. Reprocessing

The system should support manual reprocessing when necessary.

Examples:

```text
rerun failed pipeline
reload source file
rebuild date partition
recalculate reporting period
```

---

# 66. Reprocessing by Date

Example:

```bash
python -m ingestion.cli postgres invoices \
    --from 2026-10-01T00:00:00Z \
    --to 2026-10-02T00:00:00Z
```

Exact CLI syntax may vary.

---

# 67. Mart Reprocessing

Transformations should support rebuilding:

```text
specific date
specific month
recent rolling period
```

rather than always rebuilding full history.

Example:

```text
rebuild current month fact_sales
```

after a late invoice correction.

---

# 68. Partition Replacement

For snapshot or date-partitioned facts, a safe recovery pattern is:

```text
rebuild affected partition
```

rather than inserting additional duplicate data.

---

# 69. Inventory Recovery

For a bad inventory snapshot:

```text
identify snapshot_date
↓
remove / replace partition
↓
rerun source extraction
↓
validate
```

---

# 70. File Recovery

If a corrected Finance file arrives:

```text
old file
↓
superseded

new file hash
↓
validate
↓
load
↓
replace affected reporting period
```

File history should preserve the fact that multiple versions existed.

---

# 71. Operational Runbook

A small runbook should document common failures and recovery actions.

Recommended location:

```text
docs/12-operations-runbook.md
```

or:

```text
docs/runbooks/
```

---

# 72. Runbook Scenario — PostgreSQL Unavailable

Symptoms:

```text
SourceConnectionError
```

Action:

```text
1. Verify PostgreSQL container / server status
2. Test connection
3. Confirm credentials
4. Confirm network access
5. Restart source service if appropriate
6. Rerun failed pipeline
7. Confirm watermark did not advance
```

---

# 73. Runbook Scenario — API Failure

Symptoms:

```text
HTTP 503
timeout
```

Action:

```text
1. Review retry attempts
2. Verify API availability
3. Check last successful watermark
4. Rerun when source is available
5. Confirm all pages loaded
6. Confirm logistics freshness status
```

---

# 74. Runbook Scenario — Authentication Failure

Symptoms:

```text
HTTP 401
```

Action:

```text
1. Verify credential configuration
2. Confirm API key validity
3. Update secret if required
4. Rerun pipeline
```

Do not repeatedly retry invalid credentials.

---

# 75. Runbook Scenario — File Schema Changed

Symptoms:

```text
required column missing
```

Action:

```text
1. Inspect source file
2. Compare with expected schema
3. Confirm whether change is intentional
4. Update mapping only after validation
5. Rerun file pipeline
```

Do not silently infer renamed financial columns.

---

# 76. Runbook Scenario — Duplicate Target Rows

Symptoms:

```text
sales target business key duplicated
```

Action:

```text
1. Reject file
2. Identify duplicate month + sales rep
3. Request / produce corrected source file
4. Rerun file ingestion
5. Validate total target
```

---

# 77. Runbook Scenario — Revenue Reconciliation Failure

Symptoms:

```text
source revenue != mart revenue
```

Action:

```text
1. Stop Sales domain publication
2. Compare source vs staging totals
3. Compare staging vs transform
4. Compare transform vs fact_sales
5. Identify missing / duplicated records
6. Correct issue
7. Rebuild affected period
8. Rerun reconciliation
9. Publish only after PASS
```

---

# 78. Runbook Scenario — Logistics Stale

Symptoms:

```text
Logistics freshness > 6 hours
```

Action:

```text
1. Check API pipeline status
2. Verify source availability
3. Review last successful watermark
4. Attempt rerun
5. If unavailable, retain last valid data
6. Mark Logistics domain STALE
```

---

# 79. Recovery Principle

Do not manually change analytical tables unless necessary.

Preferred recovery:

```text
fix source / pipeline logic
↓
rerun
↓
validate
```

rather than manually editing BigQuery rows.

This preserves repeatability.

---

# 80. Scheduling Failure Handling

If the scheduler itself fails:

```text
expected pipeline did not start
```

monitoring should detect:

```text
missing expected run
```

through freshness or last-run checks.

This is important because no pipeline log exists if the scheduler never starts the job.

---

# 81. Missing-Run Detection

Example:

```text
sales_orders expected:
daily by 06:00

current time:
08:00

no run started today
```

Status:

```text
MISSED_SCHEDULE
```

or equivalent operational alert.

---

# 82. Scheduler Logs

Scheduler execution should log:

```text
scheduled task
planned time
actual start time
exit code
```

This provides visibility outside the internal pipeline logs.

---

# 83. Container Execution

If running on the VPS, the scheduled process may execute the Dockerized pipeline.

Conceptually:

```text
cron
  ↓
docker run
  ↓
pipeline CLI
```

or:

```text
docker exec
```

depending on deployment design.

A fresh short-lived job container is preferable where practical.

---

# 84. Why Short-Lived Jobs

Batch pipelines do not need to remain running continuously.

Short-lived execution provides:

- lower resource usage
- simpler process lifecycle
- clean dependency environment
- clearer job status

---

# 85. Docker Compose

The development / VPS environment may contain:

```text
postgres-source
pipeline
mock-logistics-api
```

The pipeline service can be invoked on demand.

BigQuery remains external.

---

# 86. Cost Monitoring

The operating model should also consider BigQuery cost.

Monitor:

```text
bytes processed
query frequency
mart refresh volume
dashboard query behavior
```

The portfolio does not require sophisticated FinOps tooling.

---

# 87. BigQuery Query Cost

Transformations should use:

- partition filters
- incremental ranges
- selected columns
- limited reprocessing windows

Avoid:

```text
full historical scans every day
```

where incremental logic is practical.

---

# 88. Dashboard Cost

Looker Studio should query curated models.

Avoid dashboards that repeatedly execute complex joins across:

```text
staging
+
multiple atomic facts
```

This reduces both latency and BigQuery usage.

---

# 89. Retention of Operational Metadata

Keep pipeline and DQ history for the complete project period.

These tables are small and valuable for:

- troubleshooting
- operational reporting
- case-study evidence

---

# 90. Log Retention

Application log files should use rotation if stored locally.

Example:

```text
daily rotation
```

or size-based rotation.

This prevents a long-running VPS from accumulating unlimited log files.

---

# 91. Success Metrics for Operations

The final solution should demonstrate:

```text
High pipeline success rate
Fresh reporting data
Low unexplained reconciliation difference
Visible failures
Safe recovery
Minimal manual intervention
```

---

# 92. Manual Intervention Target

Normal daily operation should require:

```text
no manual action
```

unless:

- a critical failure occurs
- a source file requires correction
- a business exception needs review

The original target remains:

```text
< 1 hour manual intervention
per reporting cycle
```

primarily for review rather than preparation.

---

# 93. Operational Dashboard Definition of Done

Monitoring is complete when a user can see:

1. latest status of every pipeline
2. last successful execution
3. rows processed
4. failed pipelines
5. stale reporting domains
6. critical DQ failures
7. reconciliation status
8. recent error messages
9. data publication state
10. source freshness

without reading raw application logs.

---

# 94. Scheduling Definition of Done

Scheduling is complete when:

1. daily ERP pipelines start automatically
2. logistics pipeline runs every four hours
3. files are checked automatically
4. transformations follow ingestion
5. mart refresh follows validation
6. failed jobs do not advance watermarks
7. pipeline state is recorded
8. missed executions can be detected
9. reruns are safe
10. the workflow can run unattended

---

# 95. Monitoring Definition of Done

Monitoring is complete when:

1. every run has a status
2. failures contain actionable errors
3. freshness SLAs are measured
4. DQ results are visible
5. reconciliation failures are visible
6. reporting domains show READY / STALE / BLOCKED
7. unusual row volumes can be detected
8. operational history is retained
9. recovery can be performed without corrupting data
10. the management dashboard exposes only trusted or clearly marked stale information

---

# 96. Operating Model Summary

The complete operating flow becomes:

```text
Scheduler
    ↓
Ingestion
    ↓
Validation
    ↓
Transformation
    ↓
Reconciliation
    ↓
Publication Decision
    ↓
Management Reporting
```

with continuous operational visibility through:

```text
pipeline runs
+
watermarks
+
quality checks
+
freshness
+
reconciliation
+
alerts
```

---

# 97. Core Principle

A mature reporting solution should not depend on someone remembering:

```text
Did I run the script today?
```

The system itself should know:

```text
what should run
when it should run
whether it ran
whether it succeeded
whether the data is fresh
whether the data is trusted
```

That is the difference between a data script and an operational data solution.

---

## Next Step

The next document should be:

**`12-project-documentation-case-study.md`**

It will define the final GitHub and portfolio presentation, including:

- repository structure
- README final layout
- architecture diagram
- project screenshots
- implementation documentation
- run instructions
- design decisions
- business outcomes
- limitations
- future improvements
- consulting case-study narrative
- Malt / Upwork portfolio version
- recruiter-friendly summary

This will turn the technical project into a polished consulting portfolio asset.