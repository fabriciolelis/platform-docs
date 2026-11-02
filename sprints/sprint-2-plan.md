# Sprint 2 plan

- Period: 2026-10-19 to 2026-11-09 (3 weeks)
- Status: complete (see `reviews/sprint-2-review.md`)
- Tracking: `Sprint 2` milestone in platform-docs and formula-insights-api

## Goal

Make the Formula Insights API **deployable**: a published, verifiable container
image whose database schema is managed by migrations, and an importer that is
safe to run unattended. No cluster is created in this sprint.

This narrows the scope proposed in the Sprint 1 review: observability, the
GitOps decisions, the rollback runbook, and the local Argo CD cluster are not
part of Sprint 2.

## Scope

| # | Item | Issue | Acceptance criteria (summary) |
| --- | --- | --- | --- |
| 1 | Throttle Jolpica requests | formula-insights-api#16 | At most 1 request per second and 450 per rolling hour, shared across a run; tests without real sleeps; a `429` still fails the run (ADR-006). |
| 2 | Publish the image | formula-insights-api#17 | Every push to `main` that passes all checks pushes the scanned image to GHCR, tagged with the full commit SHA (no `latest`), with an SBOM and a build provenance attestation; pull requests never push. |
| 3 | Database migrations | platform-docs#9, formula-insights-api#18 | ADR-007 records the migration tool, who runs migrations, and their order relative to deployment; a baseline migration replaces `create_all`; CI applies the migrations to PostgreSQL. |
| 4 | Docstrings | formula-insights-api#19 | Google-style docstrings for all public code in `app/`, enforced by ruff in CI; no behavior changes. |
| 5 | Sprint 2 review | — | `reviews/sprint-2-review.md` with evidence for each exit criterion. |

## Out of scope

- Observability (readiness endpoint, Prometheus metrics, structured logs):
  Sprint 3, before the first deployment, because the availability and latency
  SLOs depend on it.
- Kubernetes, Argo CD, and GitOps manifests (platform-gitops#2): Sprint 3.
- Rollback runbook and ingestion signals (platform-docs#10): Sprint 3.
- AWS resources (ADR-004).
- Non-blocking cleanups from the formula-insights-api#14 review, unless time allows.

## Exit criteria

- Every scope item is merged with CI green on `main`.
- A merge to `main` publishes `ghcr.io/fabriciolelis/formula-insights-api:<commit-sha>`,
  and `gh attestation verify` succeeds for its provenance.
- The importer is paced as ADR-006 requires.
- The schema is created by migrations, and ADR-007 is accepted.
- The Sprint 2 review is written.

## Risks

- **GHCR package visibility:** a newly published package may start private and
  need its visibility set once; verify on the first publish.
- **Migrations touch the tests:** the test fixtures create tables with
  `create_all` today and must follow the new mechanism.
- **Tool choices inside items:** the migration tool and publishing details are
  decided in their issues and pull requests, with options presented before
  implementation.
