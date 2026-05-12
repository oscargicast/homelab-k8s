# Homelab Kubernetes — Mac mini M1

Laboratorio Kubernetes personal corriendo en una **Mac mini M1 headless**, gestionado con **GitOps (Argo CD)** y expuesto al mundo vía **Cloudflare Tunnel** sobre el dominio `oscargicast.com`.

## Stack en el cluster

```mermaid
graph TD
    MBP[MacBook Pro]
    MBP -->|SSH via Tailscale| MM[Mac mini M1]
    MM --> COL[Colima VM]
    COL --> K8S[K3s en Kubernetes]

    K8S --> NS_ARGOCD[argocd<br/><i>Argo CD GitOps</i>]
    K8S --> NS_KS[kube-system<br/><i>Sealed Secrets operator (legacy, sin CRs)</i>]
    K8S --> NS_INF_OP[infisical-operator<br/><i>Infisical Secrets Operator</i>]
    K8S --> NS_INF[infisical<br/><i>Infisical backend + Redis</i>]
    K8S --> NS_CNPG[cnpg-system<br/><i>CloudNativePG operator</i>]
    K8S --> NS_DB[databases<br/><i>postgres-lab + n8n-postgres + evolution-postgres + infisical-postgres</i>]
    K8S --> NS_OBS[observability<br/><i>Prometheus + Grafana + Loki</i>]
    K8S --> NS_TRA[traefik<br/><i>Ingress controller por Host</i>]
    K8S --> NS_CFD[cloudflared<br/><i>Cloudflare Tunnel connector</i>]
    K8S --> NS_AUT[automation<br/><i>n8n + evolution-api</i>]
    K8S --> NS_HL[homelab<br/><i>Homepage + Speedtest Tracker</i>]
```

## Cómo se accede a cada servicio

```mermaid
graph LR
    USR_EXT[👤 Usuario en internet]
    USR_INT[👤 Usuario en tailnet]

    subgraph CF[Cloudflare]
        EDGE[Edge<br/>TLS + Access]
        TUNNEL[Tunnel]
    end

    subgraph CLUSTER[Cluster Mac mini]
        CFD[cloudflared pod<br/>2 réplicas]
        TRA[Traefik svc:80]
        PF8080[Mac mini host :8080<br/>kubectl port-forward]
        N8N[n8n.automation:80]
        EVO[evolution-api hostNetwork :8085]
        GRA[prometheus-grafana.observability:80]
        HOM[homepage.homelab:3000]
        ARGO[argocd-server.argocd:80]
        PROM[prometheus...:9090]
        SPD[speedtest-tracker.homelab:80]
    end

    USR_EXT -->|n8n.oscargicast.com| EDGE
    USR_EXT -->|evolution.oscargicast.com| EDGE
    USR_EXT -->|grafana.oscargicast.com| EDGE
    USR_EXT -->|homepage.oscargicast.com| EDGE
    USR_EXT -->|argocd.oscargicast.com| EDGE
    EDGE --> TUNNEL
    TUNNEL --> CFD
    CFD --> TRA
    TRA --> N8N
    TRA --> EVO
    TRA --> GRA
    TRA --> HOM
    TRA --> ARGO

    USR_INT -->|prometheus.oscargicast.com:8080<br/>CNAME→tailscale| PF8080
    USR_INT -->|speedtest.oscargicast.com<br/>CNAME→tailscale| PF8080
    PF8080 --> TRA
    TRA --> PROM
    TRA --> SPD
```

## Cómo funciona el GitOps

Patrón **App of Apps** de Argo CD. Hacer push a `main` es suficiente para que los cambios se apliquen al cluster automáticamente.

```
git push → Argo CD detecta el cambio → aplica al cluster
```

El único `kubectl apply` manual es el bootstrap inicial:

```bash
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Stack

| Componente | Rol | Namespace |
|---|---|---|
| Argo CD | GitOps controller | `argocd` |
| Infisical | Self-hosted secrets manager (fuente actual de verdad) | `infisical` |
| Infisical Secrets Operator | Materializa `InfisicalSecret` CRs → Secrets nativos | `infisical-operator` |
| Sealed Secrets | Credenciales encriptadas en Git (legacy, en migración a Infisical) | `kube-system` |
| CloudNativePG | Operador PostgreSQL | `cnpg-system` |
| postgres-lab | PostgreSQL de pruebas | `databases` |
| n8n-postgres | PostgreSQL dedicado para n8n | `databases` |
| evolution-postgres | PostgreSQL dedicado para Evolution API | `databases` |
| infisical-postgres | PostgreSQL dedicado para Infisical backend | `databases` |
| kube-prometheus-stack | Métricas (Prometheus + Grafana) | `observability` |
| Loki | Logs centralizados | `observability` |
| Traefik | Ingress controller (routing por Host) | `traefik` |
| cloudflared | Cloudflare Tunnel connector (saliente) | `cloudflared` |
| n8n | Automatización de workflows | `automation` |
| Evolution API | API de WhatsApp Web (integración con n8n) | `automation` |
| Homepage | Dashboard visual del homelab | `homelab` |
| Speedtest Tracker | Monitor de velocidad/latencia ISP (SQLite, scrape Prometheus) | `homelab` |

## Estructura del repo

```
homelab-k8s/
├── bootstrap/argocd/root-app.yaml      # bootstrap manual (una sola vez)
├── clusters/mac-mini/                  # Apps de nivel cluster
│   ├── apps.yaml                       # databases + automation + homelab
│   ├── infrastructure.yaml             # namespaces + operadores + cloudflared
│   └── observability.yaml
├── infrastructure/
│   ├── namespaces/                     # Namespace CRs
│   ├── sealed-secrets/                 # Sealed Secrets operator (legacy, sin CRs vivos)
│   ├── infisical/                      # Infisical backend (self-hosted)
│   ├── infisical-operator/             # Infisical Secrets Operator (materializa InfisicalSecret CRs)
│   ├── redis/                          # Redis dedicado para Infisical
│   ├── cloudnative-pg/                 # CNPG operator
│   ├── traefik/                        # Traefik ingress controller
│   ├── cloudflared/                    # CF Tunnel: deployment + infisical-secret + Application
│   └── argocd-config/                  # argocd-cmd-params-cm overrides
├── databases/
│   ├── postgres-lab/                   # cluster.yaml + infisical-secret.yaml
│   ├── n8n-postgres/
│   ├── evolution-postgres/             # CNPG cluster para Evolution API
│   └── infisical-postgres/             # CNPG cluster para Infisical backend
├── observability/
│   ├── prometheus/                     # values.yaml + ingresses (grafana, prometheus)
│   ├── loki/                           # values.yaml
│   └── grafana-dashboards/             # ConfigMaps con dashboards (CNPG)
├── automation/
│   ├── n8n/                            # values.yaml + ingress-public.yaml
│   └── evolution-api/                  # manifest-based + hostNetwork + ServiceMonitor
├── homelab/
│   ├── homepage/                       # values.yaml + ingress-public.yaml + infisical-secret.yaml
│   └── speedtest-tracker/              # manifest-based + ingress-internal + ServiceMonitor + infisical-secret.yaml
└── docs/runbooks/                      # runbooks de operación
```

## Gestión de secrets

Los secretos activos viven en **Infisical self-hosted** (`https://infisical.oscargicast.com`). El repo solo contiene `InfisicalSecret` CRs (punteros sin valores) que el Infisical Secrets Operator materializa como `Secret` nativos en ≤60s. Para crear/editar un secreto: editar el valor en la UI de Infisical + commitear el `InfisicalSecret` CR apuntando a su `secretsPath`. Ver [`docs/runbooks/infisical-migration.md`](docs/runbooks/infisical-migration.md) para el flujo completo y un ejemplo de CR.

El operator de Bitnami Sealed Secrets sigue desplegado en `kube-system` por seguridad histórica, pero **no hay `SealedSecret` CRs en Git**. El flujo `kubeseal` queda disponible como fallback de emergencia (ver runbook). `**/secret.yaml` sigue en `.gitignore` — nunca va plaintext a Git.

## Bootstrap (primera vez)

```bash
# 1. Instalar Argo CD en Mac mini (--server-side evita error de annotation demasiado larga)
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Bootstrap de Infisical: crear el Secret `infisical-bootstrap` (admin email/password
#    iniciales) y `infisical-redis-auth` en el namespace `infisical` antes del primer sync,
#    y luego configurar la Machine Identity (k8sAuth) en `infisical-operator`.
#    Detalles paso a paso en docs/runbooks/infisical-migration.md (Fase 0-2).

# 3. Aplicar el root Application (único apply manual)
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Acceder a los servicios

| Servicio | URL | Acceso |
|---|---|---|
| n8n | `https://n8n.oscargicast.com` | Público (auth propia) |
| Evolution API | `https://evolution.oscargicast.com` | Público + Cloudflare Access (manager UI + API key) |
| Grafana | `https://grafana.oscargicast.com` | Público + Cloudflare Access |
| Homepage | `https://homepage.oscargicast.com` | Público + Cloudflare Access |
| Argo CD | `https://argocd.oscargicast.com` | Público + Cloudflare Access (+ admin password de Argo CD) |
| Prometheus | `http://prometheus.oscargicast.com:8080` | Solo tailnet (CNAME→Tailscale) |
| Speedtest Tracker | `https://speedtest.oscargicast.com` | Solo tailnet (CNAME→Tailscale) |

Los públicos pasan por **Cloudflare Tunnel** (cloudflared corriendo en cluster con conexión saliente a CF edge), TLS terminado en CF, sin abrir puertos en la red.

Para Prometheus (interno por tailnet) hay que tener corriendo el port-forward de Traefik en Mac mini:

```bash
kubectl port-forward -n traefik svc/traefik --address 0.0.0.0 8080:80
```

Password inicial de Argo CD:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## DNS records en Cloudflare

| Hostname | Type | Target | Proxy | Origen |
|---|---|---|---|---|
| `n8n.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Public Hostname del tunnel (a veces requiere creación manual) |
| `evolution.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Public Hostname del tunnel |
| `grafana.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Public Hostname del tunnel |
| `homepage.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Public Hostname del tunnel |
| `argocd.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Public Hostname del tunnel |
| `prometheus.oscargicast.com` | CNAME | `oscar-mini-m1.tail90f0a7.ts.net` | ⚫ DNS only | Manual (no debe estar proxied — IP Tailscale es privada) |
| `speedtest.oscargicast.com` | CNAME | `oscar-mini-m1.tail90f0a7.ts.net` | ⚫ DNS only | Manual (no debe estar proxied — IP Tailscale es privada) |

## Recursos del cluster

| Recurso | Asignado |
|---|---|
| CPU | 4 vCPU |
| RAM | 8 GB |
| Disco | 200 GB |
| StorageClass | `local-path` (single-node) |

> Cluster single-node — no hay alta disponibilidad real. Ideal para aprendizaje y servicios personales.

## Documentación adicional

- [`docs/runbooks/infisical-migration.md`](docs/runbooks/infisical-migration.md) — runbook de la migración Sealed Secrets → Infisical, con flujo para crear/editar secretos
- [`docs/runbooks/cloudflare-tunnel-migration.md`](docs/runbooks/cloudflare-tunnel-migration.md) — runbook + lessons learned de la migración a subdomain-based routing
- [`docs/runbooks/evolution-api-setup.md`](docs/runbooks/evolution-api-setup.md) — runbook para desplegar Evolution API + WhatsApp + n8n
- [`docs/runbooks/secrets-cheatsheet.md`](docs/runbooks/secrets-cheatsheet.md) — comandos para extraer secrets del cluster
- [`docs/runbooks/db-access.md`](docs/runbooks/db-access.md) — acceso a las DBs CNPG vía port-forward + Tailscale
- [`docs/runbooks/speedtest-tracker-setup.md`](docs/runbooks/speedtest-tracker-setup.md) — runbook para desplegar speedtest-tracker + widget Homepage (3 tokens distintos)
- [`CLAUDE.md`](CLAUDE.md) — guía operativa completa con gotchas por chart
