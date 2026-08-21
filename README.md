# devops-todolist-gitops

Desired state for the [devops-todolist](https://github.com/HoHuuAn/devops-todolist)
stack running on k3s. ArgoCD watches this repo and reconciles the cluster to
whatever is on `main` — nothing is ever applied to the cluster by hand or from CI.

## Layout

```
charts/todolist/                   umbrella chart
  values.yaml                      defaults shared by all environments
  templates/ingress.yaml           two ingresses: API, and the one Rollouts owns
  templates/analysistemplate.yaml  canary success criteria (Prometheus)
  templates/smoke-test-hook.yaml   PostSync verification Job
  charts/backend/                  Express + SQLite API (Deployment + PVC +
                                   Service + ServiceMonitor)
  charts/frontend/                 React build served by nginx
                                   (Rollout + stable/canary Services)
  charts/redis/                    Redis cache (StatefulSet + Service)
envs/prod/values.yaml              prod overlay — CI writes image digests here
envs/staging/values.yaml           staging overlay — likewise
apps/todolist-appset.yaml          ApplicationSet generating one App per env
policies/                          Kyverno image-signature policy
docs/applicationset.md             generators, and the failures hit building this
```

## How a deploy happens

1. A push to `main` in the app repo runs security scans, builds both images and
   pushes them to Docker Hub by digest.
2. The `gitops-write-back` job commits the new tag and digest into every
   overlay under `envs/` in this repo.
3. ArgoCD sees the commit and syncs — staging first, then prod. No SSH, no
   `kubectl apply`.
4. A `PostSync` hook smoke-tests each environment after its sync applies.

## Routing

Two ingresses share the host, split by which controller owns them:

| Ingress | Path | Service | Owner |
| --- | --- | --- | --- |
| `todolist-api` | `/api/*`, `/health` | `todolist-backend` | this chart |
| `todolist` | `/*` | `todolist-frontend` | Argo Rollouts |

The split is required, not cosmetic. Argo Rollouts builds its canary ingress by
copying the stable one, so any path left on `todolist` is duplicated onto a
canary whose backend is the frontend canary Service. With `/api` there, nginx
served 404 for the API at 0% canary weight.

The frontend calls the API with a relative URL, so it needs no build-time API
host — same origin, routed by path.

## Environments

An `ApplicationSet` (`apps/todolist-appset.yaml`) generates one Application per
directory under `envs/`, so adding an environment is one values file rather
than another Application to write and keep in sync. `RollingSync` applies
staging first and only moves to prod once staging is Healthy.

| Env | Namespace | Host | Canary |
| --- | --- | --- | --- |
| prod | `todolist` | todolist.103.75.183.229.nip.io | 10 -> 25 -> 50 -> 100, long pauses |
| staging | `todolist-staging` | todolist-staging.103.75.183.229.nip.io | 50 -> 100, 60s pause |

`analysis.appLabel` must differ per environment. The AnalysisTemplate filters
metrics on that label, so a shared value would let staging traffic decide
prod's canary. It also has to match `backend.metrics.appLabel`, which is what
sets `APP_NAME` on the pod.

CI writes image digests into **every** overlay listed in `GITOPS_VALUES`.
Leaving one out does not fail — that environment just quietly stays on a stale
mutable tag until it breaks.

See `docs/applicationset.md` for generator types and the failures hit while
building this, including the sync deadlock that makes ArgoCD ignore new
commits.

## Post-sync verification

`charts/todolist/templates/smoke-test-hook.yaml` runs as an ArgoCD `PostSync`
hook, so the check is tied to the sync event rather than to a timer in CI. It
talks to ClusterIP Services, which keeps it independent of Traefik, the
self-signed nip.io certificate, and whichever pod the canary split happens to
pick.

It asserts health, that `/api/tasks` returns a JSON array, a write path (POST
then DELETE, cleaning up its own row), that `/metrics` still exports
`http_requests_total` — without which canary analysis has no data — and that
the frontend serves the app shell. A read-only check would pass with the
database mounted read-only or the disk full.

The failed Job is kept for inspection; `BeforeHookCreation` clears the
previous one on the next sync.

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

## Image signature verification

CI signs both images with Cosign keyless, but signing alone protects nothing:
Cosign only *produces* a signature, and kubelet has no concept of one. The
check is a separate admission controller, so until one exists the cluster will
happily run any image whose digest appears in `envs/prod/values.yaml`.

`policies/verify-image-signatures.yaml` is that check — a Kyverno
`ClusterPolicy` requiring a keyless signature issued to this project's CI
workflow:

| Field | Value |
| ----- | ----- |
| subject | `https://github.com/HoHuuAn/devops-todolist/.github/workflows/ci-cd.yml@refs/heads/main` |
| issuer | `https://token.actions.githubusercontent.com` |

Both values were read out of the Fulcio certificate on a published signature,
not assumed. The `@refs/heads/main` suffix is what makes this useful: a
signature produced by the same workflow on a fork or a feature branch carries
a different subject and is rejected, so pushing a malicious image to the
registry is not enough to get it running.

Only `hohuuan2003/devops-todolist-*` is in scope. Third-party images (redis,
nginx) are deliberately excluded — requiring signatures from them would block
the namespace.

**The policy is in `Audit` mode.** Violations appear in PolicyReports and Pods
are still admitted:

```bash
kubectl get policyreports -n todolist          # FAIL column is what matters
kubectl describe clusterpolicy verify-todolist-image-signatures
```

Verified behaviour at install time: the signed backend digest was admitted,
and an unsigned image from before Cosign was added was correctly flagged
`no signatures found`.

Switch to `Enforce` by changing `validationFailureAction` — but only after the
reports stay clean for a few days. Keyless verification calls out to Rekor and
Fulcio, and this host has intermittently returned 403 for ghcr.io, so a
network failure under `Enforce` would prevent every Pod in the namespace from
starting. `failurePolicy: Ignore` limits that blast radius but is not a
substitute for watching the reports first.

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
