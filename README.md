# Homelab Kubernetes en Mac mini M1

## Objetivo

Queremos convertir una **Mac mini M1 headless**, accesible solo por SSH vía Tailscale, en un pequeño laboratorio Kubernetes para aprender y correr servicios reales.

La idea no es solo instalar herramientas, sino entender qué hace cada componente y cómo se relacionan entre sí.

```txt
MacBook Pro
  └── SSH vía Tailscale
        └── Mac mini M1
              └── Colima VM
                    └── K3s / Kubernetes
                          ├── PostgreSQL con CloudNativePG
                          ├── Observabilidad con Prometheus, Loki y Grafana
                          ├── Automatización con n8n
                          └── Dashboard homelab con Homepage
```

---

## Estado actual

La Mac mini ya tiene instalado:

```txt
- Colima
- Docker
- kubectl
- Helm
- k9s
```

Actualmente el cluster se está levantando con:

```bash
colima start --kubernetes --cpu 4 --memory 8 --disk 200
```

Esto significa:

```txt
--kubernetes
  Levanta Kubernetes dentro de Colima.

--cpu 4
  Asigna 4 CPUs virtuales a la VM de Colima.

--memory 8
  Asigna 8 GB de RAM a la VM.

--disk 200
  Reserva hasta 200 GB de disco para imágenes, volúmenes y datos.
```

---

## Arquitectura general

```txt
┌──────────────────────────────────────┐
│ MacBook Pro                          │
│ - Terminal                           │
│ - SSH                                │
│ - Navegador                          │
└───────────────────┬──────────────────┘
                    │ Tailscale
                    ▼
┌──────────────────────────────────────┐
│ Mac mini M1                           │
│ - macOS                               │
│ - Colima                              │
│ - Docker CLI                          │
│ - kubectl / helm / k9s                │
└───────────────────┬──────────────────┘
                    │
                    ▼
┌──────────────────────────────────────┐
│ Colima VM Linux                       │
│ - Docker runtime                      │
│ - K3s / Kubernetes                    │
└───────────────────┬──────────────────┘
                    │
                    ▼
┌──────────────────────────────────────┐
│ Kubernetes                            │
│ - Namespaces                          │
│ - Pods                                │
│ - Services                            │
│ - PVCs                                │
│ - Helm releases                       │
│ - Operators                           │
└──────────────────────────────────────┘
```

---

## Stack principal

| Componente | Rol |
|---|---|
| Colima | VM Linux para correr contenedores y Kubernetes en macOS |
| K3s | Kubernetes ligero |
| kubectl | CLI para controlar Kubernetes |
| Helm | Instalador / package manager de Kubernetes |
| k9s | UI en terminal para inspeccionar el cluster |
| CloudNativePG | Operador para manejar PostgreSQL dentro de Kubernetes |
| Prometheus | Métricas |
| Grafana | Dashboards |
| Loki | Logs |
| n8n | Automatización de workflows |
| Homepage | Dashboard visual para servicios del homelab |

---

## Conceptos clave

### Colima

Colima es la pieza que permite que en macOS corramos contenedores Linux y Kubernetes. macOS no corre contenedores Linux de forma nativa, por eso Colima crea una VM Linux.

En nuestro caso:

```bash
colima start --kubernetes --cpu 4 --memory 8 --disk 200
```

### Kubernetes

Kubernetes es el orquestador. Su trabajo es correr aplicaciones como contenedores, mantenerlas vivas, conectarlas entre sí y administrar recursos como red, almacenamiento y configuración.

Los objetos más importantes al inicio son:

| Objeto | Qué hace |
|---|---|
| Pod | Unidad mínima que corre uno o más contenedores |
| Deployment | Mantiene réplicas de pods corriendo |
| Service | Da una IP/nombre estable para acceder a pods |
| Namespace | Agrupa recursos lógicamente |
| Secret | Guarda credenciales |
| ConfigMap | Guarda configuración no sensible |
| PVC | Pide almacenamiento persistente |
| Ingress | Expone servicios HTTP hacia fuera del cluster |

### Helm

Helm es como un package manager para Kubernetes. En vez de escribir todos los YAML manualmente, instalamos aplicaciones usando charts.

Ejemplo:

```bash
helm install grafana grafana/grafana
```

### k9s

k9s es una interfaz en terminal para navegar Kubernetes. No instala nada dentro del cluster; solo te ayuda a ver pods, logs, namespaces, services, deployments, PVCs y eventos.

---

## Restricción importante: single-node cluster

Este homelab corre en una sola máquina física.

Eso significa:

```txt
Tenemos Kubernetes, pero no alta disponibilidad real.
```

Podemos aprender:

```txt
- Deployments
- Services
- Ingress
- Operators
- Volúmenes
- Observabilidad
- Bases de datos
- Backups
- Automatización
```

Pero no tendremos alta disponibilidad real porque solo hay un nodo.

Para desarrollo y aprendizaje está perfecto.

Para producción real, no.

---

## Recursos disponibles

Tu Mac mini tiene 16 GB de memoria unificada.

Actualmente Colima usa:

```txt
CPU:    4
RAM:    8 GB
Disco:  200 GB
```

Esto es razonable para empezar.

### Presupuesto aproximado de memoria

| Servicio | Memoria aproximada |
|---|---:|
| Kubernetes / K3s base | 500 MB - 1 GB |
| CloudNativePG operator | 100 - 300 MB |
| PostgreSQL pequeño | 512 MB - 1 GB |
| Prometheus stack | 1 - 2 GB |
| Grafana | 300 - 600 MB |
| Loki | 512 MB - 1.5 GB |
| n8n | 512 MB - 1 GB |
| Homepage | 100 - 300 MB |

Con 8 GB para Colima, debemos ir paso a paso y poner límites de memoria.

---

## Namespaces propuestos

Usaremos namespaces para separar responsabilidades:

```txt
databases
  PostgreSQL y operadores relacionados a bases de datos.

observability
  Prometheus, Grafana, Loki y collectors.

automation
  n8n y workflows.

homelab
  Dashboard tipo Homepage.

ingress
  Ingress controller o recursos de entrada si los separamos.

cnpg-system
  Operador CloudNativePG.
```

Crear namespaces:

```bash
kubectl create namespace databases --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace automation --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace homelab --dry-run=client -o yaml | kubectl apply -f -
```

Ver namespaces:

```bash
kubectl get ns
```

---

## Fase 1: validar Kubernetes

Antes de instalar servicios, validamos que Kubernetes esté sano.

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass
```

Esperamos ver algo parecido a:

```txt
NAME     STATUS   ROLES
colima   Ready    control-plane
```

También deberíamos ver una StorageClass, probablemente `local-path`.

La StorageClass es importante porque las bases de datos necesitan almacenamiento persistente.

---

## Fase 2: instalar CloudNativePG

El nombre correcto es **CloudNativePG**, también abreviado **CNPG**.

CloudNativePG es un operador para manejar PostgreSQL en Kubernetes.

### Qué es un operador

Un operador es un controlador que extiende Kubernetes.

Kubernetes sabe manejar cosas genéricas como:

```txt
- Pods
- Services
- Deployments
- Secrets
- PVCs
```

Pero Kubernetes no sabe, por defecto, manejar bien algo complejo como PostgreSQL.

CloudNativePG agrega nuevos recursos, por ejemplo:

```yaml
kind: Cluster
```

Ese `Cluster` representa un cluster PostgreSQL.

### Instalar operador

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg
```

Validar:

```bash
kubectl get pods -n cnpg-system
kubectl get crds | grep cnpg
```

---

## Fase 3: crear primer PostgreSQL

Este será nuestro primer PostgreSQL manejado por CloudNativePG.

Crear estructura:

```bash
mkdir -p ~/homelab-k8s/databases
cd ~/homelab-k8s/databases
```

Crear `postgres-lab.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-lab-app-user
  namespace: databases
type: kubernetes.io/basic-auth
stringData:
  username: app
  password: app-password-change-me
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-lab
  namespace: databases
spec:
  instances: 1

  storage:
    size: 10Gi

  bootstrap:
    initdb:
      database: app
      owner: app
      secret:
        name: postgres-lab-app-user
```

Aplicar:

```bash
kubectl apply -f postgres-lab.yaml
```

Validar:

```bash
kubectl get clusters -n databases
kubectl get pods -n databases
kubectl get pvc -n databases
kubectl get svc -n databases
```

CloudNativePG normalmente crea servicios como:

```txt
postgres-lab-rw
postgres-lab-ro
postgres-lab-r
```

Conceptualmente:

| Service | Uso |
|---|---|
| `-rw` | Escritura y lectura hacia el primary |
| `-ro` | Lecturas hacia réplicas |
| `-r` | Cualquier instancia disponible |

Para este homelab, usaremos principalmente `postgres-lab-rw`.

---

## Fase 4: acceso a servicios desde la MacBook Pro

Como la Mac mini está headless y accedes por SSH vía Tailscale, al inicio usaremos `kubectl port-forward`.

Ejemplo para PostgreSQL:

```bash
kubectl port-forward -n databases svc/postgres-lab-rw 5432:5432
```

Eso expone PostgreSQL solo dentro de la Mac mini.

Si quieres acceder desde la MacBook Pro usando la IP Tailscale de la Mac mini:

```bash
kubectl port-forward \
  -n databases \
  svc/postgres-lab-rw \
  --address 0.0.0.0 \
  5432:5432
```

Luego desde la MacBook Pro:

```bash
psql "postgresql://app:app-password-change-me@<tailscale-ip-mac-mini>:5432/app"
```

Más adelante evitaremos depender de port-forward usando Ingress, Tailscale DNS o servicios internos.

---

## Fase 5: observabilidad

Queremos tres piezas:

```txt
Prometheus
  Métricas.

Loki
  Logs.

Grafana
  Dashboards para ver métricas y logs.
```

### Prometheus

Instalaremos `kube-prometheus-stack`, que empaqueta Prometheus Operator, Prometheus, Alertmanager, Grafana, dashboards y reglas.

Agregar repo:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Instalar:

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --set grafana.adminPassword='admin-change-me' \
  --set prometheus.prometheusSpec.retention=3d \
  --set prometheus.prometheusSpec.resources.requests.memory=512Mi \
  --set prometheus.prometheusSpec.resources.limits.memory=1Gi
```

Validar:

```bash
kubectl get pods -n observability
kubectl get svc -n observability
```

Acceder a Grafana:

```bash
kubectl port-forward \
  -n observability \
  svc/monitoring-grafana \
  --address 0.0.0.0 \
  3000:80
```

Desde la MacBook Pro:

```txt
http://<tailscale-ip-mac-mini>:3000
```

Credenciales:

```txt
user: admin
pass: admin-change-me
```

---

## Fase 6: Loki

Loki es para logs.

Prometheus responde preguntas como:

```txt
¿Cuánta CPU está usando este pod?
¿Cuánta memoria consume PostgreSQL?
¿Cuántos pods están corriendo?
```

Loki responde preguntas como:

```txt
¿Qué errores imprimió este pod?
¿Qué pasó antes de que n8n fallara?
¿Qué logs generó PostgreSQL?
```

Agregar repo:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Para este homelab usaremos Loki en modo monolítico. Conviene instalarlo con un `values.yaml`, no solo con muchos `--set`, porque Loki suele requerir configuración explícita de storage.

---

## Fase 7: n8n

n8n será nuestro servicio de automatización.

Casos de uso posibles:

```txt
- Automatizar tareas personales.
- Ejecutar workflows programados.
- Conectarse a APIs.
- Crear integraciones con Telegram, Slack, Gmail, GitHub, etc.
- Probar agentes o automatizaciones conectadas a servicios internos.
```

### Base de datos de n8n

No queremos que n8n use SQLite para algo persistente.

Queremos:

```txt
n8n
  └── PostgreSQL manejado por CloudNativePG
```

Crearíamos un cluster PostgreSQL dedicado:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: n8n-db-user
  namespace: automation
type: kubernetes.io/basic-auth
stringData:
  username: n8n
  password: n8n-password-change-me
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: n8n-postgres
  namespace: automation
spec:
  instances: 1

  storage:
    size: 10Gi

  bootstrap:
    initdb:
      database: n8n
      owner: n8n
      secret:
        name: n8n-db-user
```

Luego instalamos n8n apuntando a ese Postgres.

---

## Fase 8: dashboard homelab

Para una web tipo homelab usaremos **Homepage**.

Queremos ver algo como:

```txt
Homelab Dashboard
├── Observability
│   ├── Grafana
│   ├── Prometheus
│   └── Loki
├── Automation
│   └── n8n
├── Databases
│   └── PostgreSQL / CNPG
└── Kubernetes
    ├── k9s
    ├── Nodes
    └── Pods
```

Homepage no reemplaza a Grafana.

Homepage es tu portal.

Grafana es para métricas y logs.

---

## Estrategia de exposición de servicios

Como estás conectado vía SSH/Tailscale, hay tres etapas.

### Etapa 1: port-forward

Más simple para empezar.

```bash
kubectl port-forward -n observability svc/monitoring-grafana --address 0.0.0.0 3000:80
```

Ventaja:

```txt
Fácil.
No requiere Ingress.
Ideal para aprender.
```

Desventaja:

```txt
Si cierras la terminal, se corta.
No es cómodo para muchos servicios.
```

### Etapa 2: Ingress interno

Usaríamos un Ingress Controller para exponer servicios con rutas:

```txt
grafana.homelab.local
n8n.homelab.local
homepage.homelab.local
```

Ventaja:

```txt
URLs limpias.
Más parecido a producción.
```

Desventaja:

```txt
Requiere configurar DNS o /etc/hosts.
```

### Etapa 3: Tailscale-friendly

Usaríamos Tailscale para acceder de forma segura desde tus dispositivos.

Posibles rutas:

```txt
http://mac-mini-tailscale:3000
http://grafana.<tailnet>
http://n8n.<tailnet>
```

Esto lo dejamos para después de tener el stack base funcionando.

---

## Orden de implementación

No instalaremos todo de golpe.

Orden recomendado:

```txt
1. Validar Colima + Kubernetes.
2. Crear namespaces.
3. Validar StorageClass.
4. Instalar CloudNativePG.
5. Crear primer PostgreSQL.
6. Instalar Prometheus + Grafana.
7. Instalar Loki.
8. Conectar Loki a Grafana.
9. Crear PostgreSQL dedicado para n8n.
10. Instalar n8n.
11. Instalar Homepage.
12. Exponer servicios por Tailscale / Ingress.
13. Agregar backups.
14. Agregar dashboards útiles.
```

---

## Comandos base de diagnóstico

### Estado de Colima

```bash
colima status
colima list
```

### Estado de Kubernetes

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get svc -A
kubectl get pvc -A
kubectl get storageclass
```

### Ver eventos

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

### Entrar a k9s

```bash
k9s
```

Dentro de k9s:

```txt
:pods
:svc
:deploy
:ns
:pvc
:events
```

### Ver logs de un pod

```bash
kubectl logs -n <namespace> <pod-name>
```

### Seguir logs en vivo

```bash
kubectl logs -n <namespace> <pod-name> -f
```

### Describir un recurso

```bash
kubectl describe pod -n <namespace> <pod-name>
```

---

## Estructura de carpetas sugerida

En la Mac mini:

```bash
mkdir -p ~/homelab-k8s/{databases,observability,automation,homelab,ingress,scripts}
cd ~/homelab-k8s
```

Estructura:

```txt
homelab-k8s/
├── README.md
├── databases/
│   ├── cnpg-install.md
│   ├── postgres-lab.yaml
│   └── n8n-postgres.yaml
├── observability/
│   ├── prometheus-values.yaml
│   ├── loki-values.yaml
│   └── grafana-notes.md
├── automation/
│   ├── n8n-values.yaml
│   └── n8n-notes.md
├── homelab/
│   ├── homepage-values.yaml
│   └── homepage-config.yaml
├── ingress/
│   └── ingress-notes.md
└── scripts/
    ├── status.sh
    └── port-forwards.sh
```

---

## Primer script útil

Crear `scripts/status.sh`:

```bash
mkdir -p ~/homelab-k8s/scripts

cat <<'EOF' > ~/homelab-k8s/scripts/status.sh
#!/usr/bin/env bash
set -euo pipefail

echo "== Colima =="
colima status || true

echo
echo "== Kubernetes Nodes =="
kubectl get nodes -o wide

echo
echo "== Namespaces =="
kubectl get ns

echo
echo "== Pods =="
kubectl get pods -A

echo
echo "== Services =="
kubectl get svc -A

echo
echo "== PVCs =="
kubectl get pvc -A

echo
echo "== StorageClass =="
kubectl get storageclass
EOF

chmod +x ~/homelab-k8s/scripts/status.sh
```

Ejecutar:

```bash
~/homelab-k8s/scripts/status.sh
```

---

## Reglas del homelab

### Regla 1: no instalar todo sin entenderlo

Cada componente debe responder:

```txt
¿Qué problema resuelve?
¿Qué recursos consume?
Dónde guarda datos?
Cómo lo expongo?
Cómo lo respaldo?
Cómo lo desinstalo?
```

### Regla 2: todo con namespaces

No instalaremos todo en `default`.

### Regla 3: todo persistente debe tener PVC

Aplica para:

```txt
- PostgreSQL
- n8n
- Loki si guarda logs en disco
- Grafana si queremos persistencia
```

### Regla 4: credenciales fuera de YAML público

Para empezar usaremos `Secret` simple.

Luego podemos mejorar con:

```txt
- SOPS
- Sealed Secrets
- External Secrets
- 1Password Connect
```

### Regla 5: backups antes de confiar datos

Antes de usar n8n o PostgreSQL para algo importante, debemos configurar backups.

---

## Lo que NO buscamos todavía

Por ahora evitaremos:

```txt
- Alta disponibilidad real.
- Multi-node Kubernetes.
- Exponer servicios a internet público.
- Certificados TLS públicos.
- GitOps con ArgoCD o Flux.
- Service mesh.
- CI/CD complejo.
```

Eso puede venir después.

Primero queremos una base sólida.

---

## Roadmap corto

### Milestone 1: cluster funcional

```txt
- Colima corriendo.
- Kubernetes Ready.
- k9s funcionando.
- StorageClass disponible.
```

### Milestone 2: PostgreSQL

```txt
- CloudNativePG instalado.
- postgres-lab corriendo.
- PVC creado.
- Conexión vía port-forward.
```

### Milestone 3: observabilidad

```txt
- Prometheus corriendo.
- Grafana accesible.
- Métricas de Kubernetes visibles.
```

### Milestone 4: logs

```txt
- Loki instalado.
- Logs llegando a Grafana.
```

### Milestone 5: n8n

```txt
- PostgreSQL dedicado para n8n.
- n8n instalado.
- n8n accesible por Tailscale.
```

### Milestone 6: dashboard

```txt
- Homepage instalado.
- Links a Grafana, n8n y servicios internos.
- Vista básica del homelab.
```

---

## Siguiente paso inmediato

Ejecutar:

```bash
colima status
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass
```

Si eso está sano, el siguiente paso será:

```txt
Instalar CloudNativePG y crear el primer PostgreSQL.
```
