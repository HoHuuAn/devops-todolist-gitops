# devops-todolist-gitops

Desired state for the [devops-todolist](https://github.com/HoHuuAn/devops-todolist)
stack running on k3s. ArgoCD watches this repo and reconciles the cluster to
whatever is on `main` — nothing is ever applied to the cluster by hand or from CI.

## Layout

```
charts/todolist/                  umbrella chart
  Chart.yaml                      declares the three subcharts
  values.yaml                     defaults shared by all environments
  templates/ingress.yaml          single ingress fronting both services
  templates/analysistemplate.yaml canary success criteria (Prometheus)
  charts/backend/                 Express + SQLite API (Deployment + PVC +
                                  Service + ServiceMonitor)
  charts/frontend/                React build served by nginx
                                  (Rollout + stable/canary Services)
  charts/redis/                   Redis cache (StatefulSet + Service)
envs/prod/values.yaml             production overlay — CI writes image tags here
apps/todolist-prod.yaml           ArgoCD Application pointing at chart + overlay
```

## How a deploy happens

1. A push to `main` in the app repo runs security scans, builds both images and
   pushes them to Docker Hub by digest.
2. The `gitops-write-back` job commits the new tag and digest into
   `envs/prod/values.yaml` in this repo.
3. ArgoCD sees the commit and syncs the cluster. No SSH, no `kubectl apply`.

## Routing

One ingress serves everything on `todolist.103.75.183.229.nip.io`:

| Path      | Service            |
| --------- | ------------------ |
| `/api/*`  | `todolist-backend` |
| `/health` | `todolist-backend` |
| `/*`      | `todolist-frontend`|

The frontend calls the API with a relative URL, so it needs no build-time API
host — same origin, routed by path.

## Progressive delivery

The frontend is an Argo Rollout, not a Deployment. A new image is shifted in
stages — 10% → 25% → 50% → 100% — with pauses between them, while an
`AnalysisTemplate` queries Prometheus every 30s throughout:

| Metric | Threshold | On breach |
| ------ | --------- | --------- |
| Backend 5xx rate | ≤ 1% of requests | 3 consecutive breaches abort the rollout |
| Backend p99 latency | ≤ 1.5s | 3 consecutive breaches abort the rollout |

An aborted analysis returns 100% of traffic to the stable ReplicaSet within
seconds, with no human involved.

Both success conditions begin with `len(result) == 0 ||`. That guard is
load-bearing: an idle canary returns an empty vector from Prometheus, and
evaluating `result[0]` against it errors. Without the guard those errors count
as breaches and a healthy deploy gets rolled back purely for lack of traffic.

The signal is the *backend's* error rate, because the frontend image is stock
nginx serving static files and exports no metrics of its own. A broken
frontend bundle shows up as failing API calls. The `app` label the query
filters on comes from `APP_NAME` on the backend pod — `backend.metrics.appLabel`
and `analysis.appLabel` must stay in agreement or the query silently matches
nothing.

Watching and controlling a rollout:

```bash
kubectl argo rollouts get rollout todolist-frontend -n todolist --watch
kubectl argo rollouts promote todolist-frontend -n todolist   # skip a pause
kubectl argo rollouts abort   todolist-frontend -n todolist   # back to stable
```

Setting `frontend.rollout.enabled: false` renders a plain Deployment instead,
which is the fallback if Argo Rollouts is ever uninstalled.

## Constraints worth knowing

**The backend is pinned to one replica, and is deliberately not canaried.**
It stores data in SQLite via `better-sqlite3` on a `ReadWriteOnce` volume, so
only one pod may hold the database file open. The deployment uses the
`Recreate` strategy for the same reason: a rolling update would briefly run
two writers. A canary is worse still — it would run stable and canary pods
against the same database file concurrently, risking corruption. Canarying
the backend (and scaling it at all) requires moving to Postgres first.

The backend PVC is annotated `helm.sh/resource-policy: keep` and
`Prune=false`, so an ArgoCD prune or a `helm uninstall` will not delete the
database.

## Rolling back

Set `backend.image.digest` (or `frontend.image.digest`) in
`envs/prod/values.yaml` to a previously deployed digest and push. ArgoCD syncs
the older image within a minute. Rolling back through Git keeps the repo as the
single source of truth — prefer it over `argocd app rollback`.

## Working on the chart locally

```bash
helm lint charts/todolist -f envs/prod/values.yaml
helm template todolist charts/todolist -f envs/prod/values.yaml --namespace todolist
```

Subcharts are committed as plain directories, not `.tgz` archives, so there is
no `helm dependency build` step.
