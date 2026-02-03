# Qué se crea en Rancher al configurar una tienda en atlas-stores

Cuando configuras una **tienda nueva en atlas-stores** y Fleet aplica el bundle en un cluster de Rancher, se crean los recursos que definen los **templates de cada chart**. Todo se crea en el **namespace `poslite`** (definido en `defaultNamespace` del `fleet.yaml`).

Cada **target** del bundle es un release de Helm (un chart desde ACR). Los recursos que ves en Rancher son los que generan los **templates** de cada chart.

---

## 1. Chart poslite-db (todas las tiendas: Core, PAM, Horustech)

| Recurso | Cantidad | Nombre / Origen | Notas |
|--------|----------|------------------|--------|
| **StatefulSet** | 1 | `<release>-postgres` | PostgreSQL. Incluye **volumeClaimTemplates** → el controlador del StatefulSet crea **1 PVC** por réplica: `data-<release>-postgres-0`. |
| **Deployment** | 1 | `<release>-pgadmin` | Solo si `pgadmin.enabled: true`. |
| **ConfigMap** | 1 | `<release>-config` | Contenido de `postgresql.conf`. |
| **Secret** | 1 | `<release>-secret` | Contraseñas de Postgres y PgAdmin (base64). |
| **PersistentVolumeClaim** | 1 explícito | `<release>-pgadmin-data` | Solo si `pgadmin.enabled` y `persistence.enabled`. Tamaño 5Gi. |
| **PersistentVolumeClaim** | 1 por réplica | `data-<release>-postgres-0` | Creados por el StatefulSet (volumeClaimTemplates), no por un template aparte. Tamaño según `persistence.size` (ej. 20Gi). |

**Nota:** El StatefulSet tiene `serviceName: <release>-postgres`. En Kubernetes, un StatefulSet suele usar un **Service** (Headless) con ese nombre. En el chart **no hay** un template `Service` para Postgres; si en tu cluster no existe ese Service, puede que lo creéis aparte o que falte añadirlo al chart.

**Resumen poslite-db:** 1 StatefulSet, 1 Deployment (PgAdmin), 1 ConfigMap, 1 Secret, 1 PVC explícito (PgAdmin) + 1 PVC por réplica de Postgres (creado por el StatefulSet).

---

## 2. Chart poslite-core (solo tiendas Core)

| Recurso | Cantidad | Descripción |
|--------|----------|-------------|
| **Deployment** | 1 | Portal |
| **Deployment** | 1 | WebAPI |
| **Deployment** | 11 | Workers (price, shift, errorReports, ierp: dispatchesToInvoice, dispatchPayments, invoicesToIerp, customers, markets, basicData, invoiceSeries, products) |
| **Service** | 1 | Asociado a WebAPI (ClusterIP) |
| **ConfigMap** | 1 | Configuración del chart |
| **Secret** | 1 | Credenciales |
| **PersistentVolumeClaim** | 1 | Uploads (si `persistence.uploads.enabled`) |

**Resumen poslite-core:** 13 Deployments, 1 Service, 1 ConfigMap, 1 Secret, 1 PVC (uploads).

---

## 3. Chart poslite-cloudflared-core (solo tiendas Core)

| Recurso | Cantidad | Descripción |
|--------|----------|-------------|
| **Deployment** | 1 | Cloudflared (túnel; suele ir con `hostNetwork: true`) |
| **ConfigMap** | 1 | Config del túnel (ingress, hostnames) |

**Resumen poslite-cloudflared-core:** 1 Deployment, 1 ConfigMap. No Secret ni PVC en el chart.

---

## 4. Chart poslite-pam (solo tiendas PAM)

| Recurso | Cantidad | Descripción |
|--------|----------|-------------|
| **Deployment** | 8 principales | Portal, Guard API, TCP Connector, Scraper, Playwright, Core WebAPI, Core WebEvents, Workers (varios en un solo template) |
| **Deployment** | 16 workers | Dentro de `deployment-workers.yaml`: price, shift, errorReports, dispatches, shifts, totalizers, tankstock, loyaltyConfig, ierp (varios) |
| **Service** | 6 | TCP Connector, Scraper, Core WebAPI, Core WebEvents, Guard API, Playwright (uno por Deployment que expone puerto) |
| **ConfigMap** | 1 | Configuración |
| **Secret** | 1 | Credenciales |
| **PersistentVolumeClaim** | 1 | Uploads |
| **PersistentVolumeClaim** | 1 | Licenses |

**Resumen poslite-pam:** 24 Deployments (8 + 16), 6 Services, 1 ConfigMap, 1 Secret, 2 PVCs (uploads, licenses).

---

## 5. Chart poslite-cloudflared-pam (solo tiendas PAM)

| Recurso | Cantidad | Descripción |
|--------|----------|-------------|
| **Deployment** | 1 | Cloudflared |
| **ConfigMap** | 1 | Config del túnel |

**Resumen poslite-cloudflared-pam:** 1 Deployment, 1 ConfigMap.

---

## 6. Chart poslite-horustech (solo tiendas Horustech)

| Recurso | Cantidad | Descripción |
|--------|----------|-------------|
| **Deployment** | 7 principales | Portal, WebAPI, Guard API, Core WebAPI, Core WebEvents, Core Loyalty Config Worker, Workers (varios en un solo template) |
| **Deployment** | 12 workers | Dentro de `deployment-workers.yaml` (price, shift, errorReports, dispatches, tanks, ierp: varios) |
| **Service** | 5 | WebAPI, Guard API, Core WebAPI, Core WebEvents (uno por Deployment que expone puerto) |
| **ConfigMap** | 1 | Configuración |
| **Secret** | 1 | Credenciales |
| **PersistentVolumeClaim** | 1 | Uploads |
| **PersistentVolumeClaim** | 1 | Licenses |
| **PersistentVolumeClaim** | 1 | Pointer |

**Resumen poslite-horustech:** 19 Deployments (7 + 12), 5 Services, 1 ConfigMap, 1 Secret, 3 PVCs (uploads, licenses, pointer).

---

## 7. Chart poslite-cloudflared-horustech (solo tiendas Horustech)

| Recurso | Cantidad | Descripción |
|--------|----------|-------------|
| **Deployment** | 1 | Cloudflared |
| **ConfigMap** | 1 | Config del túnel |

**Resumen poslite-cloudflared-horustech:** 1 Deployment, 1 ConfigMap.

---

# Resumen por tipo de tienda (qué se crea en Rancher)

## Tienda solo Core (db + core + cloudflared-core)

| Tipo de objeto | Cantidad total |
|----------------|----------------|
| **StatefulSet** | 1 (Postgres) |
| **Deployments** | 1 (PgAdmin) + 13 (Core) + 1 (Cloudflared) = **15** |
| **Services** | **1** (WebAPI) |
| **ConfigMaps** | 1 + 1 + 1 = **3** |
| **Secrets** | 1 + 1 = **2** |
| **PVCs** | 1 (pgadmin) + 1 (postgres data, por volumeClaimTemplates) + 1 (uploads) = **3** (más el que crea el StatefulSet para datos Postgres) |

---

## Tienda PAM (db + cloudflared-pam + pam)

| Tipo de objeto | Cantidad total |
|----------------|----------------|
| **StatefulSet** | 1 |
| **Deployments** | 1 (PgAdmin) + 1 (Cloudflared) + 24 (PAM) = **26** |
| **Services** | **6** |
| **ConfigMaps** | **3** |
| **Secrets** | **2** |
| **PVCs** | 1 (pgadmin) + 1 (postgres data) + 2 (uploads, licenses) = **4** (+ el del StatefulSet) |

---

## Tienda Horustech (db + cloudflared-horustech + horustech)

| Tipo de objeto | Cantidad total |
|----------------|----------------|
| **StatefulSet** | 1 |
| **Deployments** | 1 (PgAdmin) + 1 (Cloudflared) + 19 (Horustech) = **21** |
| **Services** | **5** |
| **ConfigMaps** | **3** |
| **Secrets** | **2** |
| **PVCs** | 1 (pgadmin) + 1 (postgres data) + 3 (uploads, licenses, pointer) = **5** (+ el del StatefulSet) |

---

# Resumen final

- **Deployments:** muchos (portal, webapi, workers, cloudflared, pgadmin, etc.), según el tipo de tienda (Core ~15, PAM ~26, Horustech ~21).
- **StatefulSet:** 1 por tienda (PostgreSQL en poslite-db).
- **Services:** los que definen los templates (Core 1, PAM 6, Horustech 5). El StatefulSet de Postgres usa `serviceName` pero el chart no incluye template de Service para él.
- **PVCs:** creados por el **StatefulSet** (volumeClaimTemplates) para datos de Postgres; más los **explícitos** en los charts: PgAdmin, uploads, licenses (PAM/Horustech), pointer (Horustech).
- **Secrets:** 1 en poslite-db (postgres + pgadmin), 1 en core/pam/horustech (credenciales de aplicación).
- **ConfigMaps:** 1 por chart (db, core/pam/horustech, cloudflared-*).

Los nombres concretos en Rancher llevan el **release name** que Fleet asigna a cada target (p. ej. derivado del nombre del bundle y del target: db, core, cloudflared-core, etc.), todos en el namespace **`poslite`**.
