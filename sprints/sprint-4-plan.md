# Sprint 4 plan

- Period: 2026-11-18 to 2026-12-08 (3 weeks)
- Status: in progress
- Tracking: `Sprint 4` milestone in platform-docs, formula-insights-api, and platform-gitops

## Goal

Make the four Release 1 SLOs **measurable**: the local environment runs a
monitoring stack, deployed through GitOps, and each SLO has one documented
query that returns its current value. The local environment demonstrates the
measurement; it is not judged against the targets.

The detailed design is the spec in platform-docs#32 and the stack is chosen in
ADR-009. This plan fixes the scope, the order, and the exit criteria.

## Scope

| # | Item | Issue | Acceptance criteria (summary) |
| --- | --- | --- | --- |
| 1 | Observability stack decisions | platform-docs#33 | ADR-009 accepted (done, platform-docs#35). |
| 2 | Monitoring stack in the local environment | platform-gitops#15 | Prometheus, kube-state-metrics, and the blackbox exporter run as an Argo CD Application, come up after bootstrap with no extra step, and pass `Validate manifests`. |
| 3 | Ingestion SLOs | platform-gitops#16 | Ingestion success rate and refresh duration each have a recording rule and a documented query; manual imports are excluded and a day without a scheduled import counts as failed. |
| 4 | API metrics and latency SLO | formula-insights-api#38 | The API exposes OpenTelemetry HTTP metrics with a 0.5 s bucket; the image is promoted to `local`; the latency and non-5xx queries return values. |
| 5 | Synthetic probe and availability SLO | platform-gitops#17 | A probe calls `/ready` every 30 seconds; the availability query drops when PostgreSQL is down and recovers. |
| 6 | Sprint 4 verification and review | platform-docs#34 | A cluster rebuilt from nothing reaches Synced and Healthy including monitoring; the four queries return values; `reviews/sprint-4-review.md` records the evidence. |

## Order

Item 2 first, since every other item needs the stack. Items 3, 4, and 5 are
independent of each other and can be done in any order. Item 6 comes last.

## Decisions (ADR-009, 2026-11-18)

- **Stack:** plain Prometheus with kube-state-metrics and the blackbox
  exporter; no Helm, so ADR-008 is unchanged.
- **API metrics:** OpenTelemetry, following its HTTP semantic conventions.
- **Dashboards:** none in Sprint 4; reconsidered in Sprint 5.
- **Alerting:** not in Sprint 4; arrives with the first persistent environment.
- **Monitoring data on `local`:** disposable, like the cluster (spec #32).

## Optional, if time allows

Not part of the exit criteria:

- Cleanups formula-insights-api#30, #31, #32.
- Pin the CI actions by SHA and the base image by digest (zizmor findings).
- Remove the zlib upgrade from the API Dockerfile, once `python:3.12-slim`
  ships the fixed package.

## Out of scope

- The `dev` environment on AWS and any AWS resources (ADR-004).
- Separate database roles (ADR-007, first persistent environment).
- Dashboards, alerting, tracing, and log aggregation.
- Long-term storage or remote write of metrics.
- The consumer web app for portfolio visitors: Sprint 5, with an ADR-001
  amendment and the hosting question (ADR-004).

## Exit criteria

- Every scope item is merged with CI green.
- From no cluster, the documented commands bring every Argo CD application,
  including monitoring, to Synced and Healthy within 10 minutes, with no
  manual fixes.
- Each of the four SLOs has a recording rule and a documented query that
  returns a value on the rebuilt cluster.
- A manual import is shown to be excluded from the ingestion success rate.
- The Sprint 4 review is written, with the end-to-end output as evidence.

## Risks

- **Retention:** Prometheus keeps 15 days of data by default, shorter than
  the 30-day SLO window; the retention must be set explicitly. On `local` the
  window is nominal anyway, because a rebuilt cluster starts with no history.
- **Laptop resources:** about 700 Mi more at the limits (ADR-009) on top of
  Argo CD, PostgreSQL, and the API.
- **Beta packages:** the OpenTelemetry FastAPI instrumentation and Prometheus
  exporter are Beta; versions are pinned.
- **Hand-written configuration:** the scrape configuration and recording
  rules have no operator to check them; `promtool` can validate them in CI.
