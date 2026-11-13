# Formula Platform

A platform that runs Formula Insights, a service serving historical and current Formula 1 data imported from Jolpica, delivered to Kubernetes through GitOps.

## Ingestion

**Ingestion**:
The capability of bringing Formula 1 data from Jolpica into the database, including its pacing, failure and signal rules (ADR-006).
_Avoid_: Sync, data load

**Import**:
One run of the importer over a range of seasons.
_Avoid_: Ingestion run, fetch

**Daily import**:
The scheduled import of the current season, once a day.
_Avoid_: Incremental refresh, daily run, scheduled run

**Backfill**:
A manual, paced import of historical seasons, always run as a single season range.
_Avoid_: Historical load, full import

**Ingestion signal**:
The success or failure outcome of each daily import, with its reason; the ingestion success-rate SLO is measured over it.
_Avoid_: Import metric, ingestion status

**Re-import**:
A manual import of a past season to pick up a late correction.
_Avoid_: Refresh, reload

## Delivery

**Environment**:
A named target that Formula Insights is deployed to from Git: `local` (k3d on the maintainer's machine) now, `dev` (the AWS lab) later.
_Avoid_: Stage, cluster

**Persistent environment**:
An environment whose data must survive cluster rebuilds and releases because it cannot be recreated by importing from Jolpica again. No persistent environment exists yet.
_Avoid_: Production-like, long-lived environment

**Promotion**:
Moving an environment to a new release by changing, in a reviewed pull request, which image that environment runs.
_Avoid_: Deploy, release (as a verb)

**Rollback**:
Restoring an environment's previous release by reverting its promotion in Git; the database schema is never reverted (ADR-007).
_Avoid_: Downgrade, undo, Argo CD rollback
