# Homelab Kubernetes — Mac mini M1

Laboratorio Kubernetes personal corriendo en una **Mac mini M1 headless**, gestionado con **GitOps (Argo CD)**.

```
MacBook Pro
  └── SSH vía Tailscale
        └── Mac mini M1
              └── Colima VM
                    └── Kubernetes (K3s)
                          ├── Argo CD          ← GitOps controller
                          ├── Sealed Secrets   ← gestión de credenciales
                          ├── CloudNativePG    ← operador PostgreSQL
                          ├── PostgreSQL       ← postgres-lab + n8n-postgres
                          ├── Prometheus + Grafana + Loki ← observabilidad
                          ├── Traefik          ← ingress controller
                          ├── n8n              ← automatización de workflows
                          └── Homepage         ← dashboard del homelab
```

## Cómo funciona

Este repo usa el patrón **App of Apps** de Argo CD. Hacer push a `main` es suficiente para que los cambios se apliquen al cluster automáticamente.

```
git push → Argo CD detecta el cambio → aplica al cluster
```

El único `kubectl apply` manual es el bootstrap inicial:

```bash
# Desde Mac mini, una sola vez:
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Stack

| Componente | Rol | Namespace |
|---|---|---|
| Argo CD | GitOps controller | `argocd` |
| Sealed Secrets | Credenciales encriptadas en Git | `kube-system` |
| CloudNativePG | Operador PostgreSQL | `cnpg-system` |
| postgres-lab | PostgreSQL de pruebas | `databases` |
| n8n-postgres | PostgreSQL dedicado para n8n | `databases` |
| kube-prometheus-stack | Métricas (Prometheus + Grafana) | `observability` |
| Loki | Logs centralizados | `observability` |
| Traefik | Ingress controller (routing por path) | `traefik` |
| n8n | Automatización de workflows | `automation` |
| Homepage | Dashboard visual del homelab | `homelab` |

## Estructura del repo

```
homelab-k8s/
├── bootstrap/argocd/root-app.yaml      # bootstrap manual (una sola vez)
├── clusters/mac-mini/                  # Apps de nivel cluster
│   ├── apps.yaml                       # databases + automation + homelab
│   ├── infrastructure.yaml             # namespaces + operadores
│   └── observability.yaml
├── infrastructure/
│   ├── namespaces/                     # Namespace CRs
│   ├── sealed-secrets/                 # Sealed Secrets operator
│   ├── cloudnative-pg/                 # CNPG operator
│   ├── traefik/                        # Traefik ingress controller
│   └── argocd-config/                  # argocd-cmd-params-cm overrides
├── databases/
│   ├── postgres-lab/                   # cluster.yaml + sealed-secret.yaml
│   └── n8n-postgres/
├── observability/
│   ├── prometheus/                     # values.yaml
│   ├── loki/                           # values.yaml
│   └── grafana-dashboards/             # ConfigMaps con dashboards (CNPG)
├── automation/n8n/
└── homelab/homepage/
```

## Gestión de secrets

Los secrets se encriptan con **Sealed Secrets** antes de commitearlos. El archivo `secret.yaml` está en `.gitignore` — nunca va a Git.

```bash
# Flujo para agregar un secret:
kubeseal --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem \
  --format yaml < secret.yaml > sealed-secret.yaml

git add sealed-secret.yaml && git commit -m "feat: add sealed secret"
rm secret.yaml  # borrar el plaintext
```

## Bootstrap (primera vez)

```bash
# 1. Instalar Argo CD en Mac mini (--server-side evita error de annotation demasiado larga)
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Instalar kubeseal (con brew en Mac mini M1) y obtener el cert
brew install kubeseal
kubeseal --fetch-cert \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  > ~/.config/homelab-k8s-sealed-secrets-pub.pem

# 3. Sellar los secrets de las bases de datos
# Ver CLAUDE.md para el flujo completo

# 4. Aplicar el root Application (único apply manual)
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Acceder a los servicios

Todos los servicios se sirven a través de Traefik en un único punto de entrada:

```bash
# Port-forward desde Mac mini:
kubectl port-forward -n traefik svc/traefik --address 0.0.0.0 8080:80

# Servicios disponibles en http://oscar-mini-m1.tail90f0a7.ts.net:8080/
# - Homepage:        /                   (con widgets de Postgres en vivo)
# - Grafana:         /grafana            (dashboard CNPG en folder "Databases")
# - Prometheus:      /prometheus
# - n8n:             /n8n/
```

## Acceder a Argo CD

```bash
# Port-forward desde Mac mini (puerto 8080 lo usa Traefik):
kubectl port-forward -n argocd svc/argocd-server --address 0.0.0.0 8443:443

# Abrir: https://<tailscale-ip>:8443
# Usuario: admin
# Contraseña:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## Recursos del cluster

| Recurso | Asignado |
|---|---|
| CPU | 4 vCPU |
| RAM | 8 GB |
| Disco | 200 GB |
| StorageClass | `local-path` (single-node) |

> Cluster single-node — no hay alta disponibilidad real. Ideal para aprendizaje y servicios personales.
