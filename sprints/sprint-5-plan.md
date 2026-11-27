# Sprint 5 plan

- Period: set with the `Sprint 5` milestone
- Status: planned
- Tracking: `Sprint 5` milestone in platform-docs and formula-insights-api

## Goal

Give portfolio visitors a **public site** to read Formula Insights data, up
when the maintainer's laptop is off and no more than one day old, at no cost.

The design was settled on 2026-11-28 and is recorded in ADR-010, with
amendments to ADR-001 (one workload), ADR-004 (hosting outside AWS), and
ADR-009 (no dashboard tool yet). This plan fixes the scope, the order, and the
exit criteria.

## Scope

| # | Item | Repository | Acceptance criteria (summary) |
| --- | --- | --- | --- |
| 1 | Sprint 5 decisions | platform-docs | ADR-010 accepted; ADR-001, ADR-004, and ADR-009 amended; README and glossary updated. |
| 2 | Site renderer | formula-insights-api | `web/` renders, with Python and Jinja from the API's HTTP endpoints: the season list, each season's races, each race's results, driver and constructor standings per season, and an about page linking the repositories. Every page's footer credits Jolpica F1 and CC BY-NC-SA 4.0 and shows the date of the data. No JavaScript. |
| 3 | Site verification in CI | formula-insights-api | CI renders the site against a fixture API and checks every internal link in the output; a contract change that breaks a page fails the pull request. |
| 4 | Historical data dump | formula-insights-api | A manual workflow runs a backfill from 1950 in CI, paced under ADR-006, and attaches the dump to a GitHub Release under CC BY-NC-SA 4.0. |
| 5 | Daily publication | formula-insights-api | A scheduled workflow reads the image `local` runs from platform-gitops, restores the dump, imports the current season with that image, renders the site, and publishes it to GitHub Pages. A failed run leaves the previous publication online. |
| 6 | Sprint 5 verification and review | platform-docs | The exit criteria below are shown with evidence in `reviews/sprint-5-review.md`. |

## Order

Item 1 first. Items 2 and 4 are independent. Item 3 follows item 2. Item 5
needs items 2 and 4. Item 6 comes last.

## Decisions (ADR-010, 2026-11-28)

- **Static, not live:** the public site is HTML generated from the API; no
  cluster, API, or database is public.
- **Generated in CI:** a daily publication in GitHub Actions, independent of
  the laptop; its import of the current season is not a daily import and does
  not count in the ingestion signal.
- **Same release as `local`:** a publication uses the image `local` runs, so a
  promotion reaches the site at the next publication.
- **Hosting:** GitHub Pages at the default address, USD 0, no AWS.
- **Data licence:** Jolpica's data is CC BY-NC-SA 4.0; the site and the dump
  credit it and use the same licence.
- **No Grafana:** reconsidered with alerting in the first persistent
  environment.

## Optional, if time allows

Not part of the exit criteria:

- Items carried over from Sprint 4 (`reviews/sprint-4-review.md`): cleanups
  formula-insights-api#30, #31, #32; pinning CI actions and the base image by
  digest; removing the zlib patch; a validator rule for non-root and a
  read-only root filesystem; a CI check for platform-docs.
- Read the terms and format of Jolpica's own database dumps, as a possible
  replacement for the backfill in item 4.

## Out of scope

- Grafana or any dashboard tool (ADR-009 amendment).
- A freshness SLO for the public site, and alerting (first persistent
  environment).
- A custom domain, JavaScript, and search.
- Any public API, Ingress, or AWS resource.
- New API endpoints or pages that combine data (ADR-001).

## Exit criteria

- Every scope item is merged with CI green.
- A scheduled publication appears at the public address with no manual step.
- A promotion to `local` changes the public site at the next publication.
- A failed publication leaves the previous publication online.
- The Sprint 5 review is written, with the run evidence.

## Risks

- **Jolpica rate limits:** 500 requests an hour unauthenticated, and Jolpica
  says the limits will decrease; the backfill in item 4 may not finish within
  the 6-hour job limit if they do. The full-backfill target is 3 hours
  (README).
- **Dump and image drift:** a new migration can make an old dump unusable; the
  publication restores the dump and then runs the image's migrations, and
  fails if they do not apply.
- **Reading `local`'s image from platform-gitops:** couples the workflow to
  the layout of that repository; a change there must update the workflow.
- **Pages limits:** about 1,100 race pages plus standings, well under GitHub
  Pages' size limit, but each publication rewrites every page.
