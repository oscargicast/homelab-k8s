# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Context

This is a homelab Kubernetes project running on a **Mac mini M1** (headless, accessed via SSH over Tailscale). Kubernetes runs inside a Colima VM. All YAML manifests and Helm values are managed here via **GitOps with Argo CD**.

Access model: MacBook Pro → SSH via Tailscale → Mac mini M1 → Colima VM → Kubernetes.

**Work split:**
- **MacBook Pro (this machine):** edit YAML files, git commit, git push
- **Mac mini (`ssh macmini`):** kubectl, helm, colima, argocd CLI

## GitOps Workflow

This repo uses the **App of Apps** pattern with Argo CD. Do NOT run `kubectl apply` manually for workloads — push to Git and Argo CD syncs automatically.

```
Git push → Argo CD detects change → applies to cluster
```

The only manual `kubectl apply` is the bootstrap:
```bash
# One-time bootstrap (run on Mac mini):
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Secrets — Sealed Secrets

**Never commit plaintext secrets.** `.gitignore` blocks `**/secret.yaml`.

Workflow to add/update a secret:
```bash
# 1. Create plaintext secret in /tmp (never in repo)
cat > /tmp/my-secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: target-namespace
type: kubernetes.io/basic-auth
stringData:
  username: user
  password: password-here
EOF

# 2. Seal with cluster's public cert (run on Mac mini to get cert first)
kubeseal \
  --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem \
  --format yaml \
  < /tmp/my-secret.yaml \
  > path/to/sealed-secret.yaml

# 3. Commit the sealed-secret.yaml (safe to commit)
git add path/to/sealed-secret.yaml && git commit -m "feat: add sealed secret"

# 4. Delete the plaintext
rm /tmp/my-secret.yaml
```

Get the cluster cert (run on Mac mini):
```bash
kubeseal --fetch-cert \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  > ~/.config/homelab-k8s-sealed-secrets-pub.pem
```

## Repository Structure

```
homelab-k8s/
├── .gitignore                          # blocks **/secret.yaml
├── bootstrap/argocd/root-app.yaml      # one-time manual bootstrap
├── clusters/mac-mini/                  # cluster-level parent Applications
│   ├── apps.yaml                       # databases + automation + homelab
│   ├── infrastructure.yaml             # namespaces + operators
│   └── observability.yaml             # prometheus + loki
├── infrastructure/
│   ├── namespaces/                     # Namespace CRs
│   ├── sealed-secrets/application.yaml # Sealed Secrets operator
│   └── cloudnative-pg/application.yaml # CNPG operator
├── databases/
│   ├── postgres-lab/                   # CNPG Cluster + SealedSecret
│   └── n8n-postgres/                  # CNPG Cluster + SealedSecret
├── observability/
│   ├── prometheus/                     # kube-prometheus-stack + values
│   └── loki/                          # Loki SingleBinary + values
├── automation/n8n/                     # n8n + values
└── homelab/homepage/                   # Homepage dashboard + values
```

## Namespaces

| Namespace | Purpose |
|---|---|
| `argocd` | Argo CD (GitOps controller) |
| `kube-system` | Sealed Secrets operator |
| `cnpg-system` | CloudNativePG operator |
| `databases` | PostgreSQL clusters (postgres-lab, n8n-postgres) |
| `observability` | Prometheus, Grafana, Loki |
| `automation` | n8n |
| `homelab` | Homepage dashboard |

## Stack & Helm Charts

| Component | Helm repo | Chart | Version |
|---|---|---|---|
| Argo CD | manual install | install.yaml | stable |
| Sealed Secrets | `https://bitnami-labs.github.io/sealed-secrets` | `sealed-secrets` | `2.*` |
| CloudNativePG | `https://cloudnative-pg.github.io/charts` | `cloudnative-pg` | `0.28.*` |
| Prometheus+Grafana | `https://prometheus-community.github.io/helm-charts` | `kube-prometheus-stack` | `65.*` |
| Loki | `https://grafana.github.io/helm-charts` | `loki` | `6.*` |
| n8n | `oci://8gears.container-registry.com/library/n8n` | `n8n` | `2.0.1` |
| Homepage | `https://jameswynn.github.io/helm-charts` | `homepage` | `2.*` |

## Architecture Constraints

- **Single-node cluster** — no real HA. PVCs use `local-path` StorageClass.
- All persistent services must have PVCs with `storageClass: local-path`.
- Colima has 8 GB RAM allocated; apply resource limits to every workload.
- `ServerSideApply=true` required for kube-prometheus-stack (large CRDs).
- Multi-source Applications (`sources:` plural) require Argo CD ≥ 2.6.
- ArgoCD initial install must use `kubectl apply --server-side` to avoid annotation-too-long errors on CRDs.
- OCI Helm charts in Argo CD require an exact version (e.g. `0.25.0`) — semver wildcards (`0.25.*`) are not supported.
- Directory sources use `include: "**/application.yaml"` (with `**`) to recurse into subdirectories.

## Chart-specific gotchas

| Chart | Gotcha |
|---|---|
| **n8n 2.0.1** | All config moves under `main:` key — `config`, `extraEnv`, `persistence`, `resources` all nested under `main`. |
| **n8n 2.0.1** | Use `main.extraEnv` with `valueFrom.secretKeyRef` to inject secrets (no more `extraEnvSecrets`). |
| **n8n 2.0.1** | `persistence.type: dynamic` is required to bind a PVC — must be under `main.persistence`. |
| **n8n 2.0.1** | Ingress paths require explicit `pathType: Prefix` (previously hardcoded). |
| **n8n 2.0.1** | n8n chart uses OCI (`oci://8gears.container-registry.com/library/n8n`); the old HTTP chartrepo returns HTML. |
| **homepage 1.x** | `serviceAccount.create: true` must be set explicitly — omitting it leaves the SA missing and pods fail to schedule. |
| **homepage 1.x** | `persistence` block at root level is incompatible with bjw-s/common v1. Remove it; config is managed via the `config:` section. |
| **loki 6.x** | `loki.schemaConfig` is required — the chart refuses to render without it. Use schema `v13` / store `tsdb`. |
| **loki 6.x** | `chunksCache.enabled: false` and `resultsCache.enabled: false` are needed on single-node (memcached StatefulSets exhaust RAM). |

## Cluster Management Commands

Run these on **Mac mini** (`ssh macmini`):

```bash
# Start cluster
colima start --kubernetes --cpu 4 --memory 8 --disk 200

# Cluster health
colima status
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass

# Argo CD — check all apps
kubectl get applications -n argocd -o wide

# Force sync an app
argocd app sync <app-name>

# Events (sorted by time)
kubectl get events -A --sort-by=.metadata.creationTimestamp

# Port-forward a service (expose to MacBook via Tailscale IP)
kubectl port-forward -n <namespace> svc/<service> --address 0.0.0.0 <local>:<remote>
```

## Argo CD Access

```bash
# On Mac mini:
kubectl port-forward -n argocd svc/argocd-server --address 0.0.0.0 8080:443
# Open: https://<tailscale-ip>:8080
# User: admin
# Pass: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
