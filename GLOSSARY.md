# Formula Platform

A platform that runs Formula Insights, a service serving historical and current Formula 1 data imported from Jolpica, delivered to Kubernetes through GitOps.

## Product

**Workload**:
One product the platform runs, with one owner, which may have several deployable parts. Formula Insights is the one workload of Release 1; Football Insights would be a second. Not the Kubernetes sense of a Deployment or a Job.
_Avoid_: Application, service (for the whole product)

**Portfolio visitor**:
An anonymous person who reads Formula Insights data in a web browser, without an account and without changing anything.
_Avoid_: User, customer, end user

**Public site**:
The static publication of Formula Insights data for portfolio visitors, showing the date its data was taken. It is not an environment.
_Avoid_: Frontend, web app, production

**Publication**:
One generation of the public site from the release the `local` environment runs, with the current season imported again; the site keeps its previous publication when one fails.
_Avoid_: Deploy, release, build

## Ingestion

**Ingestion**:
The capability of bringing Formula 1 data from Jolpica into the database, including its pacing, failure and signal rules (ADR-006).
_Avoid_: Sync, data load

**Import**:
One run of the importer over a range of seasons.
_Avoid_: Ingestion run, fetch

**Daily import**:
The scheduled import of the current season, once a day, in an environment. The import a publication runs is not a daily import.
_Avoid_: Incremental refresh, daily run, scheduled run

**Backfill**:
A manual, paced import of historical seasons, always run as a single season range.
_Avoid_: Historical load, full import

**Ingestion signal**:
The success or failure outcome of each daily import, with its reason; a day without a daily import counts as a failure. The ingestion success-rate SLO is measured over it; manual imports do not count.
_Avoid_: Import metric, ingestion status

**Re-import**:
A manual import of a past season to pick up a late correction.
_Avoid_: Refresh, reload

## Delivery

**Bootstrap**:
The one-time manual setup that lets a new cluster follow Git; after it, everything that runs in the environment comes from Git (ADR-008).
_Avoid_: Install, provisioning

**Environment**:
A named target that Formula Insights is deployed to from Git: `local` (k3d on the maintainer's machine) now, `dev` (the AWS lab) later. The public site is not an environment.
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
