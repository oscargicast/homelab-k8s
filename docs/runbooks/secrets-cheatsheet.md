# Runbook: extraer secrets del cluster (desde la Mac mini)

Referencia rápida para desencriptar passwords, tokens y credenciales almacenados como Kubernetes Secrets (originados por SealedSecrets o generados por operadores como CNPG).

> Todos los comandos se ejecutan en la **Mac mini** (`ssh macmini`).

---

## Bases de datos (CNPG)

CNPG genera dos secrets por cluster: uno con las credenciales del owner (viene del SealedSecret) y otro `<cluster>-superuser` con el role `postgres`.

### postgres-lab

```bash
# Password del usuario app
kubectl -n databases get secret postgres-lab-app-user \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Username del usuario app
kubectl -n databases get secret postgres-lab-app-user \
  -o jsonpath='{.data.username}' | base64 -d; echo

# Password del superuser (postgres)
kubectl -n databases get secret postgres-lab-superuser \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### n8n-postgres

```bash
# Password del usuario n8n
kubectl -n databases get secret n8n-db-user \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Username
kubectl -n databases get secret n8n-db-user \
  -o jsonpath='{.data.username}' | base64 -d; echo

# Superuser
kubectl -n databases get secret n8n-postgres-superuser \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### evolution-postgres

```bash
# Password del usuario evolution
kubectl -n databases get secret evolution-db-user \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Username
kubectl -n databases get secret evolution-db-user \
  -o jsonpath='{.data.username}' | base64 -d; echo

# Superuser
kubectl -n databases get secret evolution-postgres-superuser \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## Evolution API

```bash
# API Key
kubectl -n automation get secret evolution-api-secrets \
  -o jsonpath='{.data.api-key}' | base64 -d; echo

# Database connection URI
kubectl -n automation get secret evolution-api-secrets \
  -o jsonpath='{.data.database-url}' | base64 -d; echo
```

---

## Argo CD

```bash
# Admin password inicial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## Cloudflare Tunnel

```bash
# Token del tunnel
kubectl -n cloudflared get secret cloudflared-token \
  -o jsonpath='{.data.token}' | base64 -d; echo
```

---

## n8n (credenciales de DB)

```bash
# Password de la DB usada por n8n
kubectl -n automation get secret n8n-db-credentials \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## Comando genérico

Para cualquier secret no listado:

```bash
# Listar todos los secrets de un namespace
kubectl -n <namespace> get secrets

# Ver las keys dentro de un secret
kubectl -n <namespace> get secret <nombre> -o jsonpath='{.data}' | python3 -m json.tool

# Desencriptar una key específica
kubectl -n <namespace> get secret <nombre> \
  -o jsonpath='{.data.<key>}' | base64 -d; echo

# Dump completo (todas las keys desencriptadas)
kubectl -n <namespace> get secret <nombre> -o json \
  | python3 -c "
import sys, json, base64
data = json.load(sys.stdin).get('data', {})
for k, v in data.items():
    print(f'{k}: {base64.b64decode(v).decode()}')
"
```
