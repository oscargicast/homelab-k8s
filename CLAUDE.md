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

## Secrets — Infisical

Los secretos viven en **Infisical self-hosted** (`https://infisical.oscargicast.com`). Los valores se editan en la UI; el repo solo contiene `InfisicalSecret` CRs (punteros, sin valores) que el **Infisical Secrets Operator** materializa como `Secret` nativos en ≤60s.

**Never commit plaintext secrets.** `.gitignore` blocks `**/secret.yaml`.

Workflow para agregar/editar un secreto:

1. **Crear/editar el valor en Infisical UI** (`https://infisical.oscargicast.com`, project `homelab-k8s`, env `prod`, ruta lógica `/<area>/<service>` — p. ej. `/automation/n8n`).
2. **Commitear el `InfisicalSecret` CR** apuntando a esa ruta. Ejemplo real (`automation/n8n/infisical-secret.yaml`):

   ```yaml
   apiVersion: secrets.infisical.com/v1alpha1
   kind: InfisicalSecret
   metadata:
     name: n8n-db-credentials
     namespace: automation
   spec:
     resyncInterval: 60
     authentication:
       kubernetesAuth:
         identityId: <machine-identity-uuid>
         serviceAccountRef:
           name: infisical-operator-controller-manager
           namespace: infisical-operator
         secretsScope:
           projectSlug: homelab-k8s
           envSlug: prod
           secretsPath: /automation/n8n
     managedSecretReference:
       secretName: n8n-db-credentials
       secretNamespace: automation
       creationPolicy: Owner
   ```

3. **Incluir el archivo en el `directory.include` del Application** del servicio (ver `automation/n8n/application.yaml`).
4. Push → ArgoCD aplica el CR → operator lee Infisical → escribe `Secret` nativo en el namespace destino. Editar el valor después solo requiere tocar la UI; no hay commit.

> Sealed Secrets operator sigue desplegado en `kube-system` por seguridad histórica, pero **no hay `SealedSecret` CRs en Git**. Migración completada — ver `docs/runbooks/infisical-migration.md` para detalles del rollout y el bootstrap (`infisical-bootstrap` Secret + Machine Identity en `infisical-operator`).

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
│   ├── sealed-secrets/application.yaml # legacy operator (sin CRs vivos en Git)
│   ├── infisical/                      # Infisical backend (self-hosted) + ingress
│   ├── infisical-operator/             # Infisical Secrets Operator + sa-token
│   ├── redis/                          # Redis dedicado para Infisical
│   ├── cloudnative-pg/application.yaml # CNPG operator
│   ├── traefik/application.yaml        # Traefik ingress controller
│   ├── cloudflared/                    # CF Tunnel: deployment + InfisicalSecret + Application
│   ├── tailscale-operator/             # Tailscale K8s Operator (expone Services al tailnet)
│   └── argocd-config/                  # argocd-cm + argocd-rbac-cm + params-cm overrides
├── databases/
│   ├── chapatuplaza/                   # CNPG Cluster + InfisicalSecret + Service tailnet
│   ├── n8n-postgres/                   # CNPG Cluster + InfisicalSecret + Service tailnet
│   ├── evolution-postgres/             # CNPG Cluster + InfisicalSecret + Service tailnet (Evolution API)
│   └── infisical-postgres/             # CNPG Cluster (Infisical backend DB)
├── observability/
│   ├── prometheus/                     # kube-prometheus-stack + values + ingresses
│   ├── loki/                           # Loki SingleBinary + values
│   └── grafana-dashboards/             # ConfigMaps con dashboards (CNPG, etc.)
├── automation/
│   ├── n8n/                            # n8n + values + ingress-public + InfisicalSecret
│   └── evolution-api/                  # manifest-based + hostNetwork + ServiceMonitor + InfisicalSecret
├── homelab/
│   ├── homepage/                       # values + ingress-public + InfisicalSecret
│   └── speedtest-tracker/              # manifest-based + ingress-internal + ServiceMonitor + InfisicalSecret
└── docs/runbooks/                      # operational runbooks
```

## Namespaces

| Namespace | Purpose |
|---|---|
| `argocd` | Argo CD (GitOps controller) |
| `kube-system` | Sealed Secrets operator (legacy, sin CRs vivos) |
| `cnpg-system` | CloudNativePG operator |
| `databases` | PostgreSQL clusters (chapatuplaza, n8n-postgres, evolution-postgres, infisical-postgres) |
| `observability` | Prometheus, Grafana, Loki |
| `traefik` | Traefik ingress controller |
| `cloudflared` | Cloudflare Tunnel connector (cloudflared Deployment) |
| `tailscale` | Tailscale K8s Operator + sidecars `ts-*` (uno por Service expuesto al tailnet) |
| `infisical` | Infisical backend (self-hosted) + dedicated Redis (in migration) |
| `infisical-operator` | Infisical Secrets Operator (materializes Secrets from InfisicalSecret CRs) |
| `automation` | n8n + evolution-api |
| `homelab` | Homepage dashboard + speedtest-tracker |

## Stack & Helm Charts

| Component | Helm repo | Chart | Version |
|---|---|---|---|
| Argo CD | manual install | install.yaml | stable |
| Sealed Secrets | `https://bitnami-labs.github.io/sealed-secrets` | `sealed-secrets` | `2.*` _(legacy, sin CRs vivos)_ |
| CloudNativePG | `https://cloudnative-pg.github.io/charts` | `cloudnative-pg` | `0.28.*` |
| Prometheus+Grafana | `https://prometheus-community.github.io/helm-charts` | `kube-prometheus-stack` | `85.0.1` (Prometheus `v3.11.3`, Grafana `13.0.1`, operator `v0.90.1`) |
| Loki | `https://grafana-community.github.io/helm-charts` | `loki` | `13.6.2` (Loki `3.7.1`) |
| Traefik | `https://helm.traefik.io/traefik` | `traefik` | `40.0.0` |
| cloudflared | _(no Helm chart, manifest-based)_ | `cloudflare/cloudflared` image | `2024.10.0` |
| n8n | `oci://8gears.container-registry.com/library/n8n` | `n8n` | `2.0.1` |
| Evolution API | _(no Helm chart, manifest-based)_ | `evoapicloud/evolution-api` image | `v2.3.7` |
| Homepage | `https://jameswynn.github.io/helm-charts` | `homepage` | `2.*` |
| Speedtest Tracker | _(no Helm chart, manifest-based)_ | `lscr.io/linuxserver/speedtest-tracker` | `1.14.1` |
| Infisical (backend+UI) | `https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/` | `infisical-standalone` | `1.8.0` (image `v0.159.28`) |
| Infisical Secrets Operator | `https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/` | `secrets-operator` | `0.10.33` |
| Redis (for Infisical) | _(no Helm chart, manifest-based)_ | `redis` (official) | `8.6.3-alpine` |
| Tailscale Operator | `https://pkgs.tailscale.com/helmcharts` | `tailscale-operator` | `1.96.5` |

## Architecture Constraints

- **Single-node cluster** — no real HA. PVCs use `local-path` StorageClass.
- All persistent services must have PVCs with `storageClass: local-path`.
- Colima has 10 GB RAM allocated; apply resource limits to every workload.
- `ServerSideApply=true` required for kube-prometheus-stack (large CRDs).
- Multi-source Applications (`sources:` plural) require Argo CD ≥ 2.6.
- ArgoCD initial install must use `kubectl apply --server-side` to avoid annotation-too-long errors on CRDs.
- OCI Helm charts in Argo CD require an exact version (e.g. `0.25.0`) — semver wildcards (`0.25.*`) are not supported.
- Directory sources use `include: "**/application.yaml"` (with `**`) to recurse into subdirectories.

## Cloudflare Tunnel routing

The cluster exposes services in two ways: **public via Cloudflare Tunnel** (n8n, Evolution API, Grafana, Homepage, Argo CD) and **internal via Tailscale + Traefik** (Prometheus, Speedtest Tracker).

### Architecture

- `cloudflared` runs as a 2-replica `Deployment` in the `cloudflared` namespace, authenticating with a token managed via Infisical (`infrastructure/cloudflared/infisical-secret.yaml` → materializes Secret `cloudflared-token`, key `token`).
- The connection is **outbound-only** from the cluster to Cloudflare edge over QUIC — no inbound ports open on the Mac mini.
- All public hostnames in CF point to **the same backend**: `traefik.traefik.svc.cluster.local:80`. Cloudflare Tunnel forwards every public request to Traefik, and **Traefik routes by `Host` header** using regular `Ingress` resources (`*-public` Ingresses living next to each app's `values.yaml`).

### DNS records (zone `oscargicast.com`)

| Hostname | Type | Target | Proxy | Source |
|---|---|---|---|---|
| `n8n.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `evolution.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `grafana.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `homepage.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `argocd.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Tunnel Public Hostname |
| `prometheus.oscargicast.com` | CNAME | `oscar-mini-m1.tail90f0a7.ts.net` | ⚫ DNS only | Manual (must NOT be proxied — Tailscale IP is private) |
| `speedtest.oscargicast.com` | CNAME | `oscar-mini-m1.tail90f0a7.ts.net` | ⚫ DNS only | Manual (must NOT be proxied — Tailscale-only speedtest-tracker UI/API) |

### Config-as-code trade-off

The Public Hostname rules (all → Traefik) live in the **CF dashboard**, not in Git. The actual routing logic (host matching, middlewares, headers) lives in **Traefik `Ingress` resources in this repo**. The CF rules are trivial and stable, so adding a new public service = adding one `Ingress` here + one CNAME/Public Hostname in CF.

### Cloudflare Access

`grafana`, `homepage`, `argocd` and `evolution.oscargicast.com` are protected by Self-hosted Access applications (policy: email = `oscar.gi.cast@gmail.com`). `n8n.oscargicast.com` is **not** behind Access — it has its own login. Prometheus has no Access (only reachable on tailnet).

### Argo CD specifics

Argo CD runs in `--insecure` mode (HTTP-only) so the tunnel can proxy it cleanly without dealing with self-signed certs. The override is in `infrastructure/argocd-config/params-cm.yaml`:

```yaml
data:
  server.insecure: "true"
```

This makes `argocd-server` serve HTTP on its container port 8080. The Service exposes both port 80 and 443 forwarding to that same port, so the Ingress targets `argocd-server:80`. CF terminates TLS at the edge, the intra-cluster hop is HTTP. No port-forward needed anymore — drop `kubectl port-forward -n argocd svc/argocd-server :8443:443` once `argocd.oscargicast.com` is verified.

### Evolution API specifics

Evolution API (`evoapicloud/evolution-api:v2.3.7`) bundlea Baileys 7.0.0-rc.9, una RC con bugs conocidos que requieren tres decisiones contraintuitivas para funcionar:

1. **`hostNetwork: true` + `dnsPolicy: ClusterFirstWithHostNet`** — sin esto, la conexión post-QR se queda en estado "connecting" indefinidamente (la red virtual de Colima no se lleva bien con los WebSockets de larga duración que usa Baileys con WhatsApp). El pod corre en puerto `8085` directamente sobre el host de Colima (no 8080, ese lo usa Traefik vía port-forward).

2. **`CACHE_REDIS_ENABLED: "false"` + `CACHE_LOCAL_ENABLED: "true"`** — Evolution API intenta conectarse a Redis por default. Sin Redis desplegado (no tenemos ni queremos), la conexión falla silenciosamente y rompe Baileys: la instancia llega a status "open" pero **no procesa mensajes/contactos/chats** (todas las métricas quedan en 0). Cambiar a cache local resuelve esto inmediatamente.

3. **`LOG_LEVEL` con scopes explícitos** (`"ERROR,WARN,INFO,LOG,VERBOSE,WEBHOOKS,WEBSOCKET"`) — con `LOG_LEVEL: "WARN"` los errores de Redis se ocultan, lo que enmascara el problema #2 durante horas. Los scopes `WEBHOOKS` y `WEBSOCKET` son clave para diagnosticar problemas de delivery.

Para acceso operacional: el manager UI está en `https://evolution.oscargicast.com/manager`. Login con SERVER_URL `https://evolution.oscargicast.com` + el API key (extraerlo con `kubectl -n automation get secret evolution-api-secrets -o jsonpath='{.data.api-key}' | base64 -d`). La instancia de WhatsApp se crea desde la UI con un nombre (ej. `personal`) + número en formato JID (sin `+`, sin espacios: `51907898162` para Perú). El QR aparece en el dashboard de la instancia.

Para n8n: configurar webhook URL como **DNS interno** (`http://n8n.automation.svc.cluster.local/webhook/<UUID>`), no la URL pública — evita hairpinning innecesario por CF Tunnel. El workflow en n8n debe estar **activo** (no en test mode), porque las URLs `/webhook-test/...` son efímeras y solo responden cuando el editor está abierto.

### Pending

- **Rotar el token de cloudflared**: el token original se compartió en chat durante el planning. Rotarlo desde `Networks → Tunnels → homelab-k8s → ⋯ → Refresh token`, pegar el nuevo valor en Infisical (`/infrastructure/cloudflared`, key `token`). El operator re-sincroniza el `Secret` en ≤60s; basta con `kubectl rollout restart deploy/cloudflared -n cloudflared` para que tome el token nuevo. Sin commits.

## Tailnet exposure (Tailscale K8s Operator)

Servicios internos del cluster que necesitan ser alcanzables desde devices del tailnet (la MacBook, otros nodos) se exponen vía el **Tailscale Operator** en el namespace `tailscale`. Patrón:

- El operator corre como Deployment en `tailscale`, autenticado vía OAuth client (credenciales en Infisical, materializadas como Secret `operator-oauth` con keys `client_id` + `client_secret`).
- Cada Service con `spec.type: LoadBalancer` + `spec.loadBalancerClass: tailscale` dispara la creación automática de un sidecar `tailscaled` (pod `ts-<service>` en el namespace `tailscale`) que se une al tailnet como un device propio.
- Anotaciones por Service: `tailscale.com/hostname: <name>` (MagicDNS → `<name>.tail90f0a7.ts.net`) y `tailscale.com/tags: tag:k8s-db` (para ACL).
- **Apps in-cluster siguen sin tocar Tailscale** — usan la ClusterIP/DNS interna del CNPG (`<cluster>-rw.databases.svc.cluster.local:5432`). El operator solo añade un camino paralelo desde el tailnet.

### Servicios expuestos actualmente

| Service tailnet | DB backend | Hostname tailnet |
|---|---|---|
| `chapatuplaza-tailnet` | `chapatuplaza` CNPG primary | `chapatuplaza.tail90f0a7.ts.net:5432` |
| `n8n-postgres-tailnet` | `n8n-postgres` CNPG primary | `n8n-postgres.tail90f0a7.ts.net:5432` |
| `evolution-postgres-tailnet` | `evolution-postgres` CNPG primary | `evolution-postgres.tail90f0a7.ts.net:5432` |

### Pre-requisitos manuales en Tailscale (no GitOps)

- **OAuth client** en Tailscale: `https://login.tailscale.com/admin/settings/trust-credentials/add` (Tailscale movió "OAuth clients" a la sección "Trust credentials"). Scopes write: `Devices Core`, `Auth Keys`, `Services`. Tag asignado al client: `tag:k8s-operator`.
- **ACL** (sintaxis `grants` moderna, no `acls` legacy) debe definir `tagOwners`:
  ```jsonc
  "tagOwners": {
    "tag:k8s-operator": [],
    "tag:k8s-db":       ["tag:k8s-operator"]
  }
  ```
- Si tu tailnet ya tiene el grant default `{"src": ["*"], "dst": ["*"], "ip": ["*"]}`, los DBs son alcanzables sin reglas extra. Para endurecer (recomendado a futuro), sustituirlo por grants explícitos: `{"src":["oscar.gi.cast@gmail.com"],"dst":["tag:k8s-db"],"ip":["tcp:5432"]}`.

### Selector de CNPG

El Service tailnet selecciona el primario por `cnpg.io/cluster: <name>` + `role: primary`. CNPG mantiene la label `role: primary` en el pod activo durante failovers (relevante si en el futuro se escala a multi-réplica). Para exponer también réplicas, replicar el Service con `role: replica`.

### Rotación del OAuth client

Generar uno nuevo en Tailscale, actualizar Infisical (`/infrastructure/tailscale-operator`, keys `client_id` + `client_secret`), `kubectl rollout restart deploy/operator -n tailscale`. Sin commits.

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
| **kube-prometheus-stack 85.x** | Default `podMonitorSelector` filters by `release: <chart>` label; PodMonitors emitted by other operators (CNPG, etc.) are ignored. Set `podMonitorSelectorNilUsesHelmValues: false` + `podMonitorSelector: {}` (and the same for `serviceMonitor*`) under `prometheusSpec` to discover cluster-wide. |
| **kube-prometheus-stack 85.x** | Ships Prometheus v3 + node-exporter as `distroless` images by default (tags suffixed `-distroless`). If using a registry mirror, sync the `-distroless` tags. CRD updates must be applied manually before sync (`kubectl apply --server-side` against the `v0.90.1` operator manifests) — Helm v3 does not upgrade CRDs and ArgoCD's `ServerSideApply=true` covers chart resources but not the CRDs themselves. |
| **kube-prometheus-stack 67.x+** | Several high-cardinality metrics are dropped by default: `apiserver_request_sli_duration_seconds_bucket`, `apiserver_request_slo_duration_seconds_bucket`, `etcd_request_duration_seconds_bucket`, `csi_operations_seconds_bucket`, `storage_operation_duration_seconds_bucket`, `container_memory_failures_total{scope="hierarchy"}`, and `container_network_*` (CNI interfaces). If a dashboard or alert breaks after upgrading, check whether it queries one of these. |
| **CloudNativePG 0.28.x** | The operator chart does NOT ship a Grafana dashboard ConfigMap — the `monitoring.grafanaDashboard.create` value lives in the `cluster` subchart only. To get the dashboard, sideload a manual ConfigMap with the JSON from grafana.com (ID `20417`, dashboard UID `cloudnative-pg`) labeled `grafana_dashboard: "1"` in the `observability` namespace so Grafana's sidecar picks it up. |
| **CloudNativePG 0.28.x** | Set `spec.monitoring.enablePodMonitor: true` on each `Cluster` resource to expose metrics on port 9187 and emit a `PodMonitor` (which Prometheus needs the relaxed selectors above to discover). |
| **CloudNativePG (any version)** | **Pinear `spec.imageName` siempre.** Sin pin, el cluster hereda el default del operator, que cambia con cada bump del chart. CNPG NO ejecuta `pg_upgrade` automático — cuando el operator suba a un major nuevo (PG 19, etc.), los pods intentan arrancar el binario nuevo sobre PGDATA del major anterior y entran en CrashLoopBackOff. Pin a un tag rolling de minor específico (`ghcr.io/cloudnative-pg/postgresql:18.3-system-trixie`) o un digest exacto. Major upgrades = dump/restore manual deliberado. |
| **loki 13.x** | `loki.schemaConfig` is required — the chart refuses to render without it. Use schema `v13` / store `tsdb`. |
| **loki 13.x** | `chunksCache.enabled: false` and `resultsCache.enabled: false` are needed on single-node (memcached StatefulSets exhaust RAM). |
| **loki chart repo split (2026-03)** | The Grafana-maintained `grafana.github.io/helm-charts` `loki` chart is now GEL-only from `7.0.0` onward. OSS users moved to the community fork at `grafana-community.github.io/helm-charts` (forked at `6.55.0`, now on `13.x`). When upgrading from a stale `6.*` pin, change BOTH `repoURL` AND `targetRevision` in the same commit; the values schema is compatible. |
| **Cloudflare Tunnel** | Adding a Public Hostname _should_ auto-create the corresponding CNAME in CF DNS, but it **fails silently** in some flows (especially for hostnames without a Cloudflare Access app on top, e.g. n8n). After adding each Public Hostname, verify with `dig <host> @1.1.1.1 +short` — it must return CF anycast IPs (`104.21.x.x`, `172.67.x.x`). If `NXDOMAIN`, create the CNAME manually: target `<TUNNEL_UUID>.cfargotunnel.com`, orange-cloud (Proxied). |
| **Tailscale MagicDNS** | The `100.100.100.100` resolver caches `NXDOMAIN` according to the upstream SOA negative TTL (Cloudflare = 1800s = 30 min). `dscacheutil -flushcache` does NOT clear Tailscale's cache. To bypass: `dig <host> @1.1.1.1`. To force refresh: `tailscale down && tailscale up`. |
| **infrastructure parent Application** | When a subdir under `infrastructure/` contains manifests beyond `application.yaml` (e.g. `infrastructure/cloudflared/deployment.yaml`), the parent in `clusters/mac-mini/infrastructure.yaml` must use `directory.include: "{**/application.yaml,argocd-config/*.yaml,namespaces/*.yaml}"`. Without that filter, the parent picks up those manifests with destination `argocd` namespace, racing the child Application. |
| **CF Tunnel 404 diagnosis** | A `404 page not found\n` body with `content-length: 19` is Go's default `http.NotFound` — returned by both **cloudflared** (catch-all `service: http_status:404` rule) and **Traefik** (no router/endpoint match). To distinguish: `kubectl logs -n cloudflared deploy/cloudflared \| grep configuration` — if the host appears in the latest config, the 404 is from Traefik (check `kubectl describe ingress` for empty `Backends`). If the host is absent, it's cloudflared's catch-all (re-add the Public Hostname in CF dashboard). |
| **Argo CD `argocd-cmd-params-cm`** | Changes to this ConfigMap (e.g. flipping `server.insecure: "true"`) **do NOT auto-restart `argocd-server`**. The pod reads params only at startup. Symptom: applied via GitOps but pod still serves with old settings (e.g. HTTPS in non-insecure mode → infinite-redirect loop behind a TLS-terminating tunnel). Fix: `kubectl rollout restart deployment/argocd-server -n argocd`. Verify the new pod logs show `tls: false` to confirm insecure mode is active. |
| **Evolution API v2.3.7** | Requires `CACHE_REDIS_ENABLED: "false"` + `CACHE_LOCAL_ENABLED: "true"`. Without these, Evolution tries to connect to a non-existent Redis and **silently breaks Baileys message processing**. Symptom: instance status "open" but `Messages/Contacts/Chats = 0` in metrics, no webhooks fire. Errors `redis disconnected` only show with `LOG_LEVEL` including `ERROR` scope (NOT shown with bare `LOG_LEVEL: "WARN"`). |
| **Evolution API v2.3.7** | Requires `hostNetwork: true` + `dnsPolicy: ClusterFirstWithHostNet`. Without hostNetwork, the post-QR connection gets stuck in "connecting" indefinitely — Baileys 7.0.0-rc.9 doesn't survive Colima's virtual network stack for long-lived WhatsApp WebSockets. |
| **Evolution API v2.3.7** | Port is configured via `SERVER_PORT` env var (NOT `PORT`). With `hostNetwork: true`, use 8085 — port 8080 is occupied by Traefik's port-forward on the host. Service + Ingress targetPort must match. |
| **Evolution API v2.3.7** | `LOG_LEVEL: "WARN"` alone hides critical Redis errors. Use explicit scopes list: `"ERROR,WARN,INFO,LOG,VERBOSE,WEBHOOKS,WEBSOCKET"`. Valid scopes: `ERROR, WARN, DEBUG, INFO, LOG, VERBOSE, DARK, WEBHOOKS, WEBSOCKET`. Without `WEBHOOKS` scope, webhook delivery attempts are invisible. |
| **Evolution API v2.3.7** | `PROMETHEUS_METRICS: "true"` + `METRICS_AUTH_REQUIRED: "false"` enables `/metrics`. Without both, the endpoint returns 404. ServiceMonitor selector matches by label `app.kubernetes.io/name: evolution-api` on the Service (already set). Exposed metrics: `evolution_instances_total`, `evolution_instance_up`, `evolution_instance_state`. |
| **Evolution API v2.3.7** | Phone numbers for the API (instance creation, send message) are entered **without `+`**, **without spaces**, **without dashes**. Just country code + number. Example for Peru: `51907898162`. Internal JID format: `<número>@s.whatsapp.net`. |
| **Evolution API v2.3.7** | Changing image version (even patch) requires full DB wipe with `DROP SCHEMA public CASCADE; CREATE SCHEMA public;` on the `evolution` database — Prisma migrations don't tolerate cross-version downgrades, and the bundled Baileys keeps stale crypto session state. Run from `psql` against `evolution-postgres-rw.databases.svc.cluster.local:5432`. |
| **Evolution API v2.3.7** | Stream error code 515 after QR scan is **NORMAL** — WhatsApp asks the client to restart with saved creds. The reconnection is automatic. Don't try to "fix" the 515 itself; verify instead that messages flow afterwards (`evolution_instance_up = 1` and message count grows). |
| **n8n webhooks from cluster** | When Evolution API (or any cluster workload) needs to call n8n webhooks, use internal DNS: `http://n8n.automation.svc.cluster.local/webhook/<UUID>` — NOT the public URL (`https://n8n.oscargicast.com/webhook/...`), which would hairpin through CF Tunnel needlessly. The `webhook-test/...` URLs are ephemeral and only respond while the n8n editor is open; production needs `/webhook/...` and the workflow must be active. |
| **Evolution API calls from cluster** | Inverse of "n8n webhooks from cluster": when n8n (or any cluster workload) needs to call Evolution API, use internal DNS: `http://evolution-api.automation.svc.cluster.local:8085/...` — NOT the public URL (`https://evolution.oscargicast.com/...`), which is protected by a Self-hosted Cloudflare Access application and returns a sign-in HTML page (HTTP 200, `<title>Sign in ・ Cloudflare Access</title>`) instead of the API JSON response. The `apikey` header still applies. The Evolution API `PROXY_*` env vars are unrelated — they control Evolution → WhatsApp outbound, not how clients reach the pod. |
| **speedtest-tracker (lscr.io/linuxserver) 1.x** | Requires `APP_KEY` (Laravel) as a `base64:<32-byte>` value, gestionado vía Infisical (`/homelab/speedtest-tracker`). The widget native `speedtest` v2 in Homepage requires `version: 2` + `key: <bearer>` where the bearer is a **Personal Access Token** generated FROM THE UI (Settings → API Keys) AFTER first login — it is NOT the same `APP_KEY` and NOT the same `PROMETHEUS_API_KEY`. The token must be injected into Homepage as `HOMEPAGE_VAR_SPEEDTEST_KEY` via el `Secret` materializado por el `InfisicalSecret` de Homepage (`homelab/homepage/infisical-secret.yaml`) consumido por `envFrom.secretRef` en los chart values — Homepage interpolates `{{HOMEPAGE_VAR_*}}` placeholders at render time. Prometheus scrape endpoint is `/api/v1/prometheus` (not `/metrics`) and requires a Bearer token in `Authorization`. UI is reachable only via Tailscale (`speedtest.oscargicast.com`, CNAME DNS-only). |
| **Homepage secret injection** | The `jameswynn/homepage` chart v2.0.2 supports both `env:` (map → name/value pairs) and `envFrom:` (list → secretRef/configMapRef). To consume `HOMEPAGE_VAR_*` placeholders in `services.yaml` from el `Secret` materializado por Infisical, add the secret name under `envFrom.secretRef` at the top level of `values.yaml`. The Application must `include` the `InfisicalSecret` file in its `directory.include` filter — e.g. `include: "{ingress-public.yaml,infisical-secret.yaml}"` — otherwise ArgoCD won't sync it. |

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

Además del usuario `admin`, hay una cuenta local `oscargicast` con rol `admin` definida en `infrastructure/argocd-config/argocd-cm.yaml` + `argocd-rbac-cm.yaml`. Habilita login web + tokens para CLI sin compartir el password de admin. Setear su password inicial con `argocd account update-password --account oscargicast`.

> Si necesitás acceso de emergencia con el tunnel caído, el fallback temporal es `kubectl port-forward -n argocd svc/argocd-server --address 0.0.0.0 8443:80` (notar `80`, no `443`, porque ahora corre en `--insecure` mode) y abrir `http://<tailscale-ip>:8443`. Usar solo en emergencia.
