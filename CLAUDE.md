# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Context

This is a homelab Kubernetes project running on a **Mac mini M1** (headless, accessed via SSH over Tailscale). Kubernetes runs inside a Colima VM. All YAML manifests and Helm values are managed here.

Access model: MacBook Pro → SSH via Tailscale → Mac mini M1 → Colima VM → Kubernetes.

## Cluster Management Commands

```bash
# Start cluster
colima start --kubernetes --cpu 4 --memory 8 --disk 200

# Cluster health
colima status
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass

# Events (sorted by time)
kubectl get events -A --sort-by=.metadata.creationTimestamp

# Apply a manifest
kubectl apply -f <file>.yaml

# Port-forward a service (expose to MacBook via Tailscale IP)
kubectl port-forward -n <namespace> svc/<service> --address 0.0.0.0 <local>:<remote>
```

## Namespaces

| Namespace | Purpose |
|---|---|
| `databases` | CloudNativePG operator + PostgreSQL clusters |
| `observability` | Prometheus, Grafana, Loki |
| `automation` | n8n |
| `homelab` | Homepage dashboard |
| `cnpg-system` | CloudNativePG operator |

## Stack & Helm Repos

| Component | Helm repo / chart |
|---|---|
| CloudNativePG | `cnpg` → `https://cloudnative-pg.github.io/charts` |
| Prometheus+Grafana | `prometheus-community` → `kube-prometheus-stack` |
| Loki | `grafana` → `https://grafana.github.io/helm-charts` |

## Architecture Constraints

- **Single-node cluster** — no real HA. PVCs use `local-path` StorageClass.
- All persistent services (PostgreSQL, n8n, Loki, Grafana) must have PVCs.
- Colima has 8 GB RAM allocated; apply resource limits to every workload.
- Service exposure order: port-forward first → Ingress with `*.homelab.local` hostnames → Tailscale DNS.

## Repository Structure (planned)

```
homelab/
├── databases/       # CNPG Cluster manifests + Secrets
├── observability/   # prometheus-values.yaml, loki-values.yaml
├── automation/      # n8n-values.yaml
├── homelab/         # Homepage values + config
├── ingress/         # Ingress resources
└── scripts/         # status.sh, port-forwards.sh
```

## Credentials Convention

Secrets use `kubernetes.io/basic-auth` type. Placeholder passwords in manifests (`*-change-me`) must be replaced before `kubectl apply`. Future direction: SOPS or Sealed Secrets.
