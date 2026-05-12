# Planning — TLS interno para servicios Tailscale-only

> **Status: NOT IMPLEMENTED** — diferido por bajo ROI (ver "Decisión y disparadores").
> Última revisión: 2026-05-12

## Problema

Los servicios internos del homelab (`speedtest.oscargicast.com`, `prometheus.oscargicast.com`) son CNAMEs DNS-only en Cloudflare apuntando a `oscar-mini-m1.tail90f0a7.ts.net`. Solo accesibles desde la tailnet. Al intentar acceder por `https://...` el navegador muestra "not secure" porque:

- Traefik solo tiene entrypoint `web` (:80), no `websecure` (:443) — ver `infrastructure/traefik/application.yaml:36-42`
- Los Ingresses no declaran `tls:` ni anotaciones de cert-manager
- No hay cert-manager instalado
- El único token CF en el repo es el del Cloudflare Tunnel, no un API Token Zone

El acceso por `http://...:8080` funciona y se usa hoy. El warning solo aparece si el user fuerza `https://`.

## Por qué se difiere

Tailscale ya cifra E2E con WireGuard. Agregar TLS sobre ese tramo es defensa en profundidad redundante. El único beneficio práctico es UX (no warning). Costos: cert-manager (3 pods ≈ 200Mi RAM), 1 sealed-secret extra, rotación de CF API Token, 1 `kubectl port-forward` adicional, ~1h de setup + debugging.

Para un homelab single-user con 2 servicios internos accedidos por tailnet, el ROI no justifica el costo hoy.

## Disparadores que harían que sí valga la pena

Implementar cuando se cumpla **al menos uno**:

- Añadir un servicio interno donde HTTP rompe la app (PWA, OAuth, panel moderno con strict CSP / secure cookies).
- Crecer a **≥5 servicios internos** (costo fijo de cert-manager se amortiza).
- Querer setear HSTS / cookies `Secure` en algún servicio interno.
- Acceso diario donde el warning del browser fricciona productividad.

## Solución diseñada (cuando se ejecute)

### Approach

cert-manager + ClusterIssuer Let's Encrypt con solver **DNS-01 Cloudflare**. HTTP-01 no funciona porque LE no puede alcanzar IPs de tailnet (`100.x.x.x`) desde internet.

### Flujo de tráfico objetivo

```
Browser (tailnet) → oscar-mini-m1.tail90f0a7.ts.net:8443
                  → kubectl port-forward
                  → traefik.traefik.svc:443 (TLS termina aquí, cert de cert-manager)
                  → Ingress (Host header routing)
                  → speedtest-tracker / prometheus
```

URLs finales: `https://speedtest.oscargicast.com:8443/` y `https://prometheus.oscargicast.com:8443/`. El puerto `:8443` es por la arquitectura del port-forward del homelab (Traefik es ClusterIP, expuesto por `kubectl port-forward` desde Mac mini). Los `http://...:8080` actuales siguen funcionando intactos.

### Archivos a tocar

| Path | Acción |
|---|---|
| `infrastructure/cert-manager/application.yaml` | CREATE — Helm chart `jetstack/cert-manager` v1.16.x con `crds.enabled: true`, sync-wave `-1`, `ServerSideApply=true` |
| `infrastructure/cert-manager/sealed-secret-cf-api-token.yaml` | CREATE — vía `kubeseal` |
| `infrastructure/cert-manager/clusterissuer-letsencrypt.yaml` | CREATE — ClusterIssuers `letsencrypt-prod` + `letsencrypt-staging` (escape hatch contra rate limit) |
| `infrastructure/traefik/application.yaml` | UPDATE — añadir bloque `ports.websecure` con `port: 8443, exposedPort: 443, tls.enabled: true` (no tocar `service.type: ClusterIP`) |
| `homelab/speedtest-tracker/ingress-internal.yaml` | UPDATE — annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` + bloque `tls:` |
| `observability/prometheus/prometheus-ingress.yaml` | UPDATE — mismo patrón |
| `README.md` línea ~186-190 | UPDATE — documentar segundo port-forward `:8443:443` |
| `CLAUDE.md` | UPDATE — tabla DNS y gotcha de cert-manager (rate limit LE, propagación TXT, `dnsZones` selector) |

### Manifiestos de referencia (snippets listos para copiar)

**ClusterIssuer:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: oscar.gi.cast@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones: ["oscargicast.com"]
```

**Traefik `ports` (reemplaza líneas 36-42 de `infrastructure/traefik/application.yaml`):**

```yaml
ports:
  web: {}
  websecure:
    port: 8443
    exposedPort: 443
    protocol: TCP
    tls:
      enabled: true
  metrics:
    port: 9101
    exposedPort: 9101
    expose:
      default: true
```

**Ingress con TLS (patrón para los dos):**

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
    - hosts: [speedtest.oscargicast.com]
      secretName: speedtest-tracker-tls
  rules:
    - host: speedtest.oscargicast.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: speedtest-tracker
                port: { number: 80 }
```

### Pasos manuales el día que se ejecute

#### Phase 1 — Crear el Cloudflare API Token (browser)

1. https://dash.cloudflare.com/profile/api-tokens → **Create Token → Custom Token**
2. Nombre: `cert-manager-dns01-homelab`
3. Permissions: `Zone → DNS → Edit` + `Zone → Zone → Read`
4. Zone Resources: `Include → Specific zone → oscargicast.com`
5. **Continue to summary → Create Token** y guardarlo (se muestra una sola vez).

#### Phase 2 — Sellar el secret (MacBook)

```bash
cd ~/Projects/homelab-k8s

cat > /tmp/cf-api-token.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
type: Opaque
stringData:
  api-token: <PEGAR_TOKEN>
EOF

kubeseal --cert ~/.config/homelab-k8s-sealed-secrets-pub.pem \
  --format yaml \
  < /tmp/cf-api-token.yaml \
  > infrastructure/cert-manager/sealed-secret-cf-api-token.yaml

rm /tmp/cf-api-token.yaml
```

#### Phase 3 — Commits en orden (rollback-friendly)

1. Commit A: solo `infrastructure/cert-manager/application.yaml`. Esperar Healthy en ArgoCD.
2. Commit B: SealedSecret + ClusterIssuers. Esperar `kubectl get clusterissuer letsencrypt-prod` → Ready=True.
3. Commit C: Traefik values con `websecure`. Esperar rollout y verificar `kubectl get svc -n traefik traefik` muestra `443/TCP`.
4. Commit D: TLS en los dos Ingresses. Observar Certificates con `kubectl get certificate -A -w`.
5. En Mac mini: lanzar segundo port-forward `kubectl port-forward -n traefik svc/traefik --address 0.0.0.0 8443:443 &`.

#### Phase 4 — Verificación

```bash
# ClusterIssuer Ready
kubectl get clusterissuer letsencrypt-prod

# Certificates
kubectl get certificate -A
kubectl get secret -n homelab speedtest-tracker-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer -dates -subject

# End-to-end
curl -v https://speedtest.oscargicast.com:8443/ 2>&1 | grep -E "SSL certificate verify"
curl -v https://prometheus.oscargicast.com:8443/ 2>&1 | grep -E "SSL certificate verify"
```

### Renovación

cert-manager renueva automáticamente cuando faltan ~30 días (`2/3` de los 90 días de LE). No es un cron de Kubernetes — es un loop de reconciliación del propio controller. Traefik recarga el cert desde el Secret sin reinicio de pod.

Forzar renovación de prueba:

```bash
kubectl delete secret speedtest-tracker-tls -n homelab   # brutal pero efectivo
# o, con plugin:
kubectl cert-manager renew speedtest-tracker-tls -n homelab
```

### Rotación del CF API Token

Repetir Phase 1 + Phase 2. ArgoCD actualiza el Secret. cert-manager re-lee al siguiente renew. Borrar el token viejo en CF dashboard.

### Troubleshooting (referencia)

| Síntoma | Causa | Acción |
|---|---|---|
| ClusterIssuer Ready=False, "secret not found" | SealedSecret no sincronizó | `kubectl get sealedsecret -n cert-manager`; logs del controller |
| "Invalid API Token" | Permisos mal o token mal copiado | Recrear con `Zone:DNS:Edit + Zone:Zone:Read` |
| Certificate atascado | TXT no propaga | `kubectl describe challenge -n <ns>`; `dig +short TXT _acme-challenge.<host> @1.1.1.1` |
| Rate limit LE prod | Demasiados intentos fallidos | Cambiar Ingress annotation a `letsencrypt-staging`, validar, volver a prod, borrar Secret viejo |

### Alternativas consideradas y descartadas

| Alternativa | Por qué no |
|---|---|
| Tailscale Serve con cert `.ts.net` | Pierde el dominio `oscargicast.com` |
| CF Origin CA cert | Solo válido si CF proxia; estos hosts son DNS-only |
| Cert wildcard manual con certbot | Renovación manual cada 90 días, frágil |
| Mover a CF Tunnel + Access | Cambio arquitectónico mayor, pierde el modo Tailscale-only |
| HTTP-01 challenge | LE no alcanza IPs de tailnet |

## Decisión y disparadores (resumen)

**Hoy: NO IMPLEMENTAR.** Acceso por `http://...:8080` cubre el uso actual.

Revisar este documento cuando se cumpla un disparador (sección "Disparadores que harían que sí valga la pena"). Cuando se ejecute, mover este archivo a `docs/runbooks/cert-manager-internal-tls.md` y actualizar el status del header.
