# Funcionamiento del Sistema - Migración Docker Compose → RKE2 con Fleet

## 📋 Índice

1. [Arquitectura General](#arquitectura-general)
2. [Componentes del Sistema](#componentes-del-sistema)
3. [Flujo de Despliegue con GitOps](#flujo-de-despliegue-con-gitops)
4. [Cómo Funcionan los Helm Charts](#cómo-funcionan-los-helm-charts)
5. [Cómo Funciona Fleet](#cómo-funciona-fleet)
6. [Ejemplo Práctico: Despliegue de una Tienda](#ejemplo-práctico-despliegue-de-una-tienda)
7. [Comparación: Docker Compose vs Kubernetes](#comparación-docker-compose-vs-kubernetes)

---

## 🏗️ Arquitectura General

### Modelo de Despliegue

```
┌─────────────────────────────────────────────────────────┐
│                    Rancher (Atlas Core)                 │
│              ┌──────────────────────────┐               │
│              │      Fleet Controller   │               │
│              │   (GitOps Engine)      │               │
│              └──────────────────────────┘               │
│                          │                              │
│                          │ Monitorea repositorios Git   │
│                          ▼                              │
│              ┌──────────────────────────┐               │
│              │   Repositorio GitOps      │               │
│              │   - atlas-apps (charts)  │               │
│              │   - atlas-stores (config)│               │
│              └──────────────────────────┘               │
└─────────────────────────────────────────────────────────┘
                          │
                          │ Aplica según labels
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Cluster RKE2 (Raspberry Pi)                │
│              Labels: atlas=true, store=tienda-1         │
│                        poslite=pam, wave=pilot          │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │              Namespace: poslite                 │    │
│  │                                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐           │    │
│  │  │ PostgreSQL   │  │   PgAdmin    │           │    │
│  │  │ StatefulSet  │  │  Deployment  │           │    │
│  │  │ hostPort:5432│  │ hostPort:5050│           │    │
│  │  └──────────────┘  └──────────────┘           │    │
│  │                                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐           │    │
│  │  │ Core Portal  │  │ Core WebAPI │           │    │
│  │  │ hostPort:    │  │ hostPort:   │           │    │
│  │  │   10014      │  │   10012     │           │    │
│  │  └──────────────┘  └──────────────┘           │    │
│  │                                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐           │    │
│  │  │ PAM Portal   │  │ PAM WebAPI   │           │    │
│  │  │ hostPort:    │  │ hostPort:    │           │    │
│  │  │   7014       │  │   7012       │           │    │
│  │  └──────────────┘  └──────────────┘           │    │
│  │                                                 │    │
│  │  ┌──────────────┐  ┌──────────────┐           │    │
│  │  │ Workers      │  │ Workers      │           │    │
│  │  │ (sin puertos)│  │ (sin puertos)│           │    │
│  │  └──────────────┘  └──────────────┘           │    │
│  │                                                 │    │
│  │  ┌──────────────────────────────────────┐    │    │
│  │  │      Cloudflared                      │    │    │
│  │  │      Deployment                       │    │    │
│  │  │      hostNetwork: true                │    │    │
│  │  │      → localhost:7010, 7012, etc.    │    │    │
│  │  └──────────────────────────────────────┘    │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 🧩 Componentes del Sistema

### 1. Helm Charts (Plantillas Reutilizables)

Los **Helm Charts** son plantillas parametrizables que definen cómo se despliegan los servicios en Kubernetes.

#### Estructura de un Chart

```
poslite-horustech/
├── Chart.yaml          # Metadatos del chart
├── values.yaml         # Valores por defecto
└── templates/          # Plantillas de recursos K8s
    ├── deployment-guard-api.yaml
    ├── deployment-portal.yaml
    ├── deployment-workers.yaml
    ├── configmap.yaml
    ├── secret.yaml
    └── pvc-*.yaml
```

#### ¿Cómo Funciona?

1. **Helm lee `values.yaml`**: Valores por defecto
2. **Helm procesa templates**: Reemplaza variables `{{ .Values.xxx }}`
3. **Helm genera manifests**: Archivos YAML de Kubernetes
4. **Kubernetes aplica**: Crea los recursos (Deployments, Services, etc.)

**Ejemplo de Template:**

```yaml
# templates/deployment-portal.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "poslite-horustech.fullname" . }}-portal
spec:
  replicas: {{ .Values.portal.replicas }}  # ← Valor desde values.yaml
  template:
    spec:
      containers:
      - name: portal
        image: "{{ .Values.imageRegistry }}/{{ .Values.imageRepository }}/{{ .Values.portal.image.repository }}:{{ .Values.portal.image.tag }}"
        ports:
        - containerPort: {{ .Values.portal.port }}      # 8080
          hostPort: {{ .Values.portal.hostPort }}       # 9014
```

**Cuando Helm procesa esto con `values.yaml`:**
```yaml
portal:
  replicas: 1
  port: 8080
  hostPort: 9014
  image:
    repository: core/portal
    tag: latest
```

**Genera:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poslite-horustech-portal
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: portal
        image: "aspposlite.azurecr.io/asptg.com/ierpposlite/core/portal:latest"
        ports:
        - containerPort: 8080
          hostPort: 9014
```

---

### 2. Fleet Bundles (Configuración de Despliegue)

Los **Fleet Bundles** definen **QUÉ** desplegar y **DÓNDE** desplegarlo según los labels de los clusters.

#### Ejemplo: Bundle de Horustech

```yaml
# fleet/bundles/horustech/fleet.yaml
defaultNamespace: poslite
targetCustomizations:
- name: horustech
  clusterSelector:
    matchLabels:
      atlas: "true"
      poslite: "horustech"    # ← Solo clusters con este label
  helm:
    chart: ../../charts/poslite-horustech
    values:
      guardApi:
        enabled: true
      portal:
        enabled: true
```

**¿Qué hace esto?**

1. Fleet busca clusters con labels: `atlas: "true"` Y `poslite: "horustech"`
2. Cuando encuentra uno, despliega el chart `poslite-horustech`
3. Aplica los valores especificados (guardApi enabled, portal enabled, etc.)

---

### 3. Repositorio atlas-stores (Configuración por Tienda)

Este repositorio contiene la configuración **específica** de cada tienda.

#### Estructura

```
atlas-stores/
├── stores/
│   └── tienda-1/
│       ├── values-db.yaml          # Config BD para esta tienda
│       ├── values-pam.yaml         # Config PAM para esta tienda
│       └── secrets.sops.yaml       # Secretos encriptados
└── groups/
    └── pilot.yaml                  # Grupos de tiendas
```

#### Ejemplo: Configuración de una Tienda

**`stores/tienda-1/values-pam.yaml`:**
```yaml
config:
  pamIp: "192.168.1.100"        # IP específica de esta tienda
  ierpUrl: "http://ierp..."     # URL específica
  timezone: "America/Panama"
```

**`stores/tienda-1/secrets.sops.yaml` (encriptado):**
```yaml
db-connection-string: "Host=..."  # Cadena de conexión específica
pam-password: "password123"        # Password específica
```

---

## 🔄 Flujo de Despliegue con GitOps

### Paso a Paso

```
┌─────────────────────────────────────────────────────────────┐
│ 1. DESARROLLADOR hace commit en Git                         │
│                                                              │
│    git add .                                                │
│    git commit -m "Actualizar versión de PAM a latest"      │
│    git push                                                 │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. FLEET detecta cambio en repositorio Git                  │
│                                                              │
│    - Fleet Controller monitorea el repo                     │
│    - Detecta nuevo commit                                   │
│    - Lee los bundles y configuración                        │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. FLEET identifica clusters objetivo                       │
│                                                              │
│    - Lee labels de clusters registrados                     │
│    - Compara con clusterSelector de bundles                 │
│    - Ejemplo: cluster con "poslite: pam" → aplica bundle   │
│               de PAM                                        │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. FLEET procesa Helm Charts                                 │
│                                                              │
│    - Toma chart de atlas-apps                               │
│    - Combina con valores de atlas-stores                    │
│    - Genera manifests de Kubernetes                        │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. FLEET aplica en el cluster RKE2                         │
│                                                              │
│    - Envía manifests al cluster                             │
│    - Kubernetes crea/actualiza recursos                     │
│    - Rolling update de Deployments                         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. SERVICIOS funcionando                                    │
│                                                              │
│    - Pods corriendo                                         │
│    - Puertos expuestos (hostPort)                          │
│    - Cloudflared conectado                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 📦 Cómo Funcionan los Helm Charts

### Ejemplo: Chart de Horustech

#### 1. Valores por Defecto (`values.yaml`)

```yaml
# Valores estándar que aplican a todas las tiendas
portal:
  enabled: true
  port: 8080
  hostPort: 9014        # ← PUERTO CONGELADO
  replicas: 1
  image:
    repository: core/portal
    tag: latest
```

#### 2. Template de Deployment

```yaml
# templates/deployment-portal.yaml
{{- if .Values.portal.enabled }}  # ← Solo se crea si enabled=true
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "poslite-horustech.fullname" . }}-portal
spec:
  replicas: {{ .Values.portal.replicas }}
  template:
    spec:
      containers:
      - name: portal
        image: "{{ .Values.imageRegistry }}/{{ .Values.imageRepository }}/{{ .Values.portal.image.repository }}:{{ .Values.portal.image.tag }}"
        ports:
        - containerPort: {{ .Values.portal.port }}
          hostPort: {{ .Values.portal.hostPort }}  # ← 9014
```

#### 3. Valores Específicos de Tienda

```yaml
# atlas-stores/stores/tienda-1/values-horustech.yaml
portal:
  image:
    tag: stable  # ← Sobrescribe el "latest" por defecto
```

#### 4. Resultado Final

Cuando Fleet despliega, combina:
- **Valores por defecto** (del chart)
- **Valores del bundle** (fleet.yaml)
- **Valores de la tienda** (values-horustech.yaml)

**Resultado:** Deployment con `hostPort: 9014` y `tag: stable`

---

## 🎯 Cómo Funciona Fleet

### Selectores de Clusters

Fleet usa **labels** para decidir qué desplegar dónde:

```yaml
# fleet/bundles/horustech/fleet.yaml
clusterSelector:
  matchLabels:
    atlas: "true"           # ← Cluster debe tener este label
    poslite: "horustech"    # ← Y este también
```

### Labels en el Cluster

Cuando registras un cluster en Rancher, le asignas labels:

```yaml
# Cluster: rpi-tienda-1
labels:
  atlas: "true"
  store: "tienda-1"
  poslite: "horustech"    # ← Determina qué bundle aplicar
  wave: "pilot"
```

### Proceso de Matching

```
Fleet Bundle: "horustech"
  Requiere: atlas=true AND poslite=horustech

Cluster: "rpi-tienda-1"
  Tiene: atlas=true, poslite=horustech, store=tienda-1
  ✅ MATCH → Despliega bundle horustech

Cluster: "rpi-tienda-2"
  Tiene: atlas=true, poslite=pam, store=tienda-2
  ❌ NO MATCH → No despliega bundle horustech
```

---

## 🏪 Ejemplo Práctico: Despliegue de una Tienda

### Escenario: Tienda "Gasolinera Central" con PAM

#### Paso 1: Configurar el Cluster

En Rancher, registrar el cluster y aplicar labels:

```yaml
Cluster: rpi-gasolinera-central
Labels:
  atlas: "true"
  store: "gasolinera-central"
  poslite: "pam"
  wave: "pilot"
```

#### Paso 2: Crear Configuración de la Tienda

**`atlas-stores/stores/gasolinera-central/values-db.yaml`:**
```yaml
postgresql:
  password: ""  # Se proporciona vía Secret
  timezone: "America/Panama"
```

**`atlas-stores/stores/gasolinera-central/values-pam.yaml`:**
```yaml
config:
  pamIp: "192.168.1.50"
  ierpUrl: "http://ierpgateway.azurewebsites.net/"
  timezone: "America/Panama"
```

**`atlas-stores/stores/gasolinera-central/secrets.sops.yaml`:**
```yaml
db-connection-string: "Host=poslite-postgres-db;Database=poslite;..."
pam-password: "password123"
```

#### Paso 3: Fleet Detecta y Despliega

1. **Fleet lee `fleet/bundles/db/fleet.yaml`**:
   - Busca clusters con `atlas: "true"`
   - Encuentra "rpi-gasolinera-central"
   - Despliega chart `poslite-db`

2. **Fleet lee `fleet/bundles/pam/fleet.yaml`**:
   - Busca clusters con `atlas: "true"` AND `poslite: "pam"`
   - Encuentra "rpi-gasolinera-central"
   - Despliega chart `poslite-pam`

3. **Fleet combina valores**:
   - Chart `poslite-pam` (valores por defecto)
   - Bundle `pam` (valores del bundle)
   - Tienda `gasolinera-central` (valores específicos)

#### Paso 4: Kubernetes Crea Recursos

Fleet envía al cluster los siguientes recursos:

```yaml
# PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: poslite-db-postgres
spec:
  template:
    spec:
      containers:
      - name: postgres
        ports:
        - containerPort: 5432
          hostPort: 5432  # ← Expuesto directamente en el host
```

```yaml
# PAM Portal
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poslite-pam-portal
spec:
  template:
    spec:
      containers:
      - name: portal
        image: "aspposlite.azurecr.io/.../core/portal:stable"
        ports:
        - containerPort: 8080
          hostPort: 7014  # ← PUERTO CONGELADO
        env:
        - name: PamApiClient:BaseUrl
          value: "http://poslite-pam-tcpconnector:8080/"
```

```yaml
# Cloudflared
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poslite-cloudflared
spec:
  template:
    spec:
      hostNetwork: true  # ← Accede a localhost
      containers:
      - name: cloudflared
        command:
        - cloudflared
        - tunnel
        - run
        # Config apunta a localhost:7014, localhost:7012, etc.
```

#### Paso 5: Servicios Funcionando

```
Host (Raspberry Pi)
├── Puerto 5432 → PostgreSQL
├── Puerto 5050 → PgAdmin
├── Puerto 7010 → PAM TCP Connector
├── Puerto 7011 → Playwright
├── Puerto 7012 → PAM Core WebAPI
├── Puerto 7014 → PAM Portal
├── Puerto 7015 → Guard API
├── Puerto 7016 → Scraper
└── Puerto 7020 → Core WebEvents

Cloudflared (hostNetwork: true)
└── Conecta a localhost:7010, localhost:7012, etc.
    └── Expone vía tunnel de Cloudflare
```

---

## 🔀 Comparación: Docker Compose vs Kubernetes

### Docker Compose (Antes)

```yaml
# compose.yaml
services:
  pam_portal:
    image: aspposlite.azurecr.io/.../portal:stable
    ports:
      - "7014:8080"  # ← host:container
    environment:
      TZ: "America/Panama"
      CoreApiClient:BaseUrl: "http://pam-core-webapi:8080/"
    volumes:
      - /datos/uploads:/app/wwwroot/uploads
    networks:
      - poslite_net
```

**Problemas:**
- ❌ Configuración en archivos locales
- ❌ Sin versionado de cambios
- ❌ Rollback manual
- ❌ Sin health checks automáticos
- ❌ Escalado manual

### Kubernetes con Helm + Fleet (Ahora)

```yaml
# Chart template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "poslite-pam.fullname" . }}-portal
spec:
  replicas: {{ .Values.portal.replicas }}
  template:
    spec:
      containers:
      - name: portal
        image: "{{ .Values.imageRegistry }}/.../{{ .Values.portal.image.tag }}"
        ports:
        - containerPort: 8080
          hostPort: 7014  # ← Mismo puerto
        env:
        - name: TZ
          value: {{ .Values.config.timezone }}
        volumeMounts:
        - name: uploads
          mountPath: /app/wwwroot/uploads
        livenessProbe:    # ← Health check automático
          httpGet:
            path: /
            port: 8080
```

**Ventajas:**
- ✅ Configuración en Git (versionado)
- ✅ GitOps: cambios = commit
- ✅ Rollback automático: `git revert`
- ✅ Health checks integrados
- ✅ Escalado: cambiar `replicas` en values.yaml
- ✅ Gestión centralizada desde Rancher

---

## 🔐 Gestión de Secretos

### Docker Compose (Antes)

```bash
# Archivo .env local (no versionado)
DB_PASSWORD=secret123
PAM_PASSWORD=secret456
```

### Kubernetes (Ahora)

```yaml
# Secret en Kubernetes
apiVersion: v1
kind: Secret
metadata:
  name: poslite-pam-config
type: Opaque
stringData:
  db-connection-string: "Host=..."
  pam-password: "secret456"
```

**Con SOPS (encriptado en Git):**

```yaml
# secrets.sops.yaml (encriptado)
db-connection-string: ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]
pam-password: ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]
```

**Ventaja:** Secretos versionados en Git pero encriptados.

---

## 🌊 Flujo de Actualización

### Actualizar Versión de un Servicio

#### Antes (Docker Compose)

```bash
# 1. Editar compose.yaml manualmente
# 2. Cambiar tag: latest → stable
# 3. Ejecutar
docker-compose down
docker-compose up -d
# 4. Si falla, rollback manual
```

#### Ahora (GitOps)

```bash
# 1. Editar values.yaml en Git
#    Cambiar: tag: latest → tag: stable
git add atlas-stores/stores/tienda-1/values-pam.yaml
git commit -m "Actualizar PAM a stable"
git push

# 2. Fleet detecta cambio automáticamente
# 3. Fleet aplica actualización (rolling update)
# 4. Si falla, rollback:
git revert HEAD
git push
# Fleet revierte automáticamente
```

---

## 📊 Matriz de Puertos (Funcionamiento)

### ¿Por qué hostPort?

En un cluster **single-node** (un solo Raspberry Pi), `hostPort` expone el puerto directamente en el host, igual que Docker Compose.

```
┌─────────────────────────────────────┐
│  Raspberry Pi (Host)                │
│                                     │
│  ┌───────────────────────────────┐ │
│  │  Kubernetes Pod              │ │
│  │  Container: PAM Portal        │ │
│  │  hostPort: 7014               │ │
│  └───────────────────────────────┘ │
│           │                         │
│           │ Puerto 7014 expuesto   │
│           ▼                         │
│  ┌───────────────────────────────┐ │
│  │  Cloudflared                  │ │
│  │  hostNetwork: true            │ │
│  │  → localhost:7014            │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
```

**Resultado:** 
- Puerto 7014 accesible desde el host
- Cloudflared puede conectarse a `localhost:7014`
- **Igual que Docker Compose**

---

## 🔄 Comunicación Interna

### Servicios se Comunican por Nombre

En Kubernetes, los servicios se comunican usando **Service names**:

```yaml
# Deployment de Portal
env:
- name: CoreApiClient:BaseUrl
  value: "http://poslite-pam-core-webapi:8080/"
  #     ↑ Nombre del Service (no IP)
```

**Kubernetes DNS resuelve:**
- `poslite-pam-core-webapi` → IP del Service
- El Service enruta al Pod correcto

**Equivalente a Docker Compose:**
```yaml
# Docker Compose
environment:
  CoreApiClient:BaseUrl: "http://pam-core-webapi:8080/"
  #                    ↑ Nombre del servicio
```

---

## 🎛️ Control de Despliegue

### Habilitar/Deshabilitar Servicios

**En Docker Compose:**
- Comentar servicio en `compose.yaml`
- Ejecutar `docker-compose up -d`

**En Kubernetes (GitOps):**
```yaml
# values.yaml
workers:
  ierp:
    enabled: false  # ← Deshabilita todos los workers IERP
```

**Fleet aplica automáticamente** → Workers IERP se detienen.

---

## 📝 Resumen del Funcionamiento

### Principios Clave

1. **Git es la Fuente de Verdad**
   - Todo cambio pasa por Git
   - Rollback = git revert

2. **Fleet Aplica Automáticamente**
   - Monitorea repositorios Git
   - Aplica cambios según labels

3. **Helm Parametriza Todo**
   - Charts reutilizables
   - Valores por tienda sobrescriben defaults

4. **Puertos Congelados**
   - `hostPort` mantiene puertos exactos
   - Cloudflared usa `localhost:<PUERTO>`

5. **Single-Node Clusters**
   - Un RPI = Un cluster
   - `hostPort` funciona como Docker Compose

### Flujo Completo

```
Git Commit
    ↓
Fleet Detecta
    ↓
Fleet Lee Bundles
    ↓
Fleet Busca Clusters (por labels)
    ↓
Fleet Combina Valores (chart + bundle + tienda)
    ↓
Fleet Genera Manifests (Helm)
    ↓
Fleet Aplica en Cluster
    ↓
Kubernetes Crea Recursos
    ↓
Pods Corriendo
    ↓
Puertos Expuestos (hostPort)
    ↓
Cloudflared Conecta (localhost)
    ↓
Sistema Funcionando
```

---

## ❓ Preguntas Frecuentes

### ¿Cómo actualizo una tienda específica?

1. Editar `atlas-stores/stores/<tienda>/values-*.yaml`
2. `git commit && git push`
3. Fleet aplica automáticamente

### ¿Cómo actualizo todas las tiendas?

1. Editar `atlas-apps/charts/poslite-*/values.yaml`
2. `git commit && git push`
3. Fleet aplica a todos los clusters

### ¿Cómo hago rollback?

```bash
git revert <commit-hash>
git push
# Fleet revierte automáticamente
```

### ¿Cómo veo qué está desplegado?

```bash
# En Rancher UI
# O con kubectl
kubectl get pods -n poslite
kubectl get deployments -n poslite
```

### ¿Los puertos son realmente los mismos?

**Sí.** Todos los `hostPort` son **exactamente** los mismos que Docker Compose:
- PostgreSQL: 5432 ✅
- PAM Portal: 7014 ✅
- Horustech WebAPI: 9010 ✅
- etc.

---

**¿Tienes alguna pregunta específica sobre algún componente?**
