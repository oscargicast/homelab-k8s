# Runbook: migrar Bitnami Sealed Secrets → Infisical (self-hosted)

Reemplaza el flujo `kubeseal → commit cifrado → rm /tmp` por una UI web (`https://infisical.oscargicast.com`) protegida con Cloudflare Access. Los valores viven en el Postgres de Infisical (dentro del cluster); los workloads consumen `Secret` nativos materializados por el **Infisical Secrets Operator** a partir de `InfisicalSecret` CRs versionados en Git (sin valores, solo punteros).

> Todos los `kubectl` se ejecutan en la **Mac mini** (`ssh macmini`). Las ediciones de YAML se hacen desde la **MacBook**.

## Resumen de cambios

| Elemento | Antes (Bitnami) | Después (Infisical) |
|---|---|---|
| Storage de secretos | YAML cifrado en Git (`sealed-secret.yaml`) | Postgres de Infisical (dentro del cluster) |
| Rotación de valor | `kubeseal` + commit + push + sync | Editar en UI → operator re-sincroniza en ≤60s |
| Inspección | `kubectl get secret … -o yaml \| base64 -d` | UI web + RBAC granular |
| Material en repo | `encryptedData: AgAi...` | `InfisicalSecret` CR con `secretsPath: /…` |
| Bootstrap | sealed-secrets-controller + cert público | `infisical-bootstrap` Secret + `infisical-redis-auth` Secret (manuales) |

## Arquitectura

```
Cloudflare Edge → CF Access → CF Tunnel
        ↓
   Traefik Ingress (host: infisical.oscargicast.com)
        ↓
   Service infisical:8080 (ns: infisical)
        ↓
   Infisical backend → CNPG infisical-postgres (ns: databases)
                    └─→ Redis dedicado infisical-redis (ns: infisical)

   Infisical Secrets Operator (ns: infisical-operator)
        ↓ k8sAuth Machine Identity → http://infisical.infisical.svc.cluster.local:8080
        ↓
   Lee InfisicalSecret CR de cualquier namespace → escribe Secret nativo
```

---

## Fase 0 — Pre-checks (en Mac mini)

```bash
ssh macmini
```

- [ ] `kubectl get nodes` y `kubectl get pods -A` sin errores.
- [ ] Argo CD healthy: `kubectl -n argocd get pods` (todo Running).
- [ ] Port-forwards activos:
  ```bash
  kubectl port-forward -n traefik svc/traefik --address 0.0.0.0 8080:80
  ```
- [ ] Acceso al dashboard de Cloudflare (zona `oscargicast.com`).
- [ ] Repo en branch limpia con los YAMLs de Infisical commiteados (ver `git log` del commit "feat(infisical): bootstrap stack + runbook").

---

## Fase A — Deploy del backend de Infisical

### A1. Bootstrap manual (Secrets que NO viven en Git)

Genera todas las credenciales en una sola sesión de shell y aplícalas; los valores nunca tocan disco. Opcionalmente respaldarlos en password manager antes de cerrar la shell.

```bash
# 1. Namespaces (los charts Argo no los crean por CreateNamespace=false)
kubectl create namespace infisical
kubectl create namespace infisical-operator

# 2. Generar credenciales
PG_PASSWORD=$(openssl rand -base64 24)
ENCRYPTION_KEY=$(openssl rand -hex 16)        # exactamente 32 chars hex
AUTH_SECRET=$(openssl rand -base64 32)
REDIS_PASSWORD=$(openssl rand -base64 24)

# 3. Owner del CNPG cluster (formato basic-auth)
kubectl -n databases create secret generic infisical-db-user \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=infisical \
  --from-literal=password="$PG_PASSWORD"

# 4. Secret raíz del backend (kubeSecretRef del chart)
kubectl -n infisical create secret generic infisical-bootstrap \
  --from-literal=DB_CONNECTION_URI="postgresql://infisical:$PG_PASSWORD@infisical-postgres-rw.databases.svc.cluster.local:5432/infisical" \
  --from-literal=ENCRYPTION_KEY="$ENCRYPTION_KEY" \
  --from-literal=AUTH_SECRET="$AUTH_SECRET"

# 5. Secret de Redis (consumido por pod Redis y por backend Infisical)
kubectl -n infisical create secret generic infisical-redis-auth \
  --from-literal=REDIS_PASSWORD="$REDIS_PASSWORD" \
  --from-literal=REDIS_URL="redis://default:$REDIS_PASSWORD@infisical-redis:6379"

# 6. (Opcional) Imprimir para respaldar en password manager
echo "PG_PASSWORD=$PG_PASSWORD"
echo "ENCRYPTION_KEY=$ENCRYPTION_KEY"
echo "AUTH_SECRET=$AUTH_SECRET"
echo "REDIS_PASSWORD=$REDIS_PASSWORD"

# 7. Limpiar
unset PG_PASSWORD ENCRYPTION_KEY AUTH_SECRET REDIS_PASSWORD
```

Verificación:

```bash
kubectl -n databases get secret infisical-db-user
kubectl -n infisical get secret infisical-bootstrap infisical-redis-auth
```

### A2. Push de los YAMLs (los crea Argo CD)

Desde la **MacBook**:

```bash
git push origin main
```

Espera la sincronización en Argo (`https://argocd.oscargicast.com`) — orden de sync-waves:

1. `sync-wave: -2` → `infisical-operator` (instala CRDs).
2. `sync-wave: -1` → `infisical` (backend), `infisical-redis`.
3. `sync-wave: 0` → `infisical-postgres` (CNPG cluster).

Si el backend entra en `CrashLoopBackOff`, revisa logs (`kubectl -n infisical logs deploy/infisical --tail=100`) — la causa típica es Redis no listo todavía o el CNPG aún provisionando.

### A3. Configurar CF Tunnel + Access

En el dashboard de **Cloudflare**:

1. **Networks → Tunnels → homelab-k8s → Public Hostnames → Add a public hostname**:
   - Subdomain: `infisical`
   - Domain: `oscargicast.com`
   - Path: vacío
   - Service Type: `HTTP`
   - URL: `traefik.traefik.svc.cluster.local:80`
2. Verificar que el CNAME `infisical.oscargicast.com` se haya auto-creado (bug conocido — a veces falla):
   ```bash
   dig infisical.oscargicast.com @1.1.1.1 +short
   # Debe devolver IPs de CF (104.21.x.x / 172.67.x.x)
   ```
   Si retorna `NXDOMAIN`, crear el CNAME manualmente: `infisical` → `<TUNNEL_UUID>.cfargotunnel.com`, proxied.
3. **Zero Trust → Access → Applications → Add an application → Self-hosted**:
   - Name: `Infisical`
   - Session duration: 24h
   - Application domain: `infisical.oscargicast.com`
   - Policy: `Allow` → emails `oscar.gi.cast@gmail.com`.

### A4. Verificación

```bash
# Pods Running
kubectl -n infisical get pods
kubectl -n infisical-operator get pods
kubectl -n databases get pods | grep infisical-postgres

# Health interno
kubectl -n infisical port-forward svc/infisical 8080:8080 &
curl -s localhost:8080/api/status | jq

# CRD del operator instalado
kubectl get crd infisicalsecrets.secrets.infisical.com
kubectl get crd infisicalpushsecrets.secrets.infisical.com 2>/dev/null || true

# UI pública
curl -sI https://infisical.oscargicast.com | head -1   # 302 a CF Access
# En browser: navegar a https://infisical.oscargicast.com → email → signup admin
```

---

## Fase B — Configuración inicial desde la UI

Tras login admin en `https://infisical.oscargicast.com`:

- [ ] **B1.** Signup como primer admin (email `oscar.gi.cast@gmail.com`).
- [ ] **B2.** Crear **Organization** `homelab` → **Project** `homelab-k8s` → **Environment** `prod`.
- [ ] **B3.** Crear folders (replicar la estructura del repo):
  - `/automation/n8n`
  - `/automation/evolution-api`
  - `/databases/n8n-postgres`
  - `/databases/postgres-lab`
  - `/databases/evolution-postgres`
  - `/infrastructure/cloudflared`
- [ ] **B4.** Crear **Machine Identity** `k8s-operator`:
  - Auth Method: **Kubernetes Auth**.
  - Kubernetes Host: `https://kubernetes.default.svc` (default).
  - Token Reviewer JWT: dejar vacío (el operator usa TokenReview por defecto).
  - Allowed Service Account Names: `infisical-operator-controller-manager`.
  - Allowed Namespaces: `infisical-operator`.
  - Allowed Audiences: vacío.
  - Token TTL: default.
  - Rol del Identity en el project `homelab-k8s`: `Reader` (o más restrictivo si querés scoping per-folder).
- [ ] **B5.** **Copiar el `Identity ID`** (UUID que muestra la UI del identity). Guardar localmente — se usa en cada `InfisicalSecret` CR.

---

## Fase C — Migración de los 6 secretos (gradual)

Orden (menos crítico → más crítico):

| # | Secret | Path repo (Bitnami) | Folder Infisical | Notas |
|---|---|---|---|---|
| 1 | `n8n-db-credentials` (automation) | `automation/n8n/sealed-secret.yaml` | `/automation/n8n` | Piloto |
| 2 | `evolution-api-secrets` (automation) | `automation/evolution-api/sealed-secret.yaml` | `/automation/evolution-api` | Multi-key |
| 3 | `n8n-db-user` (databases) | `databases/n8n-postgres/sealed-secret.yaml` | `/databases/n8n-postgres` | CNPG bootstrap |
| 4 | `postgres-lab-app-user` (databases) | `databases/postgres-lab/sealed-secret.yaml` | `/databases/postgres-lab` | CNPG bootstrap |
| 5 | `evolution-db-user` (databases) | `databases/evolution-postgres/sealed-secret.yaml` | `/databases/evolution-postgres` | CNPG bootstrap |
| 6 | `cloudflared-token` (cloudflared) | `infrastructure/cloudflared/sealed-secret.yaml` | `/infrastructure/cloudflared` | **CRÍTICO** — si rompe, perdés acceso público |

### Patrón por secret

Para cada uno, repetir estos 6 pasos:

#### C1. Extraer el valor actual del SealedSecret (Mac mini)

```bash
# Ejemplo para n8n-db-credentials
kubectl -n automation get secret n8n-db-credentials -o jsonpath='{.data.password}' | base64 -d; echo
```

(Ver `docs/runbooks/secrets-cheatsheet.md` para los comandos por secret.)

#### C2. Subir el valor a Infisical UI

- Navegar al folder correspondiente.
- **Add Secret** — usar la misma clave que el SealedSecret (ej. `password`, `username`, `api-key`, `database-url`, `token`).
- Pegar el valor. Repetir por cada clave del secret.

#### C3. Crear `<app>/infisical-secret.yaml` (MacBook)

Plantilla (sustituir las 5 variables):

```yaml
apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret
metadata:
  name: <SECRET-NAME>
  namespace: <NAMESPACE>
spec:
  resyncInterval: 60
  authentication:
    kubernetesAuth:
      identityId: <UUID-MACHINE-IDENTITY>
      serviceAccountRef:
        name: infisical-operator-controller-manager
        namespace: infisical-operator
  projectSlug: homelab-k8s
  envSlug: prod
  secretsPath: <FOLDER-PATH>
  managedSecretReference:
    secretName: <SECRET-NAME>
    secretNamespace: <NAMESPACE>
    creationPolicy: Owner
```

> `hostAPI` se omite — el operator usa el default del Helm values (`http://infisical.infisical.svc.cluster.local:8080/api`).

Agregar también el archivo al `directory.include` del `application.yaml` del app correspondiente (mirar el commit "feat(infisical): migrate <secret>" para el patrón exacto).

#### C4. Commit + push

```bash
git add <app>/infisical-secret.yaml <app>/application.yaml
git commit -m "feat(infisical): migrate <SECRET-NAME>"
git push
```

#### C5. Verificar (Mac mini)

```bash
# CR creado y SYNCED
kubectl -n <ns> get infisicalsecret <secret-name>
# STATUS debe ser "Synced", LAST UPDATED reciente

# Secret materializado idéntico al Bitnami
kubectl -n <ns> get secret <secret-name> -o yaml | yq '.data'
# Comparar con el original (debería ser byte-idéntico)

# Sin errores en el operator
kubectl -n infisical-operator logs -l app.kubernetes.io/name=secrets-operator --tail=50 | grep -i error
```

#### C6. Borrar el SealedSecret de Bitnami

```bash
git rm <app>/sealed-secret.yaml
# Quitar la referencia a sealed-secret.yaml del directory.include del application.yaml
git commit -m "chore(infisical): drop bitnami sealed-secret for <SECRET-NAME>"
git push
```

Argo CD elimina el `SealedSecret` CR (Bitnami). El `Secret` nativo persiste porque ya es propiedad del `InfisicalSecret` CR (no del SealedSecret).

**Restart opcional** si el workload cachea credenciales al startup:

```bash
kubectl -n <ns> rollout restart deploy/<workload>
kubectl -n <ns> logs deploy/<workload> --tail=50    # sin errores de auth
```

---

## Cleanup final (post-#6)

Solo después de verificar que `cloudflared` siga funcionando con el secret migrado:

```bash
# MacBook
git rm -r infrastructure/sealed-secrets/
git commit -m "chore: remove bitnami sealed-secrets stack (migrated to infisical)"
git push

# Mac mini — borrar el cert público local
rm ~/.config/homelab-k8s-sealed-secrets-pub.pem
```

Actualizar **CLAUDE.md**:
- Reescribir la sección "Secrets — Sealed Secrets" → "Secrets — Infisical".
- Agregar gotchas de Infisical a la tabla "Chart-specific gotchas".
- Remover del stack table la fila de Sealed Secrets.

---

## Rollback

Si un secret migrado rompe su workload:

1. **Inmediato (sin tocar Git)**: re-crear manualmente el `Secret` desde el último valor conocido:
   ```bash
   kubectl -n <ns> create secret generic <name> --from-literal=<key>=<value> --dry-run=client -o yaml | kubectl apply -f -
   ```
   El operator de Infisical sobreescribirá esto en la próxima reconciliación (60s) — usar solo como mitigación temporal.
2. **Permanente (revertir commit)**:
   ```bash
   git revert <commit-hash-de-la-migracion>
   git push
   ```
   Argo restaura el SealedSecret de Bitnami y elimina el InfisicalSecret CR.

> Si el SealedSecret Application ya fue eliminado del cluster (post-cleanup), restaurarlo primero: `kubectl apply -f infrastructure/sealed-secrets/application.yaml` desde un commit anterior.

---

## Troubleshooting

| Síntoma | Causa probable | Fix |
|---|---|---|
| `infisical` pod en `CrashLoopBackOff` con `error connecting to database` | `DB_CONNECTION_URI` mal formado o CNPG aún no Ready | `kubectl -n databases get cluster infisical-postgres` — esperar a `Cluster in healthy state`. Verificar el Secret: `kubectl -n infisical get secret infisical-bootstrap -o jsonpath='{.data.DB_CONNECTION_URI}' \| base64 -d` |
| `infisical` pod up pero `/api/status` retorna 500 | Redis no reachable | `kubectl -n infisical get pods -l app.kubernetes.io/name=infisical-redis` — debe estar Running. Probar: `kubectl -n infisical exec deploy/infisical-redis -- redis-cli -a "$(kubectl -n infisical get secret infisical-redis-auth -o jsonpath='{.data.REDIS_PASSWORD}' \| base64 -d)" ping` |
| `InfisicalSecret` en estado `Failed` con error `403 Forbidden` | Machine Identity no autoriza al SA del operator | UI Infisical → Identity `k8s-operator` → Authentication → Kubernetes Auth → verificar Allowed SA = `infisical-operator-controller-manager`, namespace `infisical-operator` |
| `InfisicalSecret` `Failed` con `project not found` | `projectSlug` o `envSlug` mal | UI → Project Settings → copiar el slug exacto |
| `https://infisical.oscargicast.com` retorna 404 page not found (Go default body) | Catch-all del cloudflared, no llegó a Traefik | `kubectl -n cloudflared logs deploy/cloudflared \| grep -i infisical` — confirmar que cloudflared cargó el hostname. Si no, revisar Public Hostnames en CF dashboard |
| `https://infisical.oscargicast.com` retorna 404 de Traefik | Ingress no encontró el Service | `kubectl -n infisical describe ingress infisical-public` — verificar `Backends` no vacío |
| Logs del operator: `failed to login: getting service account token` | El SA `infisical-operator-controller-manager` no tiene permiso para TokenRequest | Verificar que el chart creó RBAC correctamente: `kubectl -n infisical-operator describe sa infisical-operator-controller-manager` |

---

## Comandos de inspección recurrentes

```bash
# Ver todos los InfisicalSecret del cluster
kubectl get infisicalsecret -A

# Estado detallado de un CR
kubectl -n <ns> describe infisicalsecret <name>

# Logs del operator (filtrar por errores)
kubectl -n infisical-operator logs -l app.kubernetes.io/name=secrets-operator --tail=200 -f

# Logs del backend
kubectl -n infisical logs deploy/infisical --tail=100 -f

# Verificar que un Secret materializado existe y tiene las keys esperadas
kubectl -n <ns> get secret <name> -o jsonpath='{.data}' | python3 -m json.tool

# Forzar reconciliación inmediata (en vez de esperar resyncInterval)
kubectl -n <ns> annotate infisicalsecret <name> reconcile-trigger="$(date +%s)" --overwrite

# Obtener el password de Redis (si necesitás conectarte para debug)
kubectl -n infisical get secret infisical-redis-auth -o jsonpath='{.data.REDIS_PASSWORD}' | base64 -d; echo
```
