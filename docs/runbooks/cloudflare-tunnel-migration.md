# Runbook: migrar a subdomain-based routing con Cloudflare Tunnel

Migra **n8n, Grafana, Homepage y Argo CD** a subdominios públicos vía Cloudflare Tunnel (los 3 últimos detrás de Cloudflare Access), y **Prometheus** a un subdominio interno (resuelto solo en tailnet via CNAME). Resultado: cero hacks de routing path-based en el repo.

> Notas históricas: la migración inicial cubrió n8n/Grafana/Homepage/Prometheus. Argo CD se sumó como **follow-up** después — ver "Follow-up: exponer Argo CD" al final.

## Resumen de cambios

| Servicio | Antes | Después | Acceso |
|---|---|---|---|
| n8n | `oscar-mini-m1.tail90f0a7.ts.net:8080/n8n/` | `https://n8n.oscargicast.com` | Público (auth propia) |
| Grafana | `...:8080/grafana` | `https://grafana.oscargicast.com` | Público + CF Access |
| Homepage | `...:8080/` (host vacío) | `https://homepage.oscargicast.com` | Público + CF Access |
| Argo CD | `...:8443` (port-forward) | `https://argocd.oscargicast.com` | Público + CF Access (+ admin password) |
| Prometheus | `...:8080/prometheus` | `http://prometheus.oscargicast.com:8080` | Solo tailnet |

## DNS records que vas a tener

Todos en la zona `oscargicast.com` en Cloudflare.

| # | Hostname | Type | Target | Proxy | Origen |
|---|---|---|---|---|---|
| 1 | `n8n.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Auto-creado por CF al agregar Public Hostname |
| 2 | `grafana.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Auto |
| 3 | `homepage.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Auto |
| 4 | `argocd.oscargicast.com` | CNAME | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied | Public Hostname del tunnel |
| 5 | `prometheus.oscargicast.com` | CNAME | `oscar-mini-m1.tail90f0a7.ts.net` | ⚫ DNS only | **Manual** |

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

### 3a — Public Hostnames del tunnel (Published Applications)

`https://one.dash.cloudflare.com/` → Networks → Tunnels → `homelab-k8s` → tab **Public Hostname** (o **Routes → Add a route → Published application** según UI) → 3 veces:

- [ ] `n8n.oscargicast.com` → HTTP → `traefik.traefik.svc.cluster.local:80`
- [ ] `grafana.oscargicast.com` → HTTP → `traefik.traefik.svc.cluster.local:80`
- [ ] `homepage.oscargicast.com` → HTTP → `traefik.traefik.svc.cluster.local:80`

> **CF debería** auto-crear los 3 CNAMEs proxied en la zona oscargicast.com.

#### Verificación crítica (no asumas que se crearon)

`https://dash.cloudflare.com/` → `oscargicast.com` → DNS → Records. Confirmá que existen los 3 CNAMEs:

| Name | Target | Proxy |
|---|---|---|
| `n8n` | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied |
| `grafana` | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied |
| `homepage` | `<TUNNEL_UUID>.cfargotunnel.com` | ☁️ Proxied |

Y desde tu MacBook:
```bash
dig n8n.oscargicast.com @1.1.1.1 +short
dig grafana.oscargicast.com @1.1.1.1 +short
dig homepage.oscargicast.com @1.1.1.1 +short
```

Los 3 deben devolver IPs anycast de Cloudflare (`104.21.x.x`, `172.67.x.x`).

#### Si algún CNAME falta

A veces (depende del flujo UI usado) el auto-create falla silenciosamente. Crear manualmente: DNS → Add record → Type `CNAME`, Name `<subdominio>`, Target `<TUNNEL_UUID>.cfargotunnel.com`, **Proxy on** (orange-cloud), Save.

> El `<TUNNEL_UUID>` se obtiene de los logs de cloudflared (`tunnelID=...`) o del dashboard del tunnel.

> **Observación**: en flujos donde después se agregan Access apps (3b), CF puede crear el CNAME al guardar la Access app — por eso los servicios sin Access (como n8n) son los que más probablemente requieran creación manual.

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
- [ ] **Argo CD**: ver "Follow-up: exponer Argo CD" más abajo (en la migración inicial seguía como port-forward `:8443`; después se movió a `https://argocd.oscargicast.com`).
- [ ] **Webhook real**: workflow con webhook trigger en n8n + POST externo (`curl` desde Cloudflare Workers, webhook.site, etc.) → llega a n8n

---

## Fase 6 — Cleanup

- [ ] **Rotar el token** de cloudflared (porque el token se compartió en chat durante el planning):
  - CF Zero Trust → Networks → Tunnels → homelab-k8s → ⋯ → Refresh token
  - Repetir Fase 2 con el token nuevo
  - `git commit` + push del nuevo `sealed-secret.yaml`
  - Verificar que cloudflared sigue conectado después del rolling update
- [ ] Monitorear logs de cloudflared y métricas de Traefik durante un par de días

---

## Lessons learned (deploy real)

Estos son los issues que aparecieron al ejecutar este runbook por primera vez. Si los enfrentás de nuevo, ya sabés cómo diagnosticarlos.

### 1. La auto-creación del CNAME en CF DNS puede fallar silenciosamente

Al agregar Public Hostnames en el dashboard del tunnel (especialmente vía la nueva UI **Routes → Add a route → Published application**), Cloudflare _debería_ crear automáticamente el CNAME `<host>.oscargicast.com → <UUID>.cfargotunnel.com` en la zona DNS. **No siempre lo hace.**

En este deploy: `grafana` y `homepage` se crearon (probablemente por el flujo de Access app que los reconcilió). `n8n` NO se creó.

**Síntoma**: `dig n8n.oscargicast.com @1.1.1.1` devuelve `NXDOMAIN`.

**Solución**:
1. `https://dash.cloudflare.com/` → `oscargicast.com` → DNS → Records → Add record
2. Type `CNAME`, Name `<subdominio>`, Target `<TUNNEL_UUID>.cfargotunnel.com`, **Proxy on** (orange-cloud)
3. El TUNNEL_UUID se ve en los logs de cloudflared (`grep tunnelID`)

**Prevención**: después de agregar cada Public Hostname, validar inmediatamente con `dig <host> @1.1.1.1 +short` que devuelva IPs anycast de CF (`104.21.x.x`, `172.67.x.x`).

### 2. El Service de n8n expone puerto 80, no 5678

El chart `8gears/n8n:2.0.1` crea un `Service` que expone **port 80** (mapea internamente al container port `5678` del pod n8n). Si creás un `Ingress` separado del chart y le ponés `port.number: 5678`, Traefik no encuentra endpoints válidos en ese port y devuelve `404 page not found` con `content-length: 19`.

**Síntoma**: `kubectl describe ingress -n automation n8n-public` muestra `Backends: n8n:5678 ()` con paréntesis vacíos.

**Solución**: cambiar `port.number` a `80` en el Ingress.

**Prevención**: antes de definir un `Ingress` apuntando a un Service de chart, verificar el puerto real con `kubectl get svc -n <namespace>`.

### 3. Tailscale MagicDNS cachea NXDOMAIN

El resolver `100.100.100.100` (Tailscale) cachea respuestas negativas según el TTL del SOA upstream. Cloudflare devuelve `1800` segundos (30 min) en su SOA. Resultado: aunque el CNAME ya exista en CF DNS, el sistema sigue viendo `NXDOMAIN` por hasta 30 min.

**Síntoma**: `dig <host> @1.1.1.1` resuelve correctamente (con IPs anycast), pero `curl <host>` falla con `Could not resolve host`. Es decir: query directa al upstream funciona, pero el resolver del sistema (`getaddrinfo`) no.

**Diagnóstico**:
```bash
dig <host>                    # usa system resolver (Tailscale)
dig <host> @1.1.1.1           # bypass directo a Cloudflare
scutil --dns | head -10       # ver qué nameservers usa macOS
```

Si `scutil --dns` muestra `nameserver[0] : 100.100.100.100`, Tailscale está interceptando.

**Solución**: `tailscale down && sleep 2 && tailscale up` resetea el cache de Tailscale. Esperar ~5s después.

> Nota: `sudo dscacheutil -flushcache` y `sudo killall -HUP mDNSResponder` solo limpian el cache local de macOS, **NO** afectan al cache de Tailscale.

### 4. Distinguir 404 de cloudflared vs 404 de Traefik

Ambos devuelven exactamente la misma firma: body `404 page not found\n`, `content-length: 19`, `text/plain` — porque ambos usan Go's `http.NotFound` por default. Hay que diferenciarlos para saber dónde está el problema.

**cloudflared catch-all** (`service: http_status:404`): la request llegó al pod pero ningún ingress rule matchea el Host. Indica que la Public Hostname no está registrada en el config actual del tunnel.

**Traefik no-route**: la request llegó a Traefik pero no hay router/endpoint backend válido. Indica un problema en el `Ingress` (host mal escrito, backend Service inexistente, port equivocado).

**Diagnóstico**:
```bash
# 1. Ver el config actual de cloudflared
kubectl logs -n cloudflared deploy/cloudflared --tail=300 | grep configuration | tail -3

# 2a. Si el host SÍ aparece en la última "Updated to new configuration":
#     → 404 es de Traefik. Revisar el Ingress:
kubectl describe ingress -n <ns> <ingress-name>
# Buscar línea "Backends: <svc>:<port> (<endpoints>)" — los paréntesis NO deben estar vacíos.

# 2b. Si el host NO aparece en el config:
#     → 404 es de cloudflared. Ir al dashboard CF, agregar la Public Hostname.
```

---

## Follow-up: exponer Argo CD

Pasado el deploy inicial, Argo CD también se migró a subdomain público (`argocd.oscargicast.com`) detrás de Cloudflare Access. El patrón es idéntico al de los otros servicios, con una particularidad: argocd-server por default sirve **HTTPS con cert self-signed**, lo cual no se lleva bien con un tunnel que termina TLS al edge. Solución: setear `--insecure` para que sirva HTTP plano internamente.

### Pasos

1. **`infrastructure/argocd-config/params-cm.yaml`**: agregar `data: { server.insecure: "true" }` (override del ConfigMap `argocd-cmd-params-cm`).
2. **`infrastructure/argocd-config/ingress-public.yaml`** (nuevo): `Ingress` host-based `argocd.oscargicast.com` → `argocd-server:80`.
3. **CF dashboard** (mismo flujo que para los otros):
   - Public Hostname `argocd.oscargicast.com` → `traefik.traefik.svc.cluster.local:80`
   - Verificar el CNAME (probablemente requiere creación manual, ver lección #1)
   - Access app self-hosted con policy email
4. `git commit && git push` → ArgoCD aplica.
5. **Importante** (ver lección #5 abajo): forzar restart de `argocd-server` para que tome `--insecure`.
6. Detener el port-forward `:8443` de Argo CD (queda obsoleto).
7. (Recomendado) Rotar la admin password de Argo CD vía UI.

### 5. ArgoCD `argocd-cmd-params-cm` no auto-restartea argocd-server

Cambios en el ConfigMap `argocd-cmd-params-cm` (ej. flippear `server.insecure: "true"`) **no provocan** un rolling restart del `Deployment argocd-server`. Argo CD lo lee solo al startup. ArgoCD aplica el cambio del ConfigMap vía GitOps, pero el pod sigue corriendo con la config vieja.

**Síntoma**: después de hacer push y configurar el tunnel, `https://argocd.oscargicast.com` da `ERR_TOO_MANY_REDIRECTS`. argocd-server sigue serviendo HTTPS, recibe HTTP del tunnel y redirige a HTTPS, generando un loop.

**Diagnóstico**:
```bash
# El AGE del pod debe ser bajo (post-cambio del CM); si tiene varios días, no se reinició:
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server

# Logs del nuevo pod deben mostrar "tls: false":
kubectl logs -n argocd deploy/argocd-server --tail=20 | grep "serving on port"
# Esperado: "argocd v3.x.y serving on port 8080 (... tls: false ...)"
```

**Fix**:
```bash
kubectl rollout restart deployment/argocd-server -n argocd
kubectl rollout status deployment/argocd-server -n argocd
```

**Prevención**: incluir el `kubectl rollout restart` como paso obligatorio cada vez que se modifica `argocd-cmd-params-cm` (o `argocd-cm`), o agregar una annotation/label que cambie en el Deployment para forzar el rollout vía GitOps.

