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
│   ├── infrastructure.yaml             # namespaces + operators + cloudflared
│   └── observability.yaml             # prometheus + loki
├── infrastructure/
│   ├── namespaces/                     # Namespace CRs
│   ├── sealed-secrets/application.yaml # Sealed Secrets operator
│   ├── cloudnative-pg/application.yaml # CNPG operator
│   ├── traefik/application.yaml        # Traefik ingress controller
│   ├── cloudflared/                    # CF Tunnel: deployment + sealed-secret + Application
│   └── argocd-config/params-cm.yaml    # argocd-cmd-params-cm overrides
├── databases/
│   ├── postgres-lab/                   # CNPG Cluster + SealedSecret
│   └── n8n-postgres/                  # CNPG Cluster + SealedSecret
├── observability/
│   ├── prometheus/                     # kube-prometheus-stack + values + ingresses
│   ├── loki/                           # Loki SingleBinary + values
│   └── grafana-dashboards/             # ConfigMaps con dashboards (CNPG, etc.)
├── automation/n8n/                     # n8n + values + ingress-public
├── homelab/homepage/                   # Homepage dashboard + values + ingress-public
└── docs/runbooks/                      # operational runbooks
```

## Namespaces

| Namespace | Purpose |
|---|---|
| `argocd` | Argo CD (GitOps controller) |
| `kube-system` | Sealed Secrets operator |
| `cnpg-system` | CloudNativePG operator |
| `databases` | PostgreSQL clusters (postgres-lab, n8n-postgres) |
| `observability` | Prometheus, Grafana, Loki |
| `traefik` | Traefik ingress controller |
| `cloudflared` | Cloudflare Tunnel connector (cloudflared Deployment) |
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
| Traefik | `https://helm.traefik.io/traefik` | `traefik` | `40.0.0` |
| cloudflared | _(no Helm chart, manifest-based)_ | `cloudflare/cloudflared` image | `2024.10.0` |
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

## Cloudflare Tunnel routing

The cluster exposes services in two ways: **public via Cloudflare Tunnel** (n8n, Grafana, Homepage, Argo CD) and **internal via Tailscale + Traefik** (Prometheus).

### Architecture

- `cloudflared` runs as a 2-replica `Deployment` in the `cloudflared` namespace, authenticating with a token sealed via Bitnami SealedSecrets (`infrastructure/cloudflared/sealed-secret.yaml`, key `token`).
- The connection is **outbound-only** from the cluster to Cloudflare edge over QUIC — no inbound ports open on the Mac mini.
- All public hostnames in CF point to **the same backend**: `traefik.traefik.svc.cluster.local:80`. Cloudflare Tunnel forwards every public request to Traefik, and **Traefik routes by `Host` header** using regular `Ingress` resources (`*-public` Ingresses living next to each app's `values.yaml`).

### DNS records (zone `oscargicast.com`)

| Hostname | Type | Target | Proxy | Source |
|---|---|---|---|---|
| `n8n.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `grafana.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `homepage.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `argocd.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `prometheus.oscargicast.com` | CNAME | `oscar-mini-m1.tail90f0a7.ts.net` | ⚫ DNS only | Manual (must NOT be proxied — Tailscale IP is private) |

### Config-as-code trade-off

The Public Hostname rules (all → Traefik) live in the **CF dashboard**, not in Git. The actual routing logic (host matching, middlewares, headers) lives in **Traefik `Ingress` resources in this repo**. The CF rules are trivial and stable, so adding a new public service = adding one `Ingress` here + one CNAME/Public Hostname in CF.

### Cloudflare Access

`grafana`, `homepage` and `argocd.oscargicast.com` are protected by Self-hosted Access applications (policy: email = `oscar.gi.cast@gmail.com`). `n8n.oscargicast.com` is **not** behind Access — it has its own login. Prometheus has no Access (only reachable on tailnet).

### Argo CD specifics

Argo CD runs in `--insecure` mode (HTTP-only) so the tunnel can proxy it cleanly without dealing with self-signed certs. The override is in `infrastructure/argocd-config/params-cm.yaml`:

```yaml
data:
  server.insecure: "true"
```

This makes `argocd-server` serve HTTP on its container port 8080. The Service exposes both port 80 and 443 forwarding to that same port, so the Ingress targets `argocd-server:80`. CF terminates TLS at the edge, the intra-cluster hop is HTTP. No port-forward needed anymore — drop `kubectl port-forward -n argocd svc/argocd-server :8443:443` once `argocd.oscargicast.com` is verified.

### Pending

- **Rotar el token de cloudflared**: el token original se compartió en chat durante el planning. Rotarlo desde `Networks → Tunnels → homelab-k8s → ⋯ → Refresh token`, repetir kubeseal (Fase 2 del runbook), commitear y pushear el nuevo `sealed-secret.yaml`.

## Chart-specific gotchas

| Chart | Gotcha |
|---|---|
| **Traefik 40.x (v3)** | `ingressClass.fallbackApiVersion: v1` was removed in v3 — delete it or the chart fails to render. |
| **Traefik 40.x (v3)** | `ports.web.redirectTo` was removed in v3 — use `redirections` block instead, or remove entirely. |
| **n8n 2.0.1** | All config moves under `main:` key — `config`, `extraEnv`, `persistence`, `resources` all nested under `main`. |
| **n8n 2.0.1** | Use `main.extraEnv` with `valueFrom.secretKeyRef` to inject secrets (no more `extraEnvSecrets`). |
| **n8n 2.0.1** | `persistence.type: dynamic` is required to bind a PVC — must be under `main.persistence`. |
| **n8n 2.0.1** | Ingress paths require explicit `pathType: Prefix` (previously hardcoded). |
| **n8n 2.0.1** | n8n chart uses OCI (`oci://8gears.container-registry.com/library/n8n`); the old HTTP chartrepo returns HTML. |
| **n8n 2.0.1** | The `Service` created by the chart exposes port **80** (mapping internally to the n8n container's `5678`). When defining a separate `Ingress`, target `port: 80`, not `5678`, otherwise Traefik returns 404 with no endpoints. Verify with `kubectl get svc -n automation`. |
| **homepage 2.x** | `enableRbac: true` must be set explicitly for the kubernetes widget to work. |
| **homepage 2.x** | `HOMEPAGE_ALLOWED_HOSTS` env var is required when accessed via a non-standard port. Newer homepage app images reject anything not listed; use `"*"` for single-tenant homelab behind Tailscale (a literal hostname:port can break after image bumps). |
| **homepage 2.x** | `config.kubernetes.mode: cluster` is required even with `enableRbac: true` — the chart's default `kubernetes.yaml` ships with `mode: disable`, so without this override the widget API returns 500 `{error: "No kubernetes configuration"}` and the UI shows "API Error" in the header. |
| **homepage 2.x** | The Prometheus widget is named `prometheusmetric` (singular) and is **service-only** — valid only nested under `services[].<group>.<service>.widget:` (`type: prometheusmetric`). Putting it in the top-level `config.widgets:` array yields `Missing Widget Type: prometheusmetric` in the header. The top-level array only accepts info widgets: `resources`, `search`, `datetime`, `kubernetes`, `glances`, `greeting`, `logo`, `longhorn`, `openmeteo`, `openweathermap`, `stocks`, `unifi_console`. For a cluster-wide overview, create a services group (e.g. `Cluster` → `Overview`) and attach the widget there. |
| **homepage 2.x** | Widget `format:` is a **mapping**, not a string. Use `format: { type: bytes }` (or block style with `type:` nested). Declaring `format: bytes` as a string makes the renderer fail to look up `format.type` and the metric renders as `common.undefined` (service widgets) or `-` (info widgets) **even when the PromQL query returns valid data**. Valid `type:` values: `text, number, percent, bytes, bits, bbytes, bbits, byterate, bibyterate, bitrate, bibitrate, date, duration, relativeDate`. Optional siblings inside `format:`: `scale`, `prefix`, `suffix`, `options` (passed to `Intl.NumberFormat`). |
| **kube-prometheus-stack 65.x** | Default `podMonitorSelector` filters by `release: <chart>` label; PodMonitors emitted by other operators (CNPG, etc.) are ignored. Set `podMonitorSelectorNilUsesHelmValues: false` + `podMonitorSelector: {}` (and the same for `serviceMonitor*`) under `prometheusSpec` to discover cluster-wide. |
| **CloudNativePG 0.28.x** | The operator chart does NOT ship a Grafana dashboard ConfigMap — the `monitoring.grafanaDashboard.create` value lives in the `cluster` subchart only. To get the dashboard, sideload a manual ConfigMap with the JSON from grafana.com (ID `20417`, dashboard UID `cloudnative-pg`) labeled `grafana_dashboard: "1"` in the `observability` namespace so Grafana's sidecar picks it up. |
| **CloudNativePG 0.28.x** | Set `spec.monitoring.enablePodMonitor: true` on each `Cluster` resource to expose metrics on port 9187 and emit a `PodMonitor` (which Prometheus needs the relaxed selectors above to discover). |
| **loki 6.x** | `loki.schemaConfig` is required — the chart refuses to render without it. Use schema `v13` / store `tsdb`. |
| **loki 6.x** | `chunksCache.enabled: false` and `resultsCache.enabled: false` are needed on single-node (memcached StatefulSets exhaust RAM). |
| **Cloudflare Tunnel** | Adding a Public Hostname _should_ auto-create the corresponding CNAME in CF DNS, but it **fails silently** in some flows (especially for hostnames without a Cloudflare Access app on top, e.g. n8n). After adding each Public Hostname, verify with `dig <host> @1.1.1.1 +short` — it must return CF anycast IPs (`104.21.x.x`, `172.67.x.x`). If `NXDOMAIN`, create the CNAME manually: target `<TUNNEL_UUID>.cfargotunnel.com`, orange-cloud (Proxied). |
| **Tailscale MagicDNS** | The `100.100.100.100` resolver caches `NXDOMAIN` according to the upstream SOA negative TTL (Cloudflare = 1800s = 30 min). `dscacheutil -flushcache` does NOT clear Tailscale's cache. To bypass: `dig <host> @1.1.1.1`. To force refresh: `tailscale down && tailscale up`. |
| **infrastructure parent Application** | When a subdir under `infrastructure/` contains manifests beyond `application.yaml` (e.g. `infrastructure/cloudflared/deployment.yaml`), the parent in `clusters/mac-mini/infrastructure.yaml` must use `directory.include: "{**/application.yaml,argocd-config/*.yaml,namespaces/*.yaml}"`. Without that filter, the parent picks up those manifests with destination `argocd` namespace, racing the child Application. |
| **CF Tunnel 404 diagnosis** | A `404 page not found\n` body with `content-length: 19` is Go's default `http.NotFound` — returned by both **cloudflared** (catch-all `service: http_status:404` rule) and **Traefik** (no router/endpoint match). To distinguish: `kubectl logs -n cloudflared deploy/cloudflared \| grep configuration` — if the host appears in the latest config, the 404 is from Traefik (check `kubectl describe ingress` for empty `Backends`). If the host is absent, it's cloudflared's catch-all (re-add the Public Hostname in CF dashboard). |
| **Argo CD `argocd-cmd-params-cm`** | Changes to this ConfigMap (e.g. flipping `server.insecure: "true"`) **do NOT auto-restart `argocd-server`**. The pod reads params only at startup. Symptom: applied via GitOps but pod still serves with old settings (e.g. HTTPS in non-insecure mode → infinite-redirect loop behind a TLS-terminating tunnel). Fix: `kubectl rollout restart deployment/argocd-server -n argocd`. Verify the new pod logs show `tls: false` to confirm insecure mode is active. |

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

Public via CF Tunnel + Access:

```
https://argocd.oscargicast.com
```

Auth flow: CF Access (Google email) → Argo CD login (`admin` + initial password):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> Si necesitás acceso de emergencia con el tunnel caído, el fallback temporal es `kubectl port-forward -n argocd svc/argocd-server --address 0.0.0.0 8443:80` (notar `80`, no `443`, porque ahora corre en `--insecure` mode) y abrir `http://<tailscale-ip>:8443`. Usar solo en emergencia.
