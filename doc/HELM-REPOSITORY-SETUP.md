# Configuración de Helm Repository para Fleet

## 📋 Cambios Realizados

### ❌ Antes (No Funcionaba)
```yaml
helm:
  chart: git://atlas-apps/fleet/bundles/db  # ❌ Fleet no soporta referencias Git entre repos
```

### ✅ Después (Funcional)
```yaml
helm:
  repo: https://aspposlite.azurecr.io/helm  # ✅ Helm Repository
  chart: poslite-db                          # ✅ Nombre del chart
  version: 1.0.0                             # ✅ Versión específica
  values:                                    # ✅ Valores por tienda
    postgresql:
      timezone: "America/Panama"
```

## 🔄 Explicación del Cambio

**Problema Original:**
- Fleet **NO puede** referenciar charts de otro GitRepo usando `git://`
- Las rutas relativas `../../atlas-apps/` tampoco funcionan entre repositorios Git separados
- Fleet procesa cada GitRepo de forma independiente

**Solución Implementada:**
- Los charts de `atlas-apps` se publican a un **Helm Repository** (Azure Container Registry OCI)
- `atlas-stores` referencia los charts usando `helm.repo` + `helm.chart` + `helm.version`
- Fleet puede descargar charts desde Helm Repositories estándar

**Ventajas:**
- ✅ Separación clara de responsabilidades
- ✅ Versionado explícito de charts
- ✅ Compatible con estándares de Helm
- ✅ Permite actualizaciones controladas por versión

---

## 🏗️ Estructura Correcta de Repositorios

### Repositorio: `atlas-apps`
**Propósito:** Contiene y publica Helm Charts

```
atlas-apps/
├── charts/
│   ├── poslite-db/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   ├── poslite-core/
│   ├── poslite-horustech/
│   ├── poslite-pam/
│   └── poslite-cloudflared/
├── .github/workflows/
│   └── publish-charts.yml  # CI/CD para publicar charts
└── README.md
```

**Responsabilidades:**
- Desarrollar y mantener Helm Charts
- Publicar charts a Helm Repository (ACR OCI)
- Versionar charts (semver)

### Repositorio: `atlas-stores`
**Propósito:** Configuración por tienda usando Fleet

```
atlas-stores/
├── groups/
│   ├── pilot/
│   │   └── fleet.yaml      # ✅ Usa helm.repo + chart + version
│   ├── wave1/
│   │   └── fleet.yaml
│   └── wave2/
│       └── fleet.yaml
├── stores/
│   └── <tienda>/
│       ├── values-common.yaml
│       ├── values-db.yaml
│       ├── values-horustech.yaml
│       └── secrets.yaml
└── README.md
```

**Responsabilidades:**
- Configuración específica por tienda
- Valores de Helm por tienda
- Gestión de secretos
- Selectores de clusters (labels)

---

## ✅ Checklist de Verificación en Rancher/Fleet

### 1. Verificar Helm Repository

```bash
# Verificar que el Helm Repository es accesible
helm repo add atlas-apps https://aspposlite.azurecr.io/helm
helm repo update
helm search repo atlas-apps

# Deberías ver:
# atlas-apps/poslite-db          1.0.0
# atlas-apps/poslite-core        1.0.0
# atlas-apps/poslite-cloudflared 1.0.0
```

**En Rancher:**
- [ ] Helm Repository es accesible desde los clusters
- [ ] Autenticación configurada (si es privado)
- [ ] Charts están publicados y visibles

### 2. Verificar GitRepo `atlas-stores`

```bash
# Ver estado del GitRepo
kubectl get gitrepos.fleet.cattle.io atlas-stores -n fleet-default

# Ver detalles
kubectl describe gitrepo.fleet.cattle.io atlas-stores -n fleet-default
```

**Verificar:**
- [ ] Estado: `Ready` o `Active` (no `Error` o `Stalled`)
- [ ] Último commit detectado correctamente
- [ ] No hay errores en "Conditions"

### 3. Verificar Bundles Creados

```bash
# Ver todos los bundles
kubectl get bundles.fleet.cattle.io -n fleet-default

# Ver detalles de un bundle
kubectl describe bundle.fleet.cattle.io pilot-stores -n fleet-default
```

**Verificar:**
- [ ] Bundles aparecen: `pilot-stores`, `pilot-core`, `pilot-cloudflared`
- [ ] Estado: `Ready` (no `Pending` o `Error`)
- [ ] "Target Clusters" muestra los clusters correctos
- [ ] No hay errores en "Status" o "Conditions"

### 4. Verificar Despliegue en Clusters

```bash
# Ver pods desplegados
kubectl get pods -n poslite

# Ver deployments
kubectl get deployments -n poslite

# Ver statefulsets
kubectl get statefulsets -n poslite
```

**Verificar:**
- [ ] Pods en estado `Running` (no `Pending` o `Error`)
- [ ] Deployments con réplicas deseadas = disponibles
- [ ] StatefulSet de PostgreSQL corriendo
- [ ] No hay errores en eventos: `kubectl get events -n poslite`

### 5. Verificar Labels de Clusters

```bash
# Ver labels de los clusters
kubectl get clusters.fleet.cattle.io -A -o yaml | grep -A 10 "labels:"
```

**Verificar que los clusters tengan:**
- [ ] `atlas: "true"`
- [ ] `wave: "pilot"` (o wave1, wave2)
- [ ] `store: "<nombre-tienda>"`
- [ ] `poslite: "horustech"` o `"pam"`

### 6. Verificar Logs de Fleet

```bash
# Logs del Fleet Controller
kubectl logs -n fleet-system -l app=fleet-controller --tail=50

# Buscar errores
kubectl logs -n fleet-system -l app=fleet-controller --tail=100 | grep -i error
```

**Verificar:**
- [ ] No hay errores relacionados con `git://`
- [ ] No hay errores de "chart not found"
- [ ] Bundles se procesan correctamente

---

## 🔧 Configuración del Helm Repository

### Opción 1: Azure Container Registry (OCI) - Recomendado

Si ya usas `aspposlite.azurecr.io`, puedes usar ACR como Helm Repository OCI:

```bash
# Login a ACR
az acr login --name aspposlite

# Publicar chart
helm package charts/poslite-db
helm push poslite-db-1.0.0.tgz oci://aspposlite.azurecr.io/helm
```

**URL del Repo:** `https://aspposlite.azurecr.io/helm`

### Opción 2: Helm Chart Repository (HTTP/HTTPS)

Si prefieres un repositorio HTTP tradicional:

```bash
# Usar ChartMuseum, Harbor, o similar
helm repo add atlas-apps https://charts.tu-dominio.com
```

**URL del Repo:** `https://charts.tu-dominio.com`

### Opción 3: GitHub Pages (Público)

Para repositorios públicos:

```bash
# Publicar charts a GitHub Pages
helm package charts/poslite-db
# Subir a gh-pages branch
```

**URL del Repo:** `https://<usuario>.github.io/<repo>/charts`

---

## 🚀 Pipeline CI/CD Sugerido

### GitHub Actions para `atlas-apps`

Crea `.github/workflows/publish-charts.yml`:

```yaml
name: Publish Helm Charts

on:
  push:
    branches: [main]
    paths:
      - 'charts/**'
  release:
    types: [created]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.12.0'
      
      - name: Login to ACR
        uses: azure/docker-login@v1
        with:
          login-server: aspposlite.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      
      - name: Package and Push Charts
        run: |
          for chart in charts/*/; do
            chart_name=$(basename "$chart")
            helm package "$chart"
            helm push "${chart_name}-*.tgz" oci://aspposlite.azurecr.io/helm
          done
      
      - name: Update Chart Versions
        if: github.event_name == 'release'
        run: |
          # Actualizar versiones en Chart.yaml basado en tag
          VERSION=${GITHUB_REF#refs/tags/v}
          # Lógica para actualizar versiones
```

### Variables de Entorno Necesarias

En GitHub Secrets:
- `ACR_USERNAME`: Usuario de Azure Container Registry
- `ACR_PASSWORD`: Password o token de ACR

---

## 📝 Notas Importantes

### Versiones de Charts

**Opción A: Versión Fija (Recomendado para Producción)**
```yaml
helm:
  version: 1.0.0  # Versión específica, no cambia automáticamente
```

**Opción B: Última Versión**
```yaml
helm:
  version: ">0.0.0"  # Siempre la última versión (menos control)
```

**Opción C: Rango de Versiones**
```yaml
helm:
  version: "^1.0.0"  # Compatible con 1.x.x
```

### Autenticación del Helm Repository

Si el Helm Repository es privado, necesitas configurar autenticación en Fleet:

```yaml
# En el GitRepo o Bundle, agregar:
helm:
  repo: https://aspposlite.azurecr.io/helm
  chart: poslite-db
  version: 1.0.0
  repoCAFile: ""  # Si usa certificado custom
  repoCredentialSecretName: "helm-repo-credentials"  # Secret con credenciales
```

Crear el Secret:
```bash
kubectl create secret generic helm-repo-credentials \
  --from-literal=username=<user> \
  --from-literal=password=<pass> \
  -n fleet-default
```

---

## 🔍 Troubleshooting

### Error: "chart not found"
**Causa:** Chart no está publicado en el Helm Repository
**Solución:** Verificar que el chart esté publicado: `helm search repo atlas-apps/poslite-db`

### Error: "unauthorized" o "authentication required"
**Causa:** Helm Repository requiere autenticación
**Solución:** Configurar `repoCredentialSecretName` en el fleet.yaml

### Error: "no such host" o "connection refused"
**Causa:** Helm Repository no es accesible desde los clusters
**Solución:** Verificar conectividad de red y URL del repositorio

### Bundles no se crean
**Causa:** Errores en el procesamiento del fleet.yaml
**Solución:** Verificar logs de Fleet: `kubectl logs -n fleet-system -l app=fleet-controller`

---

## ✅ Resumen

- ✅ **Eliminadas** todas las referencias `git://atlas-apps/`
- ✅ **Implementado** uso de `helm.repo` + `helm.chart` + `helm.version`
- ✅ **Mantenidos** todos los `values` específicos por tienda
- ✅ **Preservados** `targetCustomizations` y `clusterSelector`
- ✅ **Separación clara** entre `atlas-apps` (charts) y `atlas-stores` (config)

**Próximos Pasos:**
1. Publicar charts de `atlas-apps` al Helm Repository
2. Verificar que Fleet puede acceder al repositorio
3. Hacer commit y push de los cambios
4. Verificar despliegue en Rancher/Fleet
