# devops-todolist-gitops

Desired state for the [devops-todolist](https://github.com/HoHuuAn/devops-todolist)
stack running on k3s. ArgoCD watches this repo and reconciles the cluster to
whatever is on `main` — nothing is ever applied to the cluster by hand or from CI.

## Layout

```
charts/todolist/           umbrella chart
  Chart.yaml               declares the three subcharts
  values.yaml              defaults shared by all environments
  templates/ingress.yaml   single ingress fronting both services
  charts/backend/          Express + SQLite API (Deployment + PVC + Service)
  charts/frontend/         React build served by nginx (Deployment + Service)
  charts/redis/            Redis cache (StatefulSet + Service)
envs/prod/values.yaml      production overlay — CI writes image tags here
apps/todolist-prod.yaml    ArgoCD Application pointing at the chart + overlay
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

## Constraints worth knowing

**The backend is pinned to one replica.** It stores data in SQLite via
`better-sqlite3` on a `ReadWriteOnce` volume, so only one pod may hold the
database file open. The deployment uses the `Recreate` strategy for the same
reason: a rolling update would briefly run two writers. Scaling the backend
horizontally requires moving off SQLite first.

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
