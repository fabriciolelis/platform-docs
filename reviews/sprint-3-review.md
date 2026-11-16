# Sprint 3 review

- Date: 2026-11-16
- Period: 2026-11-03 to 2026-11-24 (closed early)
- Status: complete

## Outcome

Sprint 3 deployed Formula Insights to a local k3d cluster reconciled by Argo CD,
as planned (`sprints/sprint-3-plan.md`). Everything that runs comes from
`platform-gitops`: the database, the migration hook, the API, and the daily
import, with secrets committed encrypted. A cluster rebuilt from nothing
reaches that state in under three minutes, and a broken release is stopped
before it reaches the API. No AWS resources were created.

## Evidence reviewed

| Exit criterion | Evidence | Result |
| --- | --- | --- |
| Every scope item merged with CI green | formula-insights-api#28, #29, #33; platform-gitops#2, #7; platform-docs#10 closed | Met. `Validate manifests` became a required check on platform-gitops `main`. |
| From no cluster to Synced and Healthy within 10 minutes, no manual fixes | End-to-end run of 2026-11-16, below | Met: **149 s** with the README commands only. On a cold start the database, then the migration hook, then the API came up in that order. |
| An image-tag change migrates before any new API pod; a failing migration leaves the previous version serving | platform-gitops#12; rollback drill #13 and #14 | Met. The migration Job ran before the API on every sync. In the drill, the hook failed, the sync stopped after 3 retries, and the same API pod kept serving. |
| A rebuilt cluster with the same age key reaches the same state without re-encrypting | End-to-end run | Met. The encrypted Secret is unchanged since platform-gitops#11. |
| ADR-008 accepted and the rollback runbook exists | platform-docs#28, #30 | Met. The runbook was written from the drill. |
| The review is written with the end-to-end output as evidence | This document | Met. |

All eight success criteria of the platform-gitops spec are met; the evidence
is in the comment that closed platform-gitops#7. Two details matter:

- **Readiness without the database:** with Argo CD's self-heal paused for the
  test, `/ready` returned 503 in 0.05 s, the API pod became NotReady after
  14 s, and it was not restarted. Re-applying `root-local` from Git restored
  self-heal, which brought the database back in 7 s.
- **Rollback:** reverting the broken promotion made the Application Synced and
  Healthy 27 s after the merge, with no rollout, because the broken release was
  never applied.

<details>
<summary>End-to-end run, 2026-11-16 (excerpt)</summary>

```text
[20:36:03] START k3d cluster create (commit f97b797)
[20:37:20] Argo CD available
[20:37:21] root-local applied
[20:38:33] ALL SYNCED/HEALTHY: formula-insights, formula-insights-postgres, root-local
[20:38:33] ELAPSED from cluster create: 149 s
[20:39:04] import-now Complete in 18 s, exit code 0
[20:39:09] GET /ready: {"status":"ready"}
[20:39:09] GET /seasons/2026/races: 23 races, first: "Australian Grand Prix"
schedule=0 6 * * * timeZone=Etc/UTC concurrencyPolicy=Forbid backoffLimit=2 activeDeadlineSeconds=1800 args=["python","-m","app.cli","current"]
[20:39:10] git status: '' ; secret.enc.yaml last changed in: 11ba1c5 (#11)
[20:39:34] PostgreSQL scaled to 0 (self-heal paused)
[20:39:36] GET /ready -> 503 in 0.045546s
[20:39:48] API pod ready=false after 14 s; restarts=0
[20:40:09] after 20 s more: ready=false, restarts=0, liveness /health -> {"status":"ok"}
[20:40:17] Recovered in 7 s: API ready=true, /ready -> {"status":"ready"}, postgres 1/1
```

</details>

<details>
<summary>Rollback drill, 2026-11-16 (excerpt)</summary>

```text
[20:46:54] API pod before: formula-insights-api-69cdd8446b-29bz2
[20:46:58] app: rev 703b3df Running, OutOfSync/Healthy
[20:51:10] app: rev 703b3df Running, retry 3, OutOfSync/Healthy
[20:54:33] app: rev 703b3df Failed: one or more synchronization tasks completed unsuccessfully (retried 3 times)
[20:54:34] migration job: failed=2; FAILED: Can't locate revision identified by 'does-not-exist'
[20:54:34] API pods now: formula-insights-api-69cdd8446b-29bz2 ready=true; drill annotation not applied
[20:54:35] GET /ready: {"status":"ready"}; races 2026: 23
[21:00:11] Rollback PR #14 merged at 20:59:48Z, commit 6fa7f04
[21:00:15] app: rev 6fa7f04 Synced/Healthy (last operation still Failed)
[21:07:27] manual sync: Succeeded after 10 s; migration job succeeded=1
```

</details>

## Additional work

- **Image scan:** new Debian advisories failed the scan. zlib is patched in the
  image until the base image catches up, and three unfixed libsqlite3 CVEs are
  accepted until 2027-01-15, since SQLite is not used by the image
  (formula-insights-api#34).
- **Ingestion signal:** the daily import runbook defines what the ingestion SLO
  counts: one scheduled Job per day including its retries, a missed day as a
  failure, and no manual runs (platform-docs#30).
- **Domain language:** a glossary for the platform (platform-docs#27), with
  "Bootstrap" added (#29); the engineering agent skills point every repository
  to it.

## Corrections made

- k3d's default K3s (1.32) is outside the Kubernetes versions Argo CD 3.5 is
  tested on and out of upstream support; the cluster is pinned to K3s 1.35.9.
- The spec asked for a failing migration "on a throwaway branch", but images
  are only published from `main`. The rollback drill was run on `main`
  instead, as a broken promotion and its revert.
- The manifest validator had three bugs that only real manifests revealed:
  CustomResourceDefinitions have no schema in the Kubernetes catalog, documents
  without containers produced empty lines, and yq's document separators were
  read as containers. Each fix came with a test case that failed before it.

## Carried over

- formula-insights-api#30, #31, #32: cleanups from the October audit.
- Pin `actions/checkout`, `actions/setup-python`, and the base image by digest
  in formula-insights-api (found by zizmor).
- Remove the zlib patch from the Dockerfile once `python:3.12-slim` ships the
  fixed package.
- Separate database roles for migrations and the application (ADR-007), with
  the first persistent environment.

## Sprint 4 entry criteria (proposed)

Sprint 4 makes the SLOs measurable: Prometheus metrics from the API, a
monitoring stack in the cluster (a new cluster component, so its choice comes
first), the synthetic `/ready` probe for availability, and the ingestion
signal read from the Job status. The `dev` environment on AWS stays behind the
cost controls of ADR-004.

## Lessons

1. **Validate against real content early.** The validator passed its own test
   cases and still failed on the first real manifests three times. Each real
   failure became a test case.
2. **Reconciliation hides failure tests.** Argo CD's self-heal restored the
   database faster than the readiness probe could notice, so the first outage
   test proved nothing. Testing failure behavior in a GitOps environment means
   pausing reconciliation on purpose, then restoring it from Git.
3. **A pre-sync migration turns a broken release into a non-event, with one
   loose end.** The API never changed, and the revert restored the desired
   state with no rollout, but the failed operation stayed until a new sync.
   The runbook now covers it.
