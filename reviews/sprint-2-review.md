# Sprint 2 review

- Date: 2026-11-02
- Period: 2026-10-19 to 2026-11-09 (closed early)
- Status: complete

## Outcome

Sprint 2 made the Formula Insights API deployable, as planned
(`sprints/sprint-2-plan.md`). Every merge to `main` now publishes a scanned,
verifiable image; the schema is managed by migrations; and the importer paces
itself as ADR-006 requires. No cluster was created, as intended.

## Evidence reviewed

| Exit criterion | Evidence | Result |
| --- | --- | --- |
| Every scope item merged with CI green on `main` | formula-insights-api#16, #17, #18, #19 and platform-docs#9 closed; the latest `main` run (formula-insights-api#26) passed every job | Met. |
| A merge to `main` publishes the image and its provenance verifies | formula-insights-api#23, #24 | Met. `ghcr.io/fabriciolelis/formula-insights-api:33e4b22a8188a76c8f084df457b3fd9d30eeb4e6` is public (anonymous pull returns 200). `gh attestation verify` succeeds for its SLSA provenance and its CycloneDX SBOM, both signed by `api-checks.yml` on `main`. The image pushed is the one Trivy scanned, and a published tag is never overwritten. |
| The importer is paced as ADR-006 requires | formula-insights-api#21 | Met. At most 1 request per second and 450 per rolling hour (pyrate-limiter); a season range shares one limiter, so a backfill runs as one command. A `429` still fails the run, with a one-line reason. |
| The schema is created by migrations, and ADR-007 is accepted | platform-docs#23, formula-insights-api#25 | Met. Alembic with a baseline migration; the API and the importer refuse a missing or older schema and accept a newer one, so rollbacks keep working. CI applies the migrations to PostgreSQL and fails when the models change without a migration. |
| The Sprint 2 review is written | This document | Met. |

## Additional work

- **CI visibility:** every job writes a summary, a final table shows each
  job's status and key result, and Semgrep findings go to code scanning
  (formula-insights-api#24).
- **First real migration:** a driver who raced two cars in one Grand Prix (for
  example, Fangio in 1956) lost one of the results. formula-insights-api#26
  fixed it with migration `0002`, the first schema change after the baseline.
- **Documentation:** the ADRs became Markdown files (platform-docs#22),
  ADR-005 and ADR-006 gained dated update notes (platform-docs#21, #23), and
  the architecture diagram now shows PostgreSQL in the cluster, matching the
  AWS lab design (platform-docs#24).
- **Audit:** an audit of formula-insights-api on 2026-10-28 found six issues;
  one was fixed in #26, and five are open (see Carried over).
- **Sprint 3 groundwork:** a draft spec for the first GitOps deployment on
  local k3d (platform-gitops#6).

## Corrections made

- A new CRITICAL advisory in `certifi` (GHSA-6q48-6c7w-9gw2) failed the image
  scan with no code change. It was fixed by upgrading the lockfile
  (formula-insights-api#22), not by an allowlist entry.
- The plan's GHCR visibility risk did not happen: the package was public from
  the first publish (2026-10-23).

## Carried over

- formula-insights-api#28: run the container as a non-root user.
- formula-insights-api#29: the `.env` file is never loaded.
- formula-insights-api#30, #31, #32: cleanups from the audit.
- platform-gitops#2 and platform-gitops#6: GitOps decisions (ADR-008) and the
  deployment spec, which has open questions.
- platform-docs#10: rollback runbook and ingestion signals.
- Observability: a readiness endpoint, Prometheus metrics, and structured
  logs, needed before the first deployment because the SLOs depend on them.
- Pin `actions/checkout`, `actions/setup-python`, and the base image by digest
  (found by zizmor).

## Sprint 3 entry criteria (proposed)

Sprint 3 deploys Formula Insights to a local k3d cluster reconciled by Argo CD,
following the approved platform-gitops spec: PostgreSQL in the cluster, the
migration Job as a PreSync hook (ADR-007), the API, and the daily importer
CronJob (ADR-006), with SOPS-encrypted secrets. The readiness endpoint and the
rollback runbook must exist before that deployment. AWS remains out of scope.

## Lessons

1. **A security gate can fail with no code change.** The `certifi` advisory
   broke every build, `main` included, overnight. Fixing the dependency kept
   the gate meaningful; an allowlist entry would have hidden a real TLS risk.
2. **"Build once" needs a guard.** Publishing the scanned image avoided a
   rebuild, but only refusing to overwrite a published tag keeps tags
   immutable when a workflow is re-run.
3. **Decide before building.** ADR-007 settled the migration tool, ordering,
   and rollback rules first, so formula-insights-api#18 was implementation
   only, including the less obvious rule that older code must accept a newer
   schema.
