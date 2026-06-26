# apply.md

GitOps apply runbook for **`github.com/rshdhere/ops`**.

Everything in this repo is delivered through **Argo CD**. Each app has
`syncPolicy.automated` with `prune: true` and `selfHeal: true`, so once you
`kubectl apply` an `Application` manifest into the `argocd` namespace, Argo CD
pulls from `HEAD` and reconciles automatically. You normally only `kubectl
apply` the `Application` objects (and a few cluster-scoped bootstrap resources);
Argo CD does the rest.

> Apply order matters: bring up cluster add-ons, then the shared platform
> (Vault / External Secrets / ClusterIssuer), then **staging**, then
> **production**.

---

## 0. Prerequisites (once per cluster)

Set your context first and confirm it:

```bash
kubectl config current-context
```

These add-ons must exist before the apps will become healthy:

| Component | Needed by | Notes |
| --- | --- | --- |
| Argo CD | everything | runs in `argocd` namespace |
| ingress-nginx | devin + metaverse ingress | `ingressClassName: nginx` |
| cert-manager | TLS certs / ClusterIssuer | `letsencrypt-prod` issuer |
| sealed-secrets controller | `production/metaverse` | decrypts the committed `SealedSecret` |
| Vault | devin `ExternalSecret` | reachable at `http://vault.vault.svc:8200` |

Install (skip any you already run):

```bash
# Argo CD
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# cert-manager (CRDs + controller)
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace --set crds.enabled=true

# sealed-secrets controller (Bitnami)
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

helm repo update
```

> **Vault**: deploy Vault into the `vault` namespace and enable the Kubernetes
> auth backend with role `external-secrets` (see the comments in
> `production/infrastructure/external-secrets/manifests/cluster-secret-store.yaml`).
> The KV v2 mount `secret/` must contain `staging/server` and `prod/server`
> before devin's `ExternalSecret` can sync.

---

## 1. Shared platform bootstrap (applies to both environments)

These are cluster-scoped and shared by staging **and** production. Apply them once.

```bash
# cert-manager ClusterIssuer (letsencrypt-prod)
kubectl apply -f production/certificate/issuer.yaml

# External Secrets Operator + Vault ClusterSecretStore (Helm + manifests via Argo CD)
kubectl apply -f production/infrastructure/external-secrets/application.yaml
```

Verify the platform is healthy before moving on:

```bash
kubectl get clusterissuer letsencrypt-prod
kubectl -n argocd get applications external-secrets
kubectl get clustersecretstore vault-backend
```

`vault-backend` should report `Valid` once Vault auth is wired up.

---

## 2. Staging

Staging consists of the **devin-staging** Argo CD app only. The
`staging/metaverse`, `staging/certificate`, and `staging/ingress` files are
empty placeholders and are intentionally **not** applied. Staging reuses the
shared `vault-backend` `ClusterSecretStore` from step 1 and pulls Vault key
`staging/server`.

```bash
kubectl apply -f staging/devin/application.yaml
```

What this deploys (path `staging/devin/overlays/external`, kustomize):

- devin namespaces, CRDs, orchestrator, server/web, ingress
- `ExternalSecret devin-server` patched to Vault key `staging/server`
- staging Firecracker host(s) from `firecracker-hosts.yaml`

Verify:

```bash
kubectl -n argocd get application devin-staging
# or with the CLI:
argocd app get devin-staging
argocd app sync devin-staging          # only if you disabled auto-sync

kubectl -n devin-app get externalsecret devin-server
kubectl -n devin-app get pods
kubectl -n devin-system get pods
```

`ExternalSecret devin-server` should reach `SecretSynced` once Vault holds
`secret/staging/server`.

---

## 3. Production

Apply in this order. Argo CD auto-syncs each one after the manifest lands.

```bash
# 1) devin (orchestrator + server/web + firecracker hosts)
kubectl apply -f production/devin/application.yaml

# 2) metaverse (frontend/backend/websocket + SealedSecret)
kubectl apply -f production/metaverse/application.yaml

# 3) metaverse ingress (not managed by an Argo CD app)
kubectl apply -f production/ingress/ingress.yaml
```

Notes:

- **devin** uses Vault key `prod/server` via the base `ExternalSecret`
  (`production/devin/base/vault/external-secret.yaml`). Ensure `secret/prod/server`
  exists in Vault.
- **metaverse** decrypts `production/metaverse/sealed-secret.yaml` into the
  `metaverse-backend-prod` Secret — this requires the sealed-secrets controller
  (step 0). The metaverse app uses `directory.recurse: false`, so it applies the
  manifests directly in `production/metaverse/` (deployment, certificate,
  sealed-secret).
- The metaverse **ingress** (`production/ingress/ingress.yaml`) and the
  metaverse **certificate** are not wrapped in an Argo CD app — apply the ingress
  manually as shown above. Hostnames: `k8s-metaverse.raashed.xyz`,
  `k8s-game-server.raashed.xyz`, `k8s-game.raashed.xyz`.

Verify:

```bash
kubectl -n argocd get applications
argocd app get devin
argocd app get metaverse

kubectl -n devin-app get externalsecret devin-server
kubectl -n metaverse get sealedsecret,secret,pods
kubectl -n metaverse get certificate
kubectl -n metaverse get ingress metaverse-ingress
```

All Argo CD apps should read `Synced / Healthy`, and the cert-manager
`Certificate` objects should become `Ready=True` once DNS resolves to the
ingress controller.

---

## Quick reference

```bash
# Platform (once)
kubectl apply -f production/certificate/issuer.yaml
kubectl apply -f production/infrastructure/external-secrets/application.yaml

# Staging
kubectl apply -f staging/devin/application.yaml

# Production
kubectl apply -f production/devin/application.yaml
kubectl apply -f production/metaverse/application.yaml
kubectl apply -f production/ingress/ingress.yaml
```

> Heads up: this working copy's git remote is `rshdhere/staging-ops`, but every
> `Application` manifest points at `github.com/rshdhere/ops`. Push to / keep
> `rshdhere/ops` as the source of truth that Argo CD tracks, or update the
> `repoURL` fields if you switch repos.
