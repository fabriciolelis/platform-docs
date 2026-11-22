# Daily import runbook

## Purpose

Defines the ingestion signal that the ingestion SLOs and future alerts are
measured on, and how to check and recover a failed daily import (ADR-006).

## The ingestion signal

| Question | Rule |
| --- | --- |
| What is one daily import? | One Job created by the `formula-insights-importer` CronJob, including its job-level retries (`backoffLimit: 2`). A day whose first attempt fails and second succeeds is a success. |
| Success | The Job ends `Complete`: the importer exited 0. |
| Failure | The Job ends `Failed`: every attempt exited non-zero, including standings not updated (ADR-006), or the run passed its 30-minute deadline. |
| A day without a daily import | Counts as a failure, whether the cluster was down or the CronJob suspended: the data is as stale as after a failed run. |
| Manual imports | Not counted. Jobs created with `kubectl create job --from=cronjob/...` (an import run now, a backfill, a re-import) carry the annotation `cronjob.kubernetes.io/instantiate: manual`. |
| Outcome and reason | The Job status is the outcome. The importer's last log line, the JSON record with `"message": "Import summary"`, gives the reason: `outcome`, `seasons_imported`, `counts`, and, on failure, `failed_season` and `reason`. |

The SLOs are measured only in environments that are always on. `local` runs
on the maintainer's machine and is not measured. Metrics read the same Job
status through kube-state-metrics rather than a new signal (see
[Read the ingestion SLOs](#read-the-ingestion-slos)).

## Check the last daily imports

```bash
kubectl -n formula-insights get jobs -l app.kubernetes.io/component=importer
kubectl -n formula-insights logs job/<job name> | tail -1
```

Scheduled Jobs are named `formula-insights-importer-<timestamp>`; the CronJob
keeps the last 3 successful and 3 failed ones.

## Read the ingestion SLOs

Prometheus records both SLOs from kube-state-metrics (ADR-009); the rules and
their tests are in `platform-gitops/applications/monitoring/`. Open Prometheus
with a port forward and run the queries at `http://localhost:9090`:

```bash
kubectl -n monitoring port-forward svc/prometheus 9090:9090
```

| SLO | Query | Reads as |
| --- | --- | --- |
| Ingestion success rate (at least 96%) | `slo:ingestion_success_rate:ratio_30d` | Successful daily imports over the daily imports due in the last 30 days. |
| Incremental refresh duration (30 minutes) | `max_over_time(slo:daily_import_within_30m:bool[30d])` | One series per daily import that completed: 1 when it finished within 30 minutes. `ingestion:daily_import_duration:seconds` gives the duration. |

How the success rate applies the signal:

- **Due imports** are the 06:00 UTC schedule times since the CronJob was
  created, at most 30. A day without a daily import is due but not successful,
  so it counts as failed. The rate is never above 100%: a daily import that the
  CronJob catches up after a stopped cluster counts once.
- **Manual imports** are excluded by their annotation; kube-state-metrics
  exports it because of `--metric-annotations-allowlist`.
- **Before the first 06:00** after the CronJob is created, nothing is due and
  the query returns no value.
- A daily import that passes its 30-minute deadline is stopped and ends
  `Failed`, so it lowers the success rate and never appears in the duration
  query.

Two limits of the `local` environment: Prometheus keeps its data in the pod,
so a restart or a configuration change starts the 30 days over, and the
CronJob's history limit also counts manual Jobs, so a few manual imports can
delete the last daily Jobs from the cluster. The recorded series stay in
Prometheus in that case.

## Recover a failed daily import

1. Read the reason in the `Import summary` line.
2. Fix the cause if it is ours. A Jolpica outage or a `429` needs no fix: the
   importer never retries on its own, by design (ADR-006).
3. Run the import again; it is idempotent:

   ```bash
   kubectl -n formula-insights create job --from=cronjob/formula-insights-importer import-now
   kubectl -n formula-insights wait --for=condition=Complete job/import-now --timeout=1800s
   ```

   A manual run does not turn the failed day into a success for the SLO; it
   only brings the data up to date. The next scheduled run does the same.

A failed daily import is not a reason to roll back a release; see
[`rollback.md`](rollback.md) for when to roll back.
