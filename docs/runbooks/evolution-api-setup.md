# Runbook: desplegar Evolution API v2.3.7 + conectar WhatsApp con n8n

Despliega [Evolution API](https://github.com/EvolutionAPI/evolution-api) en el cluster para conectar una cuenta personal de WhatsApp con n8n. El servicio queda expuesto como `https://evolution.oscargicast.com` detrás de Cloudflare Access (Google auth).

## Resumen de cambios

| Componente | Namespace | Acceso |
|---|---|---|
| `evolution-postgres` (CNPG) | `databases` | Interno (cluster DNS) |
| `evolution-api` (Deployment) | `automation` | `https://evolution.oscargicast.com` + CF Access |

Imagen Docker: `evoapicloud/evolution-api:v2.3.7`

## Arquitectura

```
WhatsApp servers ←(outbound)← Evolution API pod
                                    ↕ (internal DNS)
                                  n8n pod
                                    ↕
                              evolution-postgres (CNPG)

Browser → CF Edge → CF Access (Google) → CF Tunnel → Traefik → evolution-api:8080
```

- Evolution API se conecta **saliente** a los servidores de WhatsApp (protocolo Baileys/WhatsApp Web)
- n8n se comunica con Evolution API por DNS interno: `http://evolution-api.automation.svc.cluster.local:8080`
- Los datos (instancias, mensajes, contactos) se persisten en PostgreSQL vía CNPG
- No requiere Redis para single-instance

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

## Fase 1 — Crear manifiestos (en MacBook)

Los siguientes archivos ya deberían estar creados en el repo (si seguiste el plan de implementación). Verificar que existen:

- [ ] `databases/evolution-postgres/application.yaml`
- [ ] `databases/evolution-postgres/cluster.yaml`
- [ ] `automation/evolution-api/application.yaml`
- [ ] `automation/evolution-api/deployment.yaml`
- [ ] `automation/evolution-api/service.yaml`
- [ ] `automation/evolution-api/ingress-public.yaml`

Los parent Applications en `clusters/mac-mini/apps.yaml` auto-descubren `**/application.yaml`, así que **no hace falta modificar ningún archivo existente** para registrar las nuevas apps.

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

# 5. Guardar el API key (lo necesitás en Fase 5 para n8n)
echo "=== GUARDAR ESTE API KEY ==="
echo "API Key: ${API_KEY}"
echo "============================"

# 6. Limpiar plaintext
rm /tmp/evolution-db-user.yaml /tmp/evolution-api-secrets.yaml
```

- [ ] Verificar que `databases/evolution-postgres/sealed-secret.yaml` existe y tiene `kind: SealedSecret`
- [ ] Verificar que `automation/evolution-api/sealed-secret.yaml` existe y tiene `kind: SealedSecret`
- [ ] API key guardado en lugar seguro

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

> Si devuelve NXDOMAIN: crear CNAME manualmente en DNS → Add record → Type `CNAME`, Name `evolution`, Target `<TUNNEL_UUID>.cfargotunnel.com`, Proxy **ON** (orange-cloud).

### 3c — Cloudflare Access

Zero Trust → Access → Applications → Add → Self-hosted:

- [ ] Name: `Evolution API`
- [ ] Application domain: `evolution.oscargicast.com`
- [ ] Policy: Allow → Emails → `oscar.gi.cast@gmail.com`

---

## Fase 4 — Push y verificar

```bash
git add databases/evolution-postgres/ automation/evolution-api/ homelab/homepage/values.yaml
git commit -m "feat: add Evolution API v2.3.7 with CNPG database"
git push
```

ArgoCD sincroniza automáticamente. En Mac mini:

```bash
# ArgoCD detectó las apps
kubectl get applications -n argocd | grep evolution

# DB lista (~60s)
kubectl get cluster -n databases evolution-postgres -w

# Pod corriendo
kubectl get pods -n automation -l app.kubernetes.io/name=evolution-api

# Logs sin errores
kubectl logs -n automation -l app.kubernetes.io/name=evolution-api --tail=50

# Ingress creado
kubectl get ingress -n automation evolution-api-public
```

- [ ] `evolution-postgres` Application: Synced/Healthy
- [ ] `evolution-api` Application: Synced/Healthy
- [ ] CNPG cluster: `Cluster in healthy state`
- [ ] Pod evolution-api: Running 1/1
- [ ] Ingress: host `evolution.oscargicast.com`

Test desde browser:

- [ ] `https://evolution.oscargicast.com/` → CF Access (Google login) → respuesta de Evolution API

---

## Fase 5 — Conectar WhatsApp

### 5a — Crear instancia

```bash
curl -X POST https://evolution.oscargicast.com/instance/create \
  -H "apikey: <TU_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"instanceName":"whatsapp-personal","integration":"WHATSAPP-BAILEYS","qrcode":true}'
```

La respuesta incluye un campo `qrcode.base64` con la imagen del QR.

### 5b — Escanear QR

1. Abrir `https://evolution.oscargicast.com/manager` en el browser
2. Abrir WhatsApp en el teléfono → Dispositivos vinculados → Vincular dispositivo
3. Escanear el QR code

### 5c — Verificar conexión

```bash
curl -s https://evolution.oscargicast.com/instance/connectionState/whatsapp-personal \
  -H "apikey: <TU_API_KEY>"
```

- [ ] Respuesta: `{"state": "open"}`

---

## Fase 6 — Configurar n8n

### 6a — Instalar community node

En `https://n8n.oscargicast.com`:

1. Settings → Community Nodes → Install
2. Nombre del paquete: `n8n-nodes-evolution-api`
3. Instalar

### 6b — Crear credentials

1. Credentials → Add Credential → buscar "Evolution API"
2. Configurar:
   - **Server URL**: `http://evolution-api.automation.svc.cluster.local:8080` (DNS interno, NO la URL pública)
   - **API Key**: el key generado en Fase 2

### 6c — Test rápido

Crear un workflow simple:
1. Trigger manual
2. Nodo Evolution API → Send Text Message
3. Configurar: instancia `whatsapp-personal`, número destino, mensaje de prueba
4. Ejecutar

- [ ] Mensaje llega a WhatsApp

---

## Troubleshooting

### Pod en CrashLoopBackOff

```bash
kubectl logs -n automation -l app.kubernetes.io/name=evolution-api --tail=100
```

**Si el error es de conexión a DB**: verificar que `evolution-postgres` está healthy:
```bash
kubectl get cluster -n databases evolution-postgres
kubectl get pods -n databases -l cnpg.io/cluster=evolution-postgres
```

El pod de Evolution API puede crash-loopear si la DB aún no está lista. Kubernetes lo reintenta con backoff, y eventualmente conecta (~1-2 min).

### Health check falla

Si los probes usan path `/` y Evolution API responde en otro path:
```bash
kubectl exec -n automation deploy/evolution-api -- wget -q -O- http://localhost:8080/ 2>&1 | head -5
```

Si devuelve 404, probar `/api/health` o `/ping` y actualizar los probes en `deployment.yaml`.

### Sesión WhatsApp desconectada

WhatsApp Web puede desconectarse si:
- El teléfono estuvo offline mucho tiempo
- WhatsApp actualizó el protocolo
- Se vincularon demasiados dispositivos

Solución: re-escanear QR desde `https://evolution.oscargicast.com/manager`.

> La sesión se persiste en PostgreSQL (`DEL_INSTANCE=false`), así que reinicios del pod NO requieren re-escaneo.

### 404 desde CF Tunnel

Ver diagnóstico en `docs/runbooks/cloudflare-tunnel-migration.md` → "Lessons learned" → punto 4 (distinguir 404 de cloudflared vs Traefik).

### Homepage widgets muestran "API Error" o "common.undefined"

- Verificar que `evolution-postgres` tiene `monitoring.enablePodMonitor: true` en `cluster.yaml`
- Verificar que Prometheus descubre el PodMonitor:
  ```bash
  kubectl get podmonitor -n databases | grep evolution
  ```
- Si no aparece, revisar `podMonitorSelector` en prometheus values (debe ser `{}`)
