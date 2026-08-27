# rekordo-deployment

Kustomize manifests for **Rekordo**. ArgoCD watches this repo; nothing here is ever
`kubectl apply`-ed by hand.

## Layout

```
base/            backend, frontend, postgres — environment-invariant
overlays/
  staging/       namespace music-collector-staging · rekordo-staging.jannekeipert.de
  prod/          namespace music-collector-prod    · rekordo.jannekeipert.de
argocd/          the two Applications (registered from cluster-deployment)
```

## How a deploy happens

| Trigger in a product repo | Image tag | Lands in |
|---|---|---|
| push to `main` | `main-<sha>` | staging |
| tag `vX.Y.Z` | `X.Y.Z` | prod |

CI rewrites the `images:` pin in the matching overlay with `kustomize edit set image` and
pushes here. Treat those `newTag` values as CI-owned — don't hand-edit them except to
bootstrap.

## Secrets

Sealed Secrets only, one set per namespace (they are namespace-scoped, so staging and prod
carry different ciphertext for different values):

- `music-collector-db-secret` → `POSTGRES_PASSWORD`, shared by Postgres and the backend
- `music-collector-jwt-secret` → `MC_JWT_SECRET`; the backend has no fallback and fails to
  start rather than run on a known signing key

Regenerate one with:

```bash
kubectl create secret generic <name> -n <namespace> --dry-run=client \
  --from-literal=<KEY>="$(openssl rand -base64 48 | tr -d '\n')" -o yaml \
| kubeseal --controller-namespace kube-system --controller-name sealed-secrets-controller \
    --format yaml > overlays/<env>/sealed-secrets/<name>.yaml
```

## Names the rename deliberately left alone

The app was renamed from Music Collector to Rekordo. The namespaces, the Postgres
database and user, the photo buckets, the `MC_*` environment prefix and the Kubernetes
resource names below still carry the old name, and that is the decision rather than an
oversight: none of them is visible to a user, and moving them would cost a PVC migration,
a re-seal of every sealed secret (they are scoped to name *and* namespace) and a bucket
copy. The old hosts are still served alongside the new ones until a mobile build that
points at `rekordo.` has shipped -- iOS 1.6.0 has the old host compiled in.

## DNS

Both hosts must be **A-only**. The cluster is IPv4 single-stack, and a stray AAAA record
breaks ACME HTTP-01 validation for the host.

## Validating a change

```bash
kubectl kustomize overlays/staging
kubectl kustomize overlays/prod
```
