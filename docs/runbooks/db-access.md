# Runbook: acceso a las DBs CNPG (postgres-lab, n8n-postgres)

Las bases de datos del cluster están **solo intra-cluster** (Services ClusterIP generados por el operator de CloudNativePG). No están expuestas por Traefik ni por Cloudflare Tunnel **a propósito**: en un homelab sin necesidad de acceso 24/7 desde Internet, el criterio es minimizar superficie de ataque. El patrón soportado para acceso desde la MacBook es **port-forward ad-hoc desde la mac mini**, accesible vía Tailscale.

## Topología

CNPG genera 3 Services por `Cluster` (todos `ClusterIP`, puerto `5432`):

| Service | Apunta a | Cuándo usarlo |
|---|---|---|
| `<cluster>-rw` | primary (escritura) | DDL, escrituras, queries críticas |
| `<cluster>-ro` | réplica solo-lectura | reportes, lecturas pesadas |
| `<cluster>-r` | cualquier instancia (rw o ro) | lecturas tolerantes |

Para los clusters actuales (`databases` namespace):

| Cluster | DB inicial | Usuario | SealedSecret con credenciales |
|---|---|---|---|
| `postgres-lab` | `app` | `app` | `postgres-lab-app-user` |
| `n8n-postgres` | `n8n` | `n8n` | `n8n-db-user` |

Cada SealedSecret se descifra como `Secret` tipo `basic-auth` con claves `username` y `password`.

## Paso 1 — Extraer credenciales (en mac mini)

```bash
# postgres-lab
kubectl -n databases get secret postgres-lab-app-user -o jsonpath='{.data.password}' | base64 -d; echo

# n8n-postgres
kubectl -n databases get secret n8n-db-user -o jsonpath='{.data.password}' | base64 -d; echo
```

> Para validar el usuario, sustituí `password` por `username` en el `jsonpath`.

CNPG además crea un secret `<cluster>-superuser` con el role `postgres` — útil para tareas administrativas; mismo formato.

## Paso 2 — Abrir port-forward (en mac mini)

```bash
# postgres-lab → puerto local 15432
kubectl port-forward -n databases svc/postgres-lab-rw \
  --address 0.0.0.0 15432:5432

# n8n-postgres → puerto local 15433
kubectl port-forward -n databases svc/n8n-postgres-rw \
  --address 0.0.0.0 15433:5432
```

`--address 0.0.0.0` es necesario para que la conexión sea alcanzable vía la IP de Tailscale de la mac mini (no solo por `localhost`). Los puertos `15432`/`15433` evitan choques con un Postgres local que pudieras tener corriendo en la MacBook (`5432` típicamente ocupado).

Mantené la terminal abierta mientras dure la sesión; `Ctrl+C` cierra el port-forward.

## Paso 3 — Conectarse desde la MacBook

```bash
# postgres-lab
PGPASSWORD='<password-paso-1>' psql \
  -h oscar-mini-m1.tail90f0a7.ts.net \
  -p 15432 \
  -U app \
  -d app

# n8n-postgres
PGPASSWORD='<password-paso-1>' psql \
  -h oscar-mini-m1.tail90f0a7.ts.net \
  -p 15433 \
  -U n8n \
  -d n8n
```

Alternativa con connection string (útil para clientes GUI como TablePlus, DBeaver, DataGrip):

```
postgresql://app:<password>@oscar-mini-m1.tail90f0a7.ts.net:15432/app
postgresql://n8n:<password>@oscar-mini-m1.tail90f0a7.ts.net:15433/n8n
```

## Acceso intra-cluster (workloads dentro del cluster)

Para servicios del cluster (n8n, jobs futuros, etc.) usar DNS interno — no port-forward:

```
postgres-lab-rw.databases.svc.cluster.local:5432
n8n-postgres-rw.databases.svc.cluster.local:5432
```

n8n ya usa este patrón (`automation/n8n/values.yaml`).

## Smoke test rápido (sin levantar psql)

Si solo querés confirmar conectividad antes de armar el túnel:

```bash
# Desde la mac mini, un pod efímero dentro del cluster
kubectl run -n databases psql-test --rm -it --restart=Never \
  --image=postgres:16-alpine -- \
  psql "postgresql://app:$(kubectl -n databases get secret postgres-lab-app-user -o jsonpath='{.data.password}' | base64 -d)@postgres-lab-rw:5432/app" \
  -c '\l'
```

## ¿Por qué no exponerlo por Traefik / Cloudflare Tunnel?

- Traefik corre solo con entrypoint `web` (HTTP). Habilitar TCP requiere agregar entrypoint, abrir puerto en el host, sumar `IngressRouteTCP` y mantener esa pieza — más superficie, beneficio bajo en un homelab.
- Cloudflare Tunnel soporta TCP, pero requiere `cloudflared access tcp` del lado cliente: no es transparente para clientes Postgres, y agrega una dependencia extra.
- El port-forward via Tailscale ya es seguro (zero-trust mesh), no expone nada por defecto, y el comando es trivial.

Si en algún momento aparece un caso de uso real (e.g. un servicio externo persistente que necesita postgres), revisitar la decisión y agregar la pieza correspondiente acá.
