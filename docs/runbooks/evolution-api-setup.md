# Runbook: Evolution API v2.3.7 — WhatsApp + n8n

Despliega [Evolution API](https://github.com/EvolutionAPI/evolution-api) en el cluster para conectar WhatsApp (cuenta personal) con n8n. Expone `https://evolution.oscargicast.com` detrás de Cloudflare Access.

## Arquitectura

| Componente | Namespace | Detalle |
|---|---|---|
| `evolution-postgres` (CNPG) | `databases` | PostgreSQL dedicado, persistencia de mensajes/contactos/chats/sesión |
| `evolution-api` (Deployment) | `automation` | Manifest-based, `hostNetwork: true`, puerto 8085 |
| `evolution-api` (Service) | `automation` | ClusterIP port 8085 (para Ingress + ServiceMonitor) |
| `evolution-api` (Ingress) | `automation` | Traefik → CF Tunnel → CF Access (Google) |
| `evolution-api` (ServiceMonitor) | `automation` | Prometheus scrapea `/metrics` cada 30s |

```
Browser → CF Edge → CF Access (Google) → CF Tunnel → Traefik → evolution-api:8085

WhatsApp servers ←(WebSocket saliente)← Evolution API pod (hostNetwork)
                                              ↕ DB connection
                                       evolution-postgres (CNPG)
                                              ↕ webhook (DNS interno)
                                       n8n.automation.svc.cluster.local
```

**Decisiones clave (no obvias):**
- **`hostNetwork: true`** — sin esto, post-QR queda en "connecting" para siempre. La red virtual de Colima no se lleva bien con los WebSockets de Baileys.
- **`CACHE_REDIS_ENABLED: "false"`** — Evolution intenta conectarse a Redis por default. Sin Redis, falla silenciosamente y rompe el procesamiento de mensajes (instancia "open" pero `Messages: 0`).
- **`LOG_LEVEL` con scopes** — `"WARN"` solo oculta los errores de Redis. Usar `"ERROR,WARN,INFO,LOG,VERBOSE,WEBHOOKS,WEBSOCKET"`.
- **Puerto 8085** — 8080 lo usa Traefik vía port-forward en el host de Colima.

---

## Fase 0 — Pre-checks (en Mac mini)

```bash
ssh macmini
```

- [ ] `kubectl get nodes` y `kubectl get pods -A` sin errores
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

## Fase 1 — Manifiestos (en MacBook)

Total: **9 archivos en 2 directorios**. ArgoCD los descubre automáticamente vía los parent Applications.

### `automation/evolution-api/`

| Archivo | Notas |
|---|---|
| `application.yaml` | ArgoCD Application, `directory.include: "{deployment.yaml,service.yaml,sealed-secret.yaml,ingress-public.yaml,service-monitor.yaml}"` |
| `deployment.yaml` | `hostNetwork: true` + `dnsPolicy: ClusterFirstWithHostNet` + `SERVER_PORT=8085` + `CACHE_REDIS_ENABLED=false` + `PROMETHEUS_METRICS=true` |
| `service.yaml` | ClusterIP port 8085 |
| `service-monitor.yaml` | Selector `app.kubernetes.io/name: evolution-api`, path `/metrics`, interval 30s |
| `ingress-public.yaml` | host `evolution.oscargicast.com` → service `evolution-api:8085` |
| `sealed-secret.yaml` | Generado en Fase 2 (API key + database URL) |

### `databases/evolution-postgres/`

| Archivo | Notas |
|---|---|
| `application.yaml` | ArgoCD Application, sync wave 0 |
| `cluster.yaml` | CNPG Cluster, `storage: 5Gi`, `local-path`, `monitoring.enablePodMonitor: true` |
| `sealed-secret.yaml` | Generado en Fase 2 (user/password CNPG) |

---

## Fase 2 — Sellar secrets (en Mac mini)

```bash
ssh macmini
cd ~/Projects/homelab-k8s

# 1. Generar passwords
DB_PASSWORD=$(openssl rand -base64 24)
API_KEY=$(openssl rand -base64 32)

# 2. Secret de DB (namespace: databases)
cat > /tmp/evolution-db-user.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: evolution-db-user
  namespace: databases
type: kubernetes.io/basic-auth
stringData:
  username: evolution
  password: ${DB_PASSWORD}
EOF

# 3. Secret de app (namespace: automation)
cat > /tmp/evolution-api-secrets.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: evolution-api-secrets
  namespace: automation
type: Opaque
stringData:
  api-key: ${API_KEY}
  database-url: "postgresql://evolution:${DB_PASSWORD}@evolution-postgres-rw.databases.svc.cluster.local:5432/evolution"
EOF

# 4. Sellar ambos
kubeseal --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem --format yaml \
  < /tmp/evolution-db-user.yaml > databases/evolution-postgres/sealed-secret.yaml

kubeseal --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem --format yaml \
  < /tmp/evolution-api-secrets.yaml > automation/evolution-api/sealed-secret.yaml

# 5. Guardar el API key (para autenticar con la API y el manager)
echo "=== GUARDAR ESTE API KEY ==="
echo "API Key: ${API_KEY}"
echo "============================"

# 6. Limpiar plaintext
rm /tmp/evolution-db-user.yaml /tmp/evolution-api-secrets.yaml
```

> Más adelante para recuperar el API key sin regenerarlo: `kubectl -n automation get secret evolution-api-secrets -o jsonpath='{.data.api-key}' | base64 -d` (ver `docs/runbooks/secrets-cheatsheet.md`).

---

## Fase 3 — Configurar Cloudflare (browser)

### 3a — Public Hostname del tunnel

`https://one.dash.cloudflare.com/` → Networks → Tunnels → `homelab-k8s` → Public Hostname → Add:

- [ ] Subdomain: `evolution`
- [ ] Domain: `oscargicast.com`
- [ ] Service: HTTP → `traefik.traefik.svc.cluster.local:80`

### 3b — Verificar CNAME

```bash
dig evolution.oscargicast.com @1.1.1.1 +short
```

- [ ] Debe devolver IPs anycast de CF (`104.21.x.x`, `172.67.x.x`)

> Si devuelve `NXDOMAIN`: crear CNAME manualmente. Type `CNAME`, Name `evolution`, Target `<TUNNEL_UUID>.cfargotunnel.com`, Proxy **ON** (orange-cloud). Ver lección #1 en `cloudflare-tunnel-migration.md`.

### 3c — Cloudflare Access

Zero Trust → Access → Applications → Add → Self-hosted:

- [ ] Name: `Evolution API`
- [ ] Application domain: `evolution.oscargicast.com`
- [ ] Policy: Allow → Emails → `oscar.gi.cast@gmail.com`

---

## Fase 4 — Push, sync y verificar

```bash
git add databases/evolution-postgres/ automation/evolution-api/
git commit -m "feat: add Evolution API v2.3.7 with CNPG database"
git push
```

ArgoCD sincroniza automáticamente. En Mac mini:

```bash
# ArgoCD detectó las apps
kubectl get applications -n argocd | grep evolution

# DB lista (~60s)
kubectl get cluster -n databases evolution-postgres -w

# Pod corriendo en hostNetwork (IP del host de Colima)
kubectl get pods -n automation -l app.kubernetes.io/name=evolution-api -o wide

# Logs sin errores
kubectl logs -n automation -l app.kubernetes.io/name=evolution-api --tail=50

# Ingress + ServiceMonitor
kubectl get ingress -n automation evolution-api-public
kubectl get servicemonitor -n automation evolution-api

# /metrics responde con datos Prometheus
kubectl exec -n automation deploy/evolution-api -- wget -q -O- http://localhost:8085/metrics | head -10
```

Checklist:
- [ ] `evolution-postgres` Application: Synced/Healthy
- [ ] `evolution-api` Application: Synced/Healthy
- [ ] CNPG cluster: `Cluster in healthy state`
- [ ] Pod evolution-api: Running 1/1 con IP del host (192.168.5.1)
- [ ] Ingress: host `evolution.oscargicast.com` con backend `evolution-api:8085`
- [ ] `/metrics` devuelve `evolution_environment_info{version="2.3.7"...}`
- [ ] Prometheus descubre el target: `kubectl exec -n observability prometheus-prometheus-kube-prometheus-prometheus-0 -- wget -q -O- http://localhost:9090/api/v1/targets | grep evolution`

Test desde browser:
- [ ] `https://evolution.oscargicast.com/` → CF Access (Google login) → `{"status":200,"message":"Welcome to the Evolution API...","version":"2.3.7"...}`

---

## Fase 5 — Conectar WhatsApp

### 5a — Acceder al manager

1. Abrir `https://evolution.oscargicast.com/manager` en el browser
2. Login con:
   - **Server URL**: `https://evolution.oscargicast.com`
   - **API Key Global**: el API key generado en Fase 2

### 5b — Crear instancia

Desde el manager → New Instance:
- **Name**: `personal` (o el que prefieras)
- **Number**: formato **sin `+`, sin espacios, sin guiones** — ej. `51907898162` (Perú)
- **Integration**: `WHATSAPP-BAILEYS`

Alternativa CLI:
```bash
API_KEY=$(kubectl -n automation get secret evolution-api-secrets -o jsonpath='{.data.api-key}' | base64 -d)

curl -X POST https://evolution.oscargicast.com/instance/create \
  -H "apikey: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"instanceName":"personal","integration":"WHATSAPP-BAILEYS","qrcode":true}'
```

### 5c — Escanear QR

1. En el manager, abrir la instancia recién creada → QR code aparece en el dashboard
2. En el teléfono: WhatsApp → Dispositivos vinculados → Vincular dispositivo
3. Escanear el QR

> Vas a ver un `stream errored out code 515` en los logs — **esto es NORMAL**. WhatsApp pide al cliente que se reconecte con las credenciales guardadas. Evolution API lo hace automáticamente.

### 5d — Verificar conexión + procesamiento

```bash
API_KEY=$(kubectl -n automation get secret evolution-api-secrets -o jsonpath='{.data.api-key}' | base64 -d)

# Estado de la instancia
kubectl exec -n automation deploy/evolution-api -- \
  wget -q -O- --header="apikey: ${API_KEY}" http://localhost:8085/instance/fetchInstances
```

Checklist (estos son los indicadores que prueban que TODO funciona):
- [ ] `connectionStatus: "open"`
- [ ] `_count.Message > 0`
- [ ] `_count.Contact > 0`
- [ ] `_count.Chat > 0`

> Si `connectionStatus = "open"` pero los counts están en 0 → ver Troubleshooting #1 (Redis).

---

## Fase 6 — Configurar webhook a n8n

### 6a — Workflow en n8n

En `https://n8n.oscargicast.com`:
1. Crear/abrir el workflow con un nodo **Webhook** como trigger
2. Copiar el **Production URL** (NO Test URL — las `webhook-test/*` solo responden cuando el editor está abierto)
3. **Activar** el workflow (toggle arriba a la derecha) — sin esto, el production URL devuelve 404

### 6b — Configurar webhook en Evolution API

Desde el manager → instancia → Webhook tab:
- **URL**: `http://n8n.automation.svc.cluster.local/webhook/<UUID-del-workflow>` (DNS interno del cluster, evita hairpinning por CF Tunnel)
- **Enabled**: ON
- **Events**: `MESSAGES_UPSERT` (recibir mensajes), `SEND_MESSAGE` (mensajes enviados), `CONNECTION_UPDATE` (estado de conexión)

Alternativa CLI:
```bash
API_KEY=$(kubectl -n automation get secret evolution-api-secrets -o jsonpath='{.data.api-key}' | base64 -d)

curl -X POST https://evolution.oscargicast.com/webhook/set/personal \
  -H "apikey: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://n8n.automation.svc.cluster.local/webhook/<UUID>",
    "webhook_by_events": false,
    "events": ["MESSAGES_UPSERT","SEND_MESSAGE","CONNECTION_UPDATE"]
  }'
```

### 6c — Test end-to-end

1. Mandate un mensaje WhatsApp a tu número desde otro contacto
2. Ver logs en tiempo real:
   ```bash
   kubectl logs -n automation -l app.kubernetes.io/name=evolution-api -f | grep -iE "webhook|message"
   ```
3. Debería aparecer `MESSAGES_UPSERT` + intento de webhook delivery
4. En n8n: el workflow debería tener una nueva ejecución exitosa

---

## Troubleshooting

### "Conexión open pero `Messages: 0` en métricas"

**Causa más probable**: Redis. Evolution API intenta conectarse a Redis (no desplegado) y rompe Baileys silenciosamente.

```bash
# Buscar errores de Redis (visibles solo con LOG_LEVEL incluyendo ERROR)
kubectl logs -n automation -l app.kubernetes.io/name=evolution-api --tail=200 | grep -i "redis"
```

Si aparecen `redis disconnected`:
- Verificar env vars en `deployment.yaml`:
  - `CACHE_REDIS_ENABLED: "false"`
  - `CACHE_LOCAL_ENABLED: "true"`
- Reiniciar pod: `kubectl rollout restart deployment/evolution-api -n automation`

### "Stuck in connecting post-QR"

**Causa**: falta `hostNetwork: true`. La red virtual de Colima no maneja los WebSockets largos de Baileys.

Verificar en `deployment.yaml`:
```yaml
spec:
  template:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
```

### "Port conflict / pod queda Pending"

```
FailedScheduling  node(s) didn't have free ports for the requested pod ports
```

Con `hostNetwork: true`, los puertos compiten en el host. Si tenés Traefik en port-forward 8080:80, ese puerto está ocupado. **Usar 8085** en `SERVER_PORT`, `containerPort`, Service `port/targetPort`, e Ingress `service.port.number`.

### "Pre-key upload timeout"

Bug conocido de Baileys 7.0.0-rc.9. **No es bloqueante** si Redis está deshabilitado correctamente. La conexión se reconecta y los mensajes fluyen igual.

### "/metrics retorna 404"

```bash
kubectl exec -n automation deploy/evolution-api -- wget -q -O- http://localhost:8085/metrics
# wget: server returned error: HTTP/1.1 404 Not Found
```

Faltan env vars:
- `PROMETHEUS_METRICS: "true"` (sin esto, el endpoint ni se registra)
- `METRICS_AUTH_REQUIRED: "false"` (si está en `true` y no hay `METRICS_USER`/`METRICS_PASSWORD`, devuelve 500)

### "Webhook no llega a n8n"

Diagnóstico paso a paso:
1. **El workflow está activo en n8n?** Toggle arriba a la derecha del workflow.
2. **Estás usando production URL?** No `/webhook-test/<UUID>` (ese es ephemeral, solo funciona con editor abierto).
3. **El webhook está configurado en la instancia?**
   ```bash
   API_KEY=$(kubectl -n automation get secret evolution-api-secrets -o jsonpath='{.data.api-key}' | base64 -d)
   kubectl exec -n automation deploy/evolution-api -- wget -q -O- --header="apikey: ${API_KEY}" http://localhost:8085/webhook/find/<instance-name>
   ```
4. **Evolution intenta delivery?** Logs con scope WEBHOOKS:
   ```bash
   kubectl logs -n automation -l app.kubernetes.io/name=evolution-api -f | grep -iE "webhook"
   ```
5. **n8n recibe pero el workflow falla?** Revisar ejecuciones en `https://n8n.oscargicast.com`.

Test directo de conectividad interna:
```bash
kubectl run curl-test --rm -it --image=curlimages/curl:8.7.1 --restart=Never -n automation -- \
  curl -v -X POST "http://n8n.automation.svc.cluster.local/webhook/<UUID>" \
  -H "Content-Type: application/json" \
  -d '{"test":"internal-dns"}'
```

### "Cambié versión y ahora la DB falla"

Las migraciones de Prisma no toleran cross-version downgrades. Si pasaste de v2.3.7 a v2.3.0 (o viceversa), wipear la DB:

```bash
DB_URI=$(kubectl -n automation get secret evolution-api-secrets -o jsonpath='{.data.database-url}' | base64 -d)
kubectl run -n databases psql-wipe --rm -it --restart=Never --image=postgres:16-alpine -- \
  psql "${DB_URI}" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

# Reiniciar pod para que corra migraciones frescas
kubectl rollout restart deployment/evolution-api -n automation
```

> **Esto borra la sesión WhatsApp** — vas a tener que re-escanear el QR.

### 404 desde CF Tunnel

Ver diagnóstico en `cloudflare-tunnel-migration.md` → "Lessons learned" → punto 4 (distinguir 404 de cloudflared vs Traefik).

---

## Lessons learned (del primer deploy)

1. **El error 515 post-QR es NORMAL**, no es un bug. WhatsApp pide restart al cliente. Evolution API reconecta automáticamente. No intentes "fixearlo".

2. **"Connection open" ≠ "todo funciona"**. Siempre verificar métricas (`Messages/Contacts/Chats > 0` o `evolution_instance_up: 1`). La conexión puede llegar a "open" mientras Baileys está roto internamente.

3. **`LOG_LEVEL: "WARN"` oculta los errores críticos de Redis** durante horas. Si las métricas están raras, primero subir el log level con scopes (`ERROR,WARN,INFO,LOG,VERBOSE,WEBHOOKS,WEBSOCKET`).

4. **`hostNetwork: true` no es opcional con Colima + Baileys**. Sin esto, el debugging es interminable porque la conexión llega a estados intermedios extraños.

5. **Redis aparenta ser "opcional"** según la doc oficial, pero sin `CACHE_REDIS_ENABLED: "false"` Evolution intenta conectarse y rompe todo. Es la configuración menos documentada y la más impactante.

6. **No bajes versiones sin wipe de DB**. Las migraciones de Prisma son one-way.

7. **Probá el endpoint `/metrics`** antes de buscar problemas en otros lados — es el sensor más confiable para saber si Evolution está procesando data.
