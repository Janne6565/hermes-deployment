# Secrets

Five secrets in the `hermes` namespace. All are created with `kubeseal` and committed as
`SealedSecret` manifests — plaintext never enters Git.

Two of them (`gmail-oauth`, `claude-oauth-token`) are **account-level credentials**, not
service-scoped API keys. Treat them accordingly.

| Secret | Keys | What it is |
|---|---|---|
| `hermes-db` | `username`, `password` | Postgres credentials. Consumed by both the StatefulSet and the backend. |
| `hermes-app-key` | `encryption-key`, `admin-token` | AES key protecting the stored Google refresh token, and the single-user gate on `/api`. |
| `gmail-oauth` | `client-id`, `client-secret` | Google OAuth **client** — identifies the app, not the account. The account itself is attached in-app via Sign in with Google. |
| `claude-oauth-token` | `token` | From `claude setup-token`. Bound to the Max account, ~1 year lifetime. |
| `ntfy-credentials` | `token` | Bearer token for the publishing user on `ntfy.jannekeipert.de`. |
| `alert-webhook-secret` | `token` | Shared secret Grafana and SigNoz send as `X-Hermes-Token`. |

## Current state

All six are sealed into `base/backend/` and applied. Three hold real generated credentials;
three are placeholders you need to replace:

| Secret | State |
|---|---|
| `hermes-db` | ✅ generated |
| `hermes-app-key` | ✅ generated — the admin token was written to `~/hermes-admin-token.txt` |
| `alert-webhook-secret` | ✅ generated — read it back with the `kubectl get secret` recipe below |
| `gmail-oauth` | ✅ real client sealed (project `hermes-triage-110372`) |
| `ntfy-credentials` | ⚠️ empty — pushes will fail (harmless while `HERMES_SHADOW_MODE=true`) |

The Google Cloud project is **`hermes-triage-110372`** ("Hermes"), Gmail API enabled, consent
screen in Testing with `jabbekeipert@gmail.com` as a test user. The OAuth client is a Web
application whose only redirect URI is
`https://hermes.jannekeipert.de/api/v1/auth/google/callback` — it must match character for
character or the exchange fails.
| `claude-oauth-token` | ✅ real token sealed — `SIDECAR_ENABLED=true` |

To read a generated value back out of the cluster:

```sh
kubectl get secret alert-webhook-secret -n hermes -o jsonpath='{.data.token}' | base64 -d
```

To replace a placeholder, re-seal just that one and commit:

```sh
kubectl create secret generic claude-oauth-token -n hermes --dry-run=client -o yaml \
  --from-literal=token="<from claude setup-token>" \
  | kubeseal --controller-name sealed-secrets-controller --controller-namespace kube-system \
      --format yaml --scope strict > base/backend/sealed-claude-oauth-token.yaml
```

Then flip `SIDECAR_ENABLED` back to `"true"` in `base/backend/configmap.yaml`.

## Creating them from scratch

```sh
# 1. Database
kubectl create secret generic hermes-db -n hermes --dry-run=client -o yaml \
  --from-literal=username=hermes \
  --from-literal=password="$(openssl rand -base64 24)" \
  | kubeseal --format yaml > base/backend/sealed-hermes-db.yaml

# 2. Application keys
#    encryption-key protects the Google refresh token at rest; changing it forces a re-connect.
#    admin-token is the only thing standing between the public ingress and your mail.
kubectl create secret generic hermes-app-key -n hermes --dry-run=client -o yaml \
  --from-literal=encryption-key="$(openssl rand -base64 32)" \
  --from-literal=admin-token="$(openssl rand -hex 32)" \
  | kubeseal --format yaml > base/backend/sealed-hermes-app-key.yaml

# 3. Google OAuth *client* (no refresh token — that is obtained in-app)
#    Register the redirect URI on the client first:
#      https://hermes.jannekeipert.de/api/v1/auth/google/callback
kubectl create secret generic gmail-oauth -n hermes --dry-run=client -o yaml \
  --from-literal=client-id="…apps.googleusercontent.com" \
  --from-literal=client-secret="…" \
  | kubeseal --format yaml > base/backend/sealed-gmail-oauth.yaml

# 4. Claude OAuth token — run `claude setup-token` locally first
kubectl create secret generic claude-oauth-token -n hermes --dry-run=client -o yaml \
  --from-literal=token="$(pbpaste)" \
  | kubeseal --format yaml > base/backend/sealed-claude-oauth-token.yaml

# 5. ntfy publishing token
kubectl create secret generic ntfy-credentials -n hermes --dry-run=client -o yaml \
  --from-literal=token="tk_…" \
  | kubeseal --format yaml > base/backend/sealed-ntfy-credentials.yaml

# 6. Alert webhook shared secret
kubectl create secret generic alert-webhook-secret -n hermes --dry-run=client -o yaml \
  --from-literal=token="$(openssl rand -hex 32)" \
  | kubeseal --format yaml > base/backend/sealed-alert-webhook-secret.yaml
```

The three `ghcr.io/janne6565/hermes-*` packages are public, so no image pull secret is needed.
If that ever changes, add a `ghcr-pull-secret` docker-registry secret and an `imagePullSecrets`
block to both deployments.

Then add the generated files to `base/backend/kustomization.yaml`.

## Rotation

| Credential | Cadence | Notes |
|---|---|---|
| `claude-oauth-token` | ~annually | Expires roughly a year after `claude setup-token`. Set a calendar reminder — expiry surfaces as `authentication_failed` on the health screen and rules-only classification, not as an outage. |
| Google account grant | On revocation | Google invalidates the refresh token if the consent screen leaves testing mode, the password changes, or access is revoked. Recovery is one click: Sign in with Google again. |
| `hermes-app-key` `encryption-key` | Rarely | Rotating it orphans the stored refresh token — plan on re-connecting the account right after. |
| `hermes-app-key` `admin-token` | On suspicion | Everyone signed into the UI has to re-enter it. |
| `alert-webhook-secret` | On suspicion | Rotate the secret and the Grafana/SigNoz contact points together, or alerts stop arriving. |

## Do not

- Log any of these. The sidecar deliberately logs error *types*, never token values or mail
  content.
- Expose the sidecar. `claude-oauth-token` is a full account credential; the sidecar binds to
  loopback and has no Service. Adding one would put an account credential behind a cluster IP.
- Reuse `alert-webhook-secret` anywhere else. It is the only auth on the one externally reachable
  endpoint.
