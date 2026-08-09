# hermes-deployment

Kustomize manifests for **Hermes**, the mail triage service. ArgoCD watches `overlays/main` and
reconciles it onto `k3s-01`. Nothing here is ever `kubectl apply`-ed by hand.

## Layout

```
base/
├── postgres/   StatefulSet + headless Service + PVC (in the Velero regime)
├── backend/    Deployment (two containers), Service, ConfigMap, NetworkPolicy
└── frontend/   Deployment + Service (nginx)
overlays/main/  namespace, ingresses, Authentik forward auth, image tag pins
argocd/         the Application (register it in cluster-deployment/apps/)
docs/           secrets-required.md
```

## The pod

`hermes-backend` runs **two containers in one pod**:

| Container | Port | Reachable from |
|---|---|---|
| `backend` | 8080 | The Service, and the Ingress under `/api` |
| `claude-sidecar` | 8081 | **Only `backend`, over pod loopback** |

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

## Access

`hermes.jannekeipert.de` is one host with three Traefik routers, cut so that each gets the answer
it needs to "who may call this" (`overlays/main/ingress.yaml`):

| Router | Serves | Gate |
|---|---|---|
| `hermes` | the app, `/api`, `/v3/api-docs` | Authentik forward auth — `hermes-users` group |
| `hermes-public` | `/api/v1/events/alert`, `/api/v1/auth/google/callback` | none at the edge; each endpoint carries its own check |
| `authentik-outpost` (authentik ns) | `/outpost.goauthentik.io` | none — it *is* the login callback |

The middleware lives in `overlays/main/authentik.yaml`. The outpost's own path is routed from the
authentik namespace instead: routing it from here would need an ExternalName Service, which Traefik
refuses by default — it then drops the router and the path falls through to the SPA's 404 page. The Authentik-side provider, application and group gate are a
blueprint in `cluster-deployment/infrastructure/authentik-hermes-blueprint.yaml`.

The `hermes-app-key/admin-token` still works as `X-Hermes-Token`, but **only against a router that
has no forward-auth middleware**. Forward auth runs at Traefik, before the backend's own filter, so
a token alone no longer gets a script through `hermes` — it gets a 302 to SSO.

Nothing needs this today. When the Janus widget starts polling `/api/v1/digest/today`, give it a
path on `hermes-public`: the backend's `ApiAccessFilter` does *not* exempt that path, so it stays
token-gated exactly as it is now, just without the interactive redirect in front of it.

## Image tags

CI in each product repo rewrites its own entry in `overlays/main/kustomization.yaml` via `yq` and
pushes; ArgoCD auto-syncs. The pinned tags are `main-<sha>`, never a moving `latest`, so every
rollout is reproducible and a rollback is a git revert.

Do not hand-edit the tags except to bootstrap.
