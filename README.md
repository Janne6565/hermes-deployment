# hermes-deployment

Kustomize manifests for **Hermes**, the mail triage service. ArgoCD watches `overlays/main` and
reconciles it onto `k3s-01`. Nothing here is ever `kubectl apply`-ed by hand.

## Layout

```
base/
├── postgres/   StatefulSet + headless Service + PVC (in the Velero regime)
├── backend/    Deployment (two containers), Service, ConfigMap, NetworkPolicy
└── frontend/   Deployment + Service (nginx)
overlays/main/  namespace, ingress, image tag pins
argocd/         the Application (register it in cluster-deployment/apps/)
docs/           secrets-required.md
```

## The pod

`hermes-backend` runs **two containers in one pod**:

| Container | Port | Reachable from |
|---|---|---|
| `triage-api` | 8080 | The Service, and the Ingress under `/api` |
| `claude-sidecar` | 8081 | **Only `triage-api`, over pod loopback** |

The sidecar holds an account-level Claude credential. It binds to `127.0.0.1`, has no `ports:`
entry, no Service, and no Ingress path — three independent reasons nothing off-pod can reach it,
plus a NetworkPolicy restating the intent at the cluster layer.

`replicas: 1` with `strategy: Recreate` is deliberate: the Gmail poll loop and the digest cron are
singletons, and a second replica would double-push.

## Going live

1. Seal the five secrets — see [`docs/secrets-required.md`](docs/secrets-required.md).
2. Copy `argocd/main-app.yaml` into `cluster-deployment/apps/hermes.yaml` and commit. `root` picks
   it up; no `kubectl`.
3. Leave `HERMES_SHADOW_MODE: "true"` for a few days and review the classifications.
4. Flip it to `"false"`, then turn Gmail's own notifications off on the phone.

Step 3 is the point of the whole rollout: shadow mode classifies and stores everything but pushes
nothing, so the labels can be judged before the service is allowed to interrupt you.

## Image tags

CI in each product repo rewrites its own entry in `overlays/main/kustomization.yaml` via `yq` and
pushes; ArgoCD auto-syncs. The pinned tags are `main-<sha>`, never a moving `latest`, so every
rollout is reproducible and a rollback is a git revert.

Do not hand-edit the tags except to bootstrap.
