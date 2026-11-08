# Sprint 3 plan

- Period: 2026-11-03 to 2026-11-24 (3 weeks)
- Status: active
- Tracking: `Sprint 3` milestone in platform-docs, formula-insights-api, and platform-gitops

## Goal

Deploy Formula Insights to a **local k3d cluster reconciled by Argo CD**, with
all of its desired state in `platform-gitops` (ADR-003). The API, its
PostgreSQL database, the migration step (ADR-007), and the daily importer
(ADR-006) run from Git, with SOPS-encrypted secrets. No AWS resources are
created (ADR-004).

The detailed design is the spec in `platform-gitops/SPEC.md`; this plan fixes
the scope, the decisions on its open questions, and the exit criteria.

## Scope

| # | Item | Issue | Acceptance criteria (summary) |
| --- | --- | --- | --- |
| 1 | API changes for deployment | formula-insights-api#33 | `python -m app.cli current` imports the current UTC year; `GET /ready` returns 200 when the database answers and 503 within 2 s when it does not; logs are structured JSON. `/health` stays a liveness check. |
| 2 | Non-root image | formula-insights-api#28 | The image runs as UID 10001 (`USER` in the Dockerfile); the pod settings also require non-root. |
| 3 | `.env` that nothing reads | formula-insights-api#29 | The setup steps no longer suggest a file the code ignores. |
| 4 | GitOps decisions | platform-gitops#2 | ADR-008 accepted: Argo CD bootstrap, SOPS with age, Kustomize overlays, environment names. |
| 5 | Local GitOps deployment | platform-gitops#7 | The spec's success criteria: a cluster rebuilt from nothing reaches Synced and Healthy; the migration runs before new API pods; CI validates the manifests without a cluster. |
| 6 | Rollback runbook and ingestion signal | platform-docs#10 | A GitOps rollback procedure with verification evidence, and the ingestion success and failure signal for the SLO. |
| 7 | Sprint 3 review | — | `reviews/sprint-3-review.md` with evidence for each exit criterion. |

## Decisions on the spec's open questions

- **GHCR visibility:** the package is public, so the cluster pulls without a
  secret (verified in Sprint 2).
- **Daily import:** 06:00 UTC. Sunday results are on Jolpica by Monday
  morning; a late result is picked up by the next run.
- **Database roles:** one role on the local cluster. The local cluster is not
  persistent, so ADR-007's migration and application roles arrive with the
  first persistent environment.
- **Backfill:** a documented `kubectl create job` command, not a committed
  Job manifest.

## Out of scope

- Prometheus metrics, a monitoring stack, and SLO measurement: Sprint 4. The
  readiness endpoint and structured logs land now, because the deployment
  needs them.
- Separate database roles (ADR-007, first persistent environment).
- Ingress, cert-manager, NetworkPolicies, and automated image updates.
- AWS resources (ADR-004).
- Cleanups formula-insights-api#30, #31, #32, unless time allows.

## Exit criteria

- Every scope item is merged with CI green.
- From no cluster, the documented commands bring the Argo CD applications to
  Synced and Healthy within 10 minutes, with no manual fixes.
- Merging an image-tag change runs the migration before any new API pod; a
  failing migration leaves the previous version serving.
- A rebuilt cluster with the same age key reaches the same state without
  re-encrypting any secret.
- ADR-008 is accepted and the rollback runbook exists.
- The Sprint 3 review is written, with the end-to-end output as evidence.

## Risks

- **Read-only root filesystem:** Python and the app may try to write at
  runtime; `PYTHONDONTWRITEBYTECODE` and a writable temporary directory may be
  needed.
- **SOPS in CI:** CI has no age key, so rendering must skip or stub the KSOPS
  generator (the spec asks the implementation to show how).
- **Losing the age key** makes every committed secret unreadable; the key needs
  a backup outside the repository.
- **Laptop resources:** k3d, Argo CD, and PostgreSQL together need several GB
  of memory.
