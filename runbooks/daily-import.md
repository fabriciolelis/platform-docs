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
on the maintainer's machine and is not measured. From Sprint 4, metrics read
the same Job status (for example through kube-state-metrics) rather than a
new signal.

## Check the last daily imports

```bash
kubectl -n formula-insights get jobs -l app.kubernetes.io/component=importer
kubectl -n formula-insights logs job/<job name> | tail -1
```

Scheduled Jobs are named `formula-insights-importer-<timestamp>`; the CronJob
keeps the last 3 successful and 3 failed ones.

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
