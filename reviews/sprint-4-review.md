# Sprint 4 review

- Date: 2026-11-24
- Period: 2026-11-18 to 2026-12-08 (closed early)
- Status: complete

## Outcome

Sprint 4 made the four Release 1 SLOs measurable, as planned
(`sprints/sprint-4-plan.md`, spec platform-docs#32). The local environment
runs the monitoring stack chosen in ADR-009, deployed from `platform-gitops`
like every other workload, and each SLO is a recorded query that returns a
value on a cluster rebuilt from nothing. The API exposes OpenTelemetry request
metrics, a synthetic probe calls `/ready` every 30 seconds, and the ingestion
signal is read from the Job status. The local environment demonstrates the
measurement; it is not judged against the targets. No AWS resources were
created.

## Evidence reviewed

| Exit criterion | Evidence | Result |
| --- | --- | --- |
| Every scope item merged with CI green | platform-docs#33; platform-gitops#15, #16, #17; formula-insights-api#38 closed, each with an evidence comment | Met. |
| From no cluster to every Application, monitoring included, Synced and Healthy within 10 minutes, no manual fixes | End-to-end run of 2026-11-24, below | Met: **172 s** with the README commands only (149 s in Sprint 3, before monitoring). The database, the migration hook, the API, and then the monitoring stack came up in that order. |
| Each SLO has a recording rule and a documented query that returns a value | End-to-end run; README "SLOs for Release 1"; `runbooks/daily-import.md` | Met. Availability 1, latency 1, ingestion success rate 1, refresh within 30 minutes 1 (18 s); the secondary non-5xx signal 1. All 9 rules healthy. |
| A manual import is excluded from the ingestion success rate | End-to-end run | Met. `import-now` carries the manual annotation and is not among the scheduled daily imports; the rate counts the daily import only (1 of 1). |
| The review is written with the end-to-end output as evidence | This document | Met. |

How each SLO is computed, and why:

| SLO | Recorded query | Decision |
| --- | --- | --- |
| API availability | `slo:api_availability:ratio_30d` | Successful probes over probes that ran. The probe runs in the cluster, so a full outage of the cluster is not visible (README). Verified with PostgreSQL scaled to 0: the probe failed within 30 s and the value fell from 1 to 0.615 (platform-gitops#17). |
| API latency | `slo:api_latency:ratio_30d` | Read requests (`/seasons/...`) below 500 ms; 5xx responses left out, so fast errors never improve latency. |
| Ingestion success rate | `slo:ingestion_success_rate:ratio_30d` | Successful daily imports over the 06:00 UTC runs due since the CronJob was created (at most 30), so a day without a daily import counts as failed. |
| Incremental refresh duration | `max_over_time(slo:daily_import_within_30m:bool[30d])` | Per daily import; the Job's 30-minute deadline makes every completed run fit, so the value shows the margin. |

The rules are unit-tested with `promtool test rules` in `Validate manifests`,
including the cases a short run cannot show: a missed day, the 30-day cap, and
no value before the first run is due. Each test was checked by breaking the
rule it guards.

<details>
<summary>End-to-end run, 2026-11-24 (excerpt)</summary>

```text
== platform-gitops main: c08f0cb feat: API request metrics and the latency SLO (#22)
== 2026-11-24T20:25:58Z start (t=0), README commands
argocd available t=86s
all 4 Applications Synced/Healthy t=172s
formula-insights/formula-insights-postgres-0 2026-11-24T20:27:34Z
formula-insights/formula-insights-migrate-hmn65 2026-11-24T20:27:51Z
formula-insights/formula-insights-api-7db4bc67dd-ltw7p 2026-11-24T20:28:22Z
monitoring/prometheus-5c8858549d-lb774 2026-11-24T20:28:24Z
== 2026-11-24T20:29:03Z daily import (scheduled-like: a Job of the CronJob without the manual annotation)
formula-insights-importer-e2e   Complete   1/1   18s
== 2026-11-24T20:29:21Z manual import
import-now                      Complete   1/1   17s
== 2026-11-24T20:29:42Z traffic round 1: 80 200, 20 404
== 2026-11-24T20:31:02Z traffic round 2: 80 200, 20 404
== targets: api-ready-probe, blackbox-exporter, formula-insights-api, kube-state-metrics, prometheus: up
slo:api_availability:ratio_30d = 1
slo:api_latency:ratio_30d = 1
slo:api_non_5xx:ratio_30d = 1
slo:ingestion_success_rate:ratio_30d = 1
max_over_time(slo:daily_import_within_30m:bool[30d]) = 1 (formula-insights-importer-e2e, 18 s)
ingestion:scheduled_job:30d = formula-insights-importer-e2e   (import-now excluded)
kube_job_annotations{annotation_cronjob_kubernetes_io_instantiate="manual"} = import-now
rules health: ok 9
monitoring memory: prometheus 35Mi, kube-state-metrics 12Mi, blackbox-exporter 7Mi
```

</details>

## Additional work

- **Validation tools in one place:** `scripts/install-tools.sh` installs the
  pinned kustomize, kubeconform, yq, and promtool with their checksums, for CI
  and local runs alike (platform-gitops#19). `Validate manifests` now checks
  the Prometheus configuration and runs the rule tests.
- **Image scan:** a new OpenSSL advisory without a Debian fix
  (CVE-2026-124709, QUIC stream handling) failed the scan; the API uses no
  QUIC, so it is accepted until 2027-01-15 like the earlier base-image cases.
- **Footprint:** the monitoring stack uses about 55 Mi of memory, far below the
  estimate in ADR-009 (about 700 Mi at the limits).

## Corrections made

- The OpenTelemetry FastAPI instrumentation still emits the old HTTP semantic
  conventions by default (milliseconds, raw paths); the stable ones that
  ADR-009 chose are switched on in code (`OTEL_SEMCONV_STABILITY_OPT_IN`), and
  tests fail without it.
- kube-state-metrics exports no Job annotations by default, and the manual
  annotation is the only way to tell manual imports apart: a Job created with
  `--from=cronjob` has the CronJob as owner too. The annotation is now allowed
  explicitly.
- The CronJob's history limit counts manual Jobs, so manual imports can delete
  the last daily Jobs from the cluster. Prometheus keeps their series, so the
  SLO is unaffected; the runbook notes it.
- Requests made before Prometheus first scrapes a new pod's counters are not
  counted by `increase()`, so a check right after a rollout needs a second
  round of traffic, as in the run above.

## Carried over

- formula-insights-api#30, #31, #32: cleanups from the October audit.
- Pin `actions/checkout`, `actions/setup-python`, and the base image by digest
  in formula-insights-api (found by zizmor).
- Remove the zlib patch from the Dockerfile once `python:3.12-slim` ships the
  fixed package; review the accepted base-image CVEs by 2027-01-15.
- Separate database roles for migrations and the application (ADR-007), with
  the first persistent environment.
- Alerting on the SLOs, with the first persistent environment (ADR-009).
- Before the API is exposed through an Ingress, block `/metrics` there or move
  it to a separate port.
- A validator rule that requires non-root and a read-only root filesystem, and
  a CI check for platform-docs (from the Sprint 4 retrospective).

## Sprint 5 entry criteria (proposed)

Sprint 5 is the consumer web app for portfolio visitors, decided on
2026-11-16. It needs an ADR-001 amendment (Release 1 has one workload) and an
answer to the hosting question: visitors cannot reach the local cluster, and
public hosting meets the cost controls of ADR-004. A dashboard tool for the
SLOs is reconsidered with it (ADR-009).

## Lessons

1. **Prove a 30-day rule with unit tests, not with waiting.** A missed day or
   the 30-day cap cannot be shown on a cluster rebuilt the same afternoon.
   Rule tests on synthetic series showed each case in seconds and caught every
   rule broken on purpose.
2. **Defaults lag the decision.** The instrumentation version chosen in
   ADR-009 still defaulted to the conventions the ADR rejected, and
   kube-state-metrics hid the one label the ingestion signal depends on. Both
   showed up only when the output was read, not when the manifests were.
3. **A scan can fail a change that did nothing wrong.** The build broke on an
   advisory published that day for an unchanged base image. Expiring
   exceptions keep that from becoming either a blocker or a permanent waiver.
