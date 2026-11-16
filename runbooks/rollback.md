# Rollback runbook

## Purpose

Restores an environment's previous release when a promotion goes wrong. A
rollback reverts the promotion in Git and lets Argo CD reconcile; the database
schema is never reverted (ADR-007, ADR-008, `GLOSSARY.md`).

## When to roll back

Roll back after a promotion when any of these holds:

- The Formula Insights Application shows a failed sync, typically because the
  migration hook failed.
- The API pods are not Ready, or `/ready` returns 503 while the database is up.
- The API returns errors or wrong data that the previous release did not.

Do not roll back for a failed daily import alone; see
[`daily-import.md`](daily-import.md).

## Procedure

1. **Find the promotion.** It is the most recent merged pull request in
   `platform-gitops` that changed `newTag` in
   `applications/formula-insights/overlays/<environment>/kustomization.yaml`.
2. **Revert it in Git.** On GitHub, use **Revert** on that pull request, or
   `git revert <merge commit>` on a branch. State the reason in the pull
   request. While the service is down, it may be merged without review.
3. **Merge.** `Validate manifests` must pass, as for any change.
4. **Let Argo CD reconcile.** It reads `main` within 3 minutes; to start at
   once, refresh the Application in the Argo CD UI.

Never use Argo CD's own rollback or `kubectl set image`: automated sync undoes
any change that is not in Git. Never downgrade the schema: migrations are
backward compatible (expand, then contract), so the previous image works on
the newer schema.

### If the migration hook failed

A promotion whose migration fails is never applied: the sync stops before the
API changes, and the previous version keeps serving. After the revert, the
Application is **Synced** and **Healthy** at once, with no rollout, but its last
operation still shows **Failed** and the failed migration Job stays in the
namespace. Start one sync to rerun the hook and clear it: **Sync** in the
Argo CD UI, or

```bash
kubectl -n argocd patch application formula-insights --type merge \
  -p '{"operation":{"initiatedBy":{"username":"rollback"},"sync":{"revision":"<reverted commit>"}}}'
```

## Verify recovery

Expected evidence, to paste into the rollback pull request:

| Check | Command | Expected |
| --- | --- | --- |
| Applications | `kubectl -n argocd get applications` | every Application `Synced` and `Healthy` |
| Last operation | `kubectl -n argocd get application formula-insights -o jsonpath='{.status.operationState.phase}'` | `Succeeded` |
| Migration hook | `kubectl -n formula-insights get job formula-insights-migrate` | `1/1` completions |
| Running image | `kubectl -n formula-insights get deploy formula-insights-api -o jsonpath='{.spec.template.spec.containers[0].image}'` | the previous release's tag |
| Readiness | `kubectl -n formula-insights port-forward svc/formula-insights-api 8000:80`, then `curl -fsS localhost:8000/ready` | `{"status":"ready"}` |
| Data | `curl -fsS localhost:8000/seasons/<current year>/races` | the current season's races |

The Argo CD UI is reached the same way: `kubectl -n argocd port-forward
svc/argocd-server 8080:443`, then <https://localhost:8080> as `admin`. The
initial password is in the `argocd-initial-admin-secret` Secret and changes
with each bootstrap.

## Drill

Rehearsed on the `local` environment on 2026-11-16 (platform-gitops#13 and
#14): a promotion that broke the migration on purpose failed after three
retries in about 8 minutes while the same API pod kept serving. After the
revert, the Application was Synced and Healthy 27 seconds after the merge,
and one sync cleared the failed operation in 10 seconds.
