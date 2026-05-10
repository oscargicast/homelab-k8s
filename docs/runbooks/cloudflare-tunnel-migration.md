# Runbook: migrar a subdomain-based routing con Cloudflare Tunnel

Migra **n8n, Grafana y Homepage** a subdominios públicos vía Cloudflare Tunnel, y **Prometheus** a un subdominio interno (resuelto solo en tailnet via CNAME). Resultado: cero hacks de routing path-based en el repo.

## Resumen de cambios

| Servicio | Antes | Después | Acceso |
|---|---|---|---|
| n8n | `oscar-mini-m1.tail90f0a7.ts.net:8080/n8n/` | `https://n8n.oscargicast.com` | Público (auth propia) |
| Grafana | `...:8080/grafana` | `https://grafana.oscargicast.com` | Público + CF Access |
| Homepage | `...:8080/` (host vacío) | `https://homepage.oscargicast.com` | Público + CF Access |
| Prometheus | `...:8080/prometheus` | `http://prometheus.oscargicast.com:8080` | Solo tailnet |
| Argo CD | `...:8443` (port-forward) | igual | Solo tailnet |

## DNS records que vas a tener

Todos en la zona `oscargicast.com` en Cloudflare.

| # | Hostname | Type | Target | Proxy | Origen |
|---|---|---|---|---|---|
| 1 | `n8n.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Auto-creado por CF al agregar Public Hostname |
| 2 | `grafana.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Auto |
| 3 | `homepage.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Auto |
| 4 | `prometheus.oscargicast.com` | CNAME | `oscar-mini-m1.tail90f0a7.ts.net` | ⚫ DNS only | **Manual** |

> Crítico: el #4 NO debe estar proxied — el proxy de CF no puede llegar a una IP Tailscale privada. Tiene que ser grey-cloud (DNS only).

## Puertos en el Mac mini

Sin cambios. Los port-forwards actuales siguen necesarios:

```bash
kubectl port-forward -n traefik svc/traefik --address 0.0.0.0 8080:80
kubectl port-forward -n argocd svc/argocd-server --address 0.0.0.0 8443:443
```

cloudflared corre dentro del cluster (Deployment con 2 réplicas) y se conecta saliente a Cloudflare. **No expone ningún puerto al host ni al cluster** — la conexión es siempre outbound por QUIC/HTTPS.

---

## Fase 0 — Pre-checks (en Mac mini)

```bash
ssh macmini
```

- [ ] `kubectl get nodes` y `kubectl get pods -A` sin errores
- [ ] Port-forwards activos:
  - `kubectl port-forward -n traefik svc/traefik --address 0.0.0.0 8080:80`
  - `kubectl port-forward -n argocd svc/argocd-server --address 0.0.0.0 8443:443`
- [ ] kubeseal cert disponible:
  ```bash
  ls ~/.config/homelab-k8s-sealed-secrets-pub.pem
  # Si no existe:
  kubeseal --fetch-cert \
    --controller-namespace kube-system \
    --controller-name sealed-secrets-controller \
    > ~/.config/homelab-k8s-sealed-secrets-pub.pem
  ```
- [ ] ArgoCD apps actuales todas Synced/Healthy: `kubectl get applications -n argocd`

---

## Fase 1 — Cambios en el repo (en MacBook)

### Crear archivos nuevos

- [ ] `infrastructure/namespaces/cloudflared.yaml`
- [ ] `infrastructure/cloudflared/application.yaml`
- [ ] `infrastructure/cloudflared/deployment.yaml`
- [ ] `automation/n8n/ingress-public.yaml`
- [ ] `observability/prometheus/grafana-ingress.yaml`
- [ ] `observability/prometheus/prometheus-ingress.yaml`
- [ ] `homelab/homepage/ingress-public.yaml`

### Modificar archivos existentes

- [ ] `automation/n8n/values.yaml`
  - quitar `N8N_PATH`, `N8N_EDITOR_BASE_URL`
  - cambiar `N8N_SECURE_COOKIE: "false"` → `"true"`
  - agregar `N8N_HOST`, `N8N_PROTOCOL: "https"`, `WEBHOOK_URL`
  - eliminar la sección `ingress:` completa
- [ ] `observability/prometheus/values.yaml`
  - eliminar `grafana.ingress`, `grafana.grafana.ini.server`
  - eliminar `prometheus.ingress`, `prometheus.prometheusSpec.routePrefix`, `externalUrl` con subpath
- [ ] `homelab/homepage/values.yaml`
  - eliminar bloque `ingress:`
  - actualizar 6 hrefs (Grafana x4, n8n x1, Prometheus x1)
  - actualizar 3 widget URLs (quitar `/prometheus` al final)

### Eliminar

- [ ] `automation/n8n/n8n-strip-prefix-middleware.yaml`

### Verificar `Application` (migrar a `sources:` plural si hace falta)

- [ ] `automation/n8n/application.yaml`
- [ ] `observability/prometheus/application.yaml`
- [ ] `homelab/homepage/application.yaml`

> ⚠️ **No commitees todavía** — falta el `sealed-secret.yaml` (Fase 2).

---

## Fase 2 — Sellar el token (en Mac mini)

Tener el token de cloudflared a mano (string que empieza con `eyJ...`).

```bash
ssh macmini
cd ~/Projects/homelab-k8s   # ajustar según ubicación real

kubectl create secret generic cloudflared-token \
  -n cloudflared \
  --from-literal=token='<PEGAR_TOKEN>' \
  --dry-run=client -o yaml > /tmp/cloudflared-token.yaml

kubeseal \
  --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem \
  --format yaml \
  < /tmp/cloudflared-token.yaml \
  > infrastructure/cloudflared/sealed-secret.yaml

rm /tmp/cloudflared-token.yaml
```

- [ ] Verificar que `infrastructure/cloudflared/sealed-secret.yaml` existe y tiene `kind: SealedSecret`

---

## Fase 3 — Configurar Cloudflare (browser)

### 3a — Public Hostnames del tunnel

`https://one.dash.cloudflare.com/` → Networks → Tunnels → `homelab-k8s` → tab **Public Hostname** → Add a public hostname (3 veces):

- [ ] `n8n.oscargicast.com` → HTTP → `traefik.traefik.svc.cluster.local:80`
- [ ] `grafana.oscargicast.com` → HTTP → `traefik.traefik.svc.cluster.local:80`
- [ ] `homepage.oscargicast.com` → HTTP → `traefik.traefik.svc.cluster.local:80`

> Esto crea automáticamente los 3 CNAME proxied en la zona oscargicast.com.

### 3b — Cloudflare Access policies

Zero Trust → Access → Applications → Add an application → Self-hosted (2 veces):

- [ ] App `homepage` → host `homepage.oscargicast.com` → policy email `oscar.gi.cast@gmail.com`
- [ ] App `grafana` → host `grafana.oscargicast.com` → mismo policy

### 3c — CNAME interno para Prometheus

`https://dash.cloudflare.com/` → oscargicast.com → DNS → Records → Add record:

- [ ] Type: `CNAME` | Name: `prometheus` | Target: `oscar-mini-m1.tail90f0a7.ts.net` | Proxy: **DNS only** (grey-cloud) | TTL: Auto

---

## Fase 4 — Push a Git

```bash
git status                  # revisar diff
git add <archivos cambiados>
git commit -m "feat: expose n8n/grafana/homepage via cloudflare tunnel + prometheus subdomain"
git push
```

ArgoCD sincroniza automáticamente.

---

## Fase 5 — Verificación

### En Mac mini

```bash
ssh macmini

# ArgoCD apps
kubectl get applications -n argocd | grep -E '(cloudflared|n8n|prometheus|homepage)'

# cloudflared corriendo
kubectl get pods -n cloudflared
kubectl logs -n cloudflared deploy/cloudflared --tail=30

# Ingresses
kubectl get ingress -A | grep oscargicast
```

- [ ] ArgoCD apps Synced/Healthy
- [ ] cloudflared 2/2 Running, logs muestran "Connection registered" x2
- [ ] Tunnel HEALTHY en CF dashboard (Networks → Tunnels → homelab-k8s) con 2 connectors
- [ ] 4 Ingresses creados con `oscargicast.com`

### Tests funcionales (desde MacBook)

- [ ] **n8n**: browser → `https://n8n.oscargicast.com` → login carga sin warning de cookies
- [ ] **Grafana**: browser → `https://grafana.oscargicast.com` → CF Access (Google) → Grafana login
- [ ] **Homepage**: browser → `https://homepage.oscargicast.com` → CF Access → dashboard. Widgets Prometheus muestran valores (no "API Error")
- [ ] **Prometheus**:
  ```bash
  dig prometheus.oscargicast.com +short
  curl -I http://prometheus.oscargicast.com:8080/
  ```
  Browser: `http://prometheus.oscargicast.com:8080/` → UI Prometheus en root
- [ ] **Argo CD**: `https://oscar-mini-m1.tail90f0a7.ts.net:8443` sigue funcionando
- [ ] **Webhook real**: workflow con webhook trigger en n8n + POST externo (`curl` desde Cloudflare Workers, webhook.site, etc.) → llega a n8n

---

## Fase 6 — Cleanup

- [ ] **Rotar el token** de cloudflared (porque el token se compartió en chat durante el planning):
  - CF Zero Trust → Networks → Tunnels → homelab-k8s → ⋯ → Refresh token
  - Repetir Fase 2 con el token nuevo
  - `git commit` + push del nuevo `sealed-secret.yaml`
  - Verificar que cloudflared sigue conectado después del rolling update
- [ ] Monitorear logs de cloudflared y métricas de Traefik durante un par de días
