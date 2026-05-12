# Runbook: Speedtest Tracker — ISP throughput monitor

Despliega [speedtest-tracker](https://github.com/alexjustesen/speedtest-tracker) (imagen oficial `lscr.io/linuxserver/speedtest-tracker`) para correr tests de velocidad contra Ookla cada hora, guardar histórico en SQLite y mostrar el último resultado en la card "Speedtest" del grupo **Red** del dashboard Homepage. Solo accesible vía Tailscale en `https://speedtest.oscargicast.com` (no público).

## Arquitectura

| Componente | Namespace | Detalle |
|---|---|---|
| `speedtest-tracker` (Deployment) | `homelab` | Manifest-based, imagen LinuxServer, 1 replica, schedule cada hora |
| `speedtest-tracker-data` (PVC) | `homelab` | 2Gi `local-path` — SQLite + Laravel storage |
| `speedtest-tracker` (Service) | `homelab` | ClusterIP port 80 (UI + API + `/api/v1/prometheus`) |
| `speedtest-tracker-internal` (Ingress) | `homelab` | Traefik → host `speedtest.oscargicast.com` (CNAME DNS-only a Tailscale) |
| `speedtest-tracker` (ServiceMonitor) | `homelab` | Prometheus scrapea `/api/v1/prometheus` con Bearer token cada 60s |
| `speedtest-tracker-secrets` (SealedSecret) | `homelab` | `APP_KEY` (Laravel) + `PROMETHEUS_API_KEY` |
| `homepage-widget-secrets` (SealedSecret) | `homelab` | `HOMEPAGE_VAR_SPEEDTEST_KEY` (PAT generado en la UI) |

```
Browser (tailnet) → speedtest.oscargicast.com → CF DNS (gris) →
  Tailscale (oscar-mini-m1) → Colima → Traefik → speedtest-tracker:80

Speedtest scheduler → speedtest.net servers (egress)

Prometheus → speedtest-tracker.homelab.svc.cluster.local/api/v1/prometheus
  Authorization: Bearer <PROMETHEUS_API_KEY>

Homepage → speedtest-tracker.homelab.svc.cluster.local/api/v1/results/latest
  Authorization: Bearer <HOMEPAGE_VAR_SPEEDTEST_KEY>  (PAT, NO el mismo que arriba)
```

**Decisiones clave (no obvias):**
- **Solo Tailscale** — la UI no requiere internet abierto; minimiza superficie.
- **SQLite (no CNPG)** — la carga (1 test/hora) y los datos (rows pequeñas) no justifican un Postgres dedicado en un cluster de 10 GB RAM.
- **Tres tokens distintos** — `APP_KEY` (cifrado interno Laravel, lo genera deploy), `PROMETHEUS_API_KEY` (Bearer para `/api/v1/prometheus`, lo seteás vos), y el **PAT** (generado FROM the UI, único forma de autenticar el widget `speedtest` v2 de Homepage).
- **Schedule horario** (`0 * * * *`) — balance entre cobertura y ruido sobre el ancho de banda doméstico. Ajustable en `homelab/speedtest-tracker/deployment.yaml`.

---

## Fase 1 — Sellar `speedtest-tracker-secrets` (Mac mini)

Antes del primer sync de ArgoCD, generar APP_KEY y PROMETHEUS_API_KEY y sellarlos.

```bash
ssh macmini

APP_KEY="base64:$(openssl rand -base64 32)"
PROM_KEY="$(openssl rand -hex 32)"

cat > /tmp/speedtest-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: speedtest-tracker-secrets
  namespace: homelab
type: Opaque
stringData:
  APP_KEY: "$APP_KEY"
  PROMETHEUS_API_KEY: "$PROM_KEY"
EOF

kubeseal \
  --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem \
  --format yaml \
  < /tmp/speedtest-secret.yaml \
  > ~/Projects/homelab-k8s/homelab/speedtest-tracker/sealed-secret.yaml

rm /tmp/speedtest-secret.yaml
```

Guardá `$PROM_KEY` en tu password manager — lo vas a necesitar si querés probar el endpoint Prometheus manualmente con `curl`.

---

## Fase 2 — Crear el CNAME en Cloudflare (manual, fuera de Git)

En el dashboard de Cloudflare (zona `oscargicast.com`):

| Campo | Valor |
|---|---|
| Type | `CNAME` |
| Name | `speedtest` |
| Target | `oscar-mini-m1.tail90f0a7.ts.net` |
| Proxy status | **DNS only** (⚫ gris, NO naranja) |
| TTL | Auto |

Mismo patrón que `prometheus.oscargicast.com` y `grafana-internal.oscargicast.com`. **Crítico**: si lo dejás Proxied, CF intenta resolver la IP Tailscale (`100.x.x.x`) que es privada → infinite-spin.

Verificación:

```bash
dig speedtest.oscargicast.com @1.1.1.1 +short
# Debe retornar: oscar-mini-m1.tail90f0a7.ts.net. seguido de la IP 100.x.x.x
```

---

## Fase 3 — Commit + push + esperar ArgoCD sync

Desde la MacBook:

```bash
cd ~/Projects/homelab-k8s
git add homelab/speedtest-tracker/ homelab/homepage/ CLAUDE.md docs/runbooks/speedtest-tracker-setup.md
git commit -m "feat(homelab): speedtest-tracker + widget en Homepage"
git push
```

En la Mac mini, verificar:

```bash
ssh macmini

kubectl get applications -n argocd | grep speedtest
# speedtest-tracker   Synced   Healthy

kubectl -n homelab get pods,pvc -l app.kubernetes.io/name=speedtest-tracker
# pod/speedtest-tracker-<hash>   1/1   Running
# pvc/speedtest-tracker-data     Bound   2Gi   local-path

kubectl -n homelab logs deploy/speedtest-tracker --tail=50
# debe mostrar Laravel arrancando + scheduler activo
```

---

## Fase 4 — Bootstrap del admin user

speedtest-tracker v1.x viene con credenciales por default que **debés cambiar al primer login**.

1. Abrir `https://speedtest.oscargicast.com/` (con Tailscale activo en la MacBook).
2. Login inicial:
   - Email: `admin@example.com`
   - Password: `password`
3. **Inmediatamente** ir a Settings → Profile → cambiar email a `oscar.gi.cast@gmail.com` y setear un password fuerte. Guardalo en el password manager.
4. (Opcional) Settings → General → ajustar `Public Dashboard` a OFF.
5. (Opcional) Settings → Speedtest → confirmar `Test Schedule: 0 * * * *` y elegir un servidor preferido (o dejar auto).
6. Disparar un test manual desde el botón **Run Test** y verificar que aparece el resultado en el dashboard.

---

## Fase 5 — Generar el Personal Access Token (PAT) + sellar `homepage-widget-secrets`

El widget `speedtest` v2 de Homepage requiere un **Bearer token** que NO es el mismo que `PROMETHEUS_API_KEY` (ese solo abre `/api/v1/prometheus`). El PAT da acceso a `/api/v1/results/latest` y se genera FROM the UI.

1. En la UI: Settings → API Tokens → **Create New Token** → nombre `homepage-widget` → permisos `read` → **Generate** → copiar el token (solo se muestra una vez).
2. Sellar en `homepage-widget-secrets`:

```bash
ssh macmini

PAT="<pega-el-token-aquí>"

cat > /tmp/hpw-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: homepage-widget-secrets
  namespace: homelab
type: Opaque
stringData:
  HOMEPAGE_VAR_SPEEDTEST_KEY: "$PAT"
EOF

kubeseal \
  --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem \
  --format yaml \
  < /tmp/hpw-secret.yaml \
  > ~/Projects/homelab-k8s/homelab/homepage/sealed-secret-widget-secrets.yaml

rm /tmp/hpw-secret.yaml
```

3. Commit + push:

```bash
git add homelab/homepage/sealed-secret-widget-secrets.yaml
git commit -m "feat(homepage): sealed secret con PAT para widget speedtest"
git push
```

4. ArgoCD desencripta el SealedSecret → reinicia el Deployment de Homepage (porque cambió `envFrom`) → la card "Speedtest" del grupo **Red** muestra Download / Upload / Ping.

Si la card muestra `API Error` o `common.undefined`, ver Troubleshooting abajo.

---

## Verificación end-to-end

```bash
ssh macmini

# 1. Speedtest-tracker arriba
kubectl -n homelab get pods -l app.kubernetes.io/name=speedtest-tracker
kubectl -n homelab logs deploy/speedtest-tracker --tail=30

# 2. UI alcanzable (desde la mac mini, sin pasar por CF)
curl -sk -o /dev/null -w "%{http_code}\n" http://speedtest-tracker.homelab.svc.cluster.local/
# 200

# 3. Endpoint API con PAT (necesitás el token del paso 5)
PAT="<tu-pat>"
curl -s -H "Authorization: Bearer $PAT" \
  http://speedtest-tracker.homelab.svc.cluster.local/api/v1/results/latest | jq

# 4. Endpoint Prometheus con su key (la sellada en speedtest-tracker-secrets)
PROM_KEY=$(kubectl -n homelab get secret speedtest-tracker-secrets -o jsonpath='{.data.PROMETHEUS_API_KEY}' | base64 -d)
curl -s -H "Authorization: Bearer $PROM_KEY" \
  http://speedtest-tracker.homelab.svc.cluster.local/api/v1/prometheus | head

# 5. Prometheus target UP
kubectl -n observability port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 &
open "http://localhost:9090/targets"
# Buscar serviceMonitor/homelab/speedtest-tracker → debe estar UP
```

Desde la MacBook (Tailscale on), abrir `https://homepage.oscargicast.com` → grupo **Red** → card "Speedtest" debe mostrar valores reales.

---

## Troubleshooting

| Síntoma | Causa probable | Fix |
|---|---|---|
| Card "Speedtest" muestra `API Error` | PAT no llegó al pod Homepage (envFrom no aplicó) | `kubectl -n homelab rollout restart deploy/homepage` |
| Card muestra `common.undefined` en todos los campos | El widget no encuentra resultados (no se corrió ningún test todavía) | Disparar **Run Test** desde la UI manualmente |
| Pod en `CrashLoopBackOff` con `No application encryption key has been specified` | El SealedSecret de `APP_KEY` falla o no llegó | Verificar `kubectl -n homelab get secret speedtest-tracker-secrets`. Re-sellar con Fase 1. |
| Pod corre pero `/api/v1/prometheus` retorna 401 | Bearer token incorrecto en ServiceMonitor | Verificar key `PROMETHEUS_API_KEY` en el secret coincide con la que speedtest-tracker tiene en `/config/.env` |
| `dig speedtest.oscargicast.com` retorna IPs de Cloudflare (no Tailscale) | El record está Proxied (naranja) en CF | Cambiar a DNS only (gris) |
| Test schedule no corre | Container caído justo cuando tocaba el cron | El cron es interno a Laravel — se ejecuta en el siguiente intervalo. Verificar con `kubectl -n homelab logs deploy/speedtest-tracker | grep -i schedule` |

---

## Rotar el PAT de Homepage

Si comprometés el token (o querés rotación periódica):

1. UI → Settings → API Tokens → **Revoke** el viejo.
2. Crear uno nuevo (mismos permisos).
3. Re-sellar `homepage-widget-secrets` con el nuevo valor (Fase 5).
4. Commit + push.

Ese flujo no requiere reiniciar speedtest-tracker; solo el pod de Homepage (lo hace ArgoCD automáticamente al cambiar el secret).
