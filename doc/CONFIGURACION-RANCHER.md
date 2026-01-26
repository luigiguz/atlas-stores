# Guía de Configuración en Rancher - PosLite RKE2

Esta guía te llevará paso a paso para configurar Rancher, Fleet y los repositorios GitOps para desplegar PosLite en clusters RKE2.

## 📋 Índice

1. [Prerrequisitos](#prerrequisitos)
2. [Configurar Repositorios Git en Fleet](#configurar-repositorios-git-en-fleet)
3. [Registrar Clusters RKE2](#registrar-clusters-rke2)
4. [Aplicar Labels a Clusters](#aplicar-labels-a-clusters)
5. [Configurar SOPS para Secretos](#configurar-sops-para-secretos)
6. [Verificar Despliegue](#verificar-despliegue)
7. [Troubleshooting](#troubleshooting)

---

## 🔧 Prerrequisitos

### 1. Acceso a Rancher

- Acceso administrativo a Rancher (Atlas Core)
- Permisos para crear repositorios Git en Fleet
- Permisos para registrar clusters

### 2. Repositorios Git Preparados

Asegúrate de que los repositorios estén disponibles y accesibles:

- **`atlas-apps`**: Repositorio con Helm charts y Fleet bundles
- **`atlas-stores`**: Repositorio con configuración por tienda

**Requisitos:**
- Repositorios en Git (GitHub, GitLab, Bitbucket, etc.)
- Acceso desde Rancher (URL pública o credenciales)
- Branch principal: `main` o `master`

### 3. Clusters RKE2 Preparados

- Clusters RKE2 instalados y funcionando
- Clusters registrados o listos para registrar en Rancher
- Acceso de red desde Rancher a los clusters

### 4. Labels Requeridos por Cluster

**IMPORTANTE:** Cada cluster RKE2 debe tener las siguientes labels configuradas para que Fleet pueda desplegar los bundles correctos. Estas labels son **obligatorias** y deben aplicarse antes de que Fleet intente desplegar.

#### Labels Obligatorias para Todos los Clusters

```yaml
atlas: "true"              # Label base que identifica clusters de Atlas
```

#### Labels Específicas por Tienda

Cada cluster debe tener además:

```yaml
store: "<nombre-tienda>"    # Nombre único de la tienda (ej: "tienda-1", "gasolinera-central")
poslite: "pam"             # o "horustech" - Define qué stack se despliega
wave: "pilot"              # o "wave1", "wave2" - Define la fase de migración
```

#### Ejemplos de Configuración de Labels

**Cluster con PAM:**
```yaml
atlas: "true"
store: "tienda-1"
poslite: "pam"
wave: "pilot"
```

**Cluster con Horustech:**
```yaml
atlas: "true"
store: "tienda-2"
poslite: "horustech"
wave: "wave1"
```

#### Cómo Fleet Usa las Labels

- **`atlas: "true"`**: Requerido por todos los bundles (db, core, cloudflared). Identifica que el cluster pertenece a Atlas.
- **`poslite: "pam"`**: Bundle `pam` se despliega solo en clusters con este label. Define qué stack de aplicación se ejecuta.
- **`poslite: "horustech"`**: Bundle `horustech` se despliega solo en clusters con este label. Define qué stack de aplicación se ejecuta.
- **`store`**: Identificador único de la tienda (usado para configuración específica y tracking).
- **`wave`**: Usado para organizar tiendas en fases de migración. **Es opcional** pero muy recomendado para gestión gradual.

#### ¿Qué es la Label `wave` y Cómo Funciona?

La label `wave` permite organizar la migración en **fases controladas**. No es estrictamente necesaria para que los bundles se desplieguen (los bundles principales solo requieren `atlas: "true"` y `poslite: "pam"`/`"horustech"`), pero es muy útil para:

**1. Organización de Fases de Migración:**
```
pilot  → Tiendas piloto (pruebas iniciales, validación)
wave1  → Primera ola de migración (tiendas seleccionadas)
wave2  → Segunda ola de migración (expansión)
```

**2. Aplicar Configuraciones por Fase:**
Los archivos en `atlas-stores/groups/` (pilot.yaml, wave1.yaml, wave2.yaml) pueden usar `wave` para aplicar configuraciones específicas a cada fase:

```yaml
# groups/pilot.yaml
targetCustomizations:
- name: pilot-stores
  clusterSelector:
    matchLabels:
      atlas: "true"
      wave: "pilot"    # ← Solo clusters con wave=pilot
  helm:
    chart: ../../atlas-apps/fleet/bundles/db
    values:
      # Configuración específica para pilot
```

**3. Control de Ritmo de Migración:**
- Permite migrar tiendas en grupos controlados
- Facilita el rollback si hay problemas en una fase
- Permite aplicar configuraciones diferentes por fase

**4. Tracking y Monitoreo:**
- Identificar fácilmente en qué fase está cada tienda
- Filtrar clusters por fase en Rancher
- Generar reportes por fase

**Ejemplo de Uso:**

```yaml
# Cluster en fase piloto
atlas: "true"
store: "tienda-piloto-1"
poslite: "pam"
wave: "pilot"    # ← Identifica que está en fase piloto

# Después de validar, mover a wave1
wave: "wave1"    # ← Cambiar label para mover a siguiente fase
```

**Nota Importante:**
- Los bundles principales (`db`, `core`, `pam`, `horustech`, `cloudflared`) **NO requieren** la label `wave` para funcionar.
- La label `wave` es principalmente para **organización y gestión de fases**.
- Si usas los archivos de grupos (`groups/pilot.yaml`, etc.), entonces `wave` sí es necesaria para esos grupos específicos.

**Nota:** Las labels deben aplicarse **antes** de que Fleet intente desplegar. Si un cluster no tiene las labels correctas, los bundles no se aplicarán automáticamente.

### 5. SOPS (Opcional pero Recomendado)

Para gestionar secretos encriptados:

```bash
# Instalar SOPS
# Linux
wget https://github.com/mozilla/sops/releases/download/v3.8.0/sops-v3.8.0.linux
sudo mv sops-v3.8.0.linux /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# macOS
brew install sops

# Windows (con Chocolatey)
choco install sops
```

---

## 📦 Configurar Repositorios Git en Fleet

### Paso 1: Acceder a Fleet en Rancher

1. Inicia sesión en Rancher
2. En el menú principal, busca **"Fleet"** o **"GitOps"**
3. Si no ves Fleet, verifica que esté habilitado en tu instalación de Rancher

### Paso 2: Crear Repositorio `atlas-apps`

1. Ve a **Fleet > Git Repos**
2. Haz clic en **"Create"** o **"Add Repository"**
3. Completa el formulario:

**Configuración Básica:**
```
Name: atlas-apps
Repository URL: https://github.com/tu-org/atlas-apps.git
Branch: main
```

**O si es privado:**
```
Name: atlas-apps
Repository URL: https://github.com/tu-org/atlas-apps.git
Branch: main
Authentication:
  - Username: tu-usuario
  - Password: tu-token-o-password
```

**O con SSH:**
```
Name: atlas-apps
Repository URL: git@github.com:tu-org/atlas-apps.git
Branch: main
Authentication:
  - SSH Private Key: [pegar tu clave privada SSH]
```

4. Haz clic en **"Create"**

### Paso 3: Crear Repositorio `atlas-stores`

Repite el proceso anterior para el repositorio `atlas-stores`:

```
Name: atlas-stores
Repository URL: https://github.com/tu-org/atlas-stores.git
Branch: main
```

### Paso 4: Verificar Repositorios

1. En **Fleet > Git Repos**, deberías ver ambos repositorios
2. Verifica que el estado sea **"Active"** o **"Ready"**
3. Si hay errores, revisa los logs haciendo clic en el repositorio

**Troubleshooting:**
- Si el estado es "Error", verifica:
  - URL del repositorio
  - Credenciales (si es privado)
  - Acceso de red desde Rancher al repositorio

---

## 🖥️ Registrar Clusters RKE2

### Opción A: Cluster Ya Instalado (Importar)

Si ya tienes un cluster RKE2 funcionando:

1. En Rancher, ve a **"Clusters"**
2. Haz clic en **"Import Existing"** o **"Import"**
3. Sigue las instrucciones para:
   - Generar token de importación
   - Ejecutar comando en el cluster RKE2
4. El cluster aparecerá en Rancher una vez importado

**Comando típico en el cluster RKE2:**
```bash
curl --insecure -fL https://rancher.tu-dominio.com/v3-public/import/XXXXX.yaml | kubectl apply -f -
```

### Opción B: Crear Cluster desde Rancher

Si quieres que Rancher cree el cluster:

1. En Rancher, ve a **"Clusters"**
2. Haz clic en **"Create"**
3. Selecciona **"RKE2"** como tipo de cluster
4. Completa la configuración:
   - Nombre del cluster (ej: `rpi-tienda-1`)
   - Configuración de red
   - Nodos (en este caso, 1 nodo = Raspberry Pi)
5. Sigue las instrucciones para instalar el agente en el Raspberry Pi

### Paso 5: Verificar Cluster Registrado

1. En **Clusters**, verifica que el cluster aparezca
2. Estado debe ser **"Active"**
3. Haz clic en el cluster para ver detalles

---

## 🏷️ Aplicar Labels a Clusters

Los labels son **críticos** porque Fleet los usa para decidir qué desplegar dónde.

> **📌 Nota:** Para ver la lista completa de labels requeridas, consulta la sección [Labels Requeridos por Cluster](#4-labels-requeridos-por-cluster) en los Prerrequisitos.

### Paso 1: Acceder a Configuración del Cluster

1. En **Clusters**, haz clic en el cluster (ej: `rpi-tienda-1`)
2. Haz clic en el menú de tres puntos (⋮) o en **"Edit Config"**
3. Busca la sección **"Labels"** o **"Annotations"**

### Paso 2: Agregar Labels Obligatorios

Agrega los siguientes labels:

```yaml
atlas: "true"
store: "tienda-1"              # Nombre de la tienda
poslite: "pam"                 # o "horustech"
wave: "pilot"                  # o "wave1", "wave2"
```

**Ejemplo visual en Rancher UI:**
```
Labels:
  atlas = true
  store = tienda-1
  poslite = pam
  wave = pilot
```

### Paso 3: Guardar Cambios

1. Haz clic en **"Save"** o **"Update"**
2. Espera a que se apliquen los cambios

### Paso 4: Verificar Labels

Puedes verificar los labels con `kubectl`:

```bash
# Desde el cluster o desde Rancher Shell
kubectl get clusters.fleet.cattle.io -n fleet-default -o yaml
```

O desde la UI de Rancher:
1. Ve al cluster
2. En **"Fleet"** o **"GitOps"**, deberías ver los bundles aplicados

---

## 🔐 Configurar SOPS para Secretos (Opcional)

### Paso 1: Generar Clave PGP (si no tienes una)

```bash
# Generar nueva clave PGP
gpg --full-generate-key

# Seleccionar:
# - Tipo: RSA and RSA
# - Tamaño: 4096
# - Expiración: 0 (sin expiración)
# - Nombre y email

# Listar claves para obtener fingerprint
gpg --list-keys
```

### Paso 2: Configurar SOPS en el Repositorio

En el repositorio `atlas-stores`, crea `.sops.yaml` en la raíz:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: stores/.*/secrets\.sops\.yaml$
    pgp: >-
      FINGERPRINT_DE_TU_CLAVE_PGP
```

**Ejemplo:**
```yaml
creation_rules:
  - path_regex: stores/.*/secrets\.sops\.yaml$
    pgp: >-
      A1B2C3D4E5F6G7H8I9J0K1L2M3N4O5P6Q7R8S9T0
```

### Paso 3: Encriptar Secretos

```bash
# Navegar al repositorio
cd atlas-stores

# Editar secretos (se desencripta automáticamente)
sops stores/tienda-1/secrets.sops.yaml

# O crear desde cero
cp stores/ejemplo-tienda/secrets.sops.yaml.example stores/tienda-1/secrets.sops.yaml
sops stores/tienda-1/secrets.sops.yaml

# Guardar (se encripta automáticamente al guardar)
```

### Paso 4: Configurar SOPS en Rancher/Fleet

Para que Fleet pueda desencriptar los secretos, necesitas:

**Opción A: Usar Sealed Secrets (Recomendado para producción)**

1. Instalar Sealed Secrets en el cluster
2. Usar `kubeseal` para encriptar secretos
3. Fleet aplica los Sealed Secrets
4. El controller de Sealed Secrets los desencripta

**Opción B: Usar External Secrets Operator**

1. Instalar External Secrets Operator
2. Configurar proveedor (AWS Secrets Manager, HashiCorp Vault, etc.)
3. Fleet crea ExternalSecret resources
4. El operator sincroniza desde el proveedor

**Opción C: Desencriptar Manualmente y Aplicar**

Si usas SOPS directamente:

1. Desencriptar secretos localmente
2. Aplicar manualmente con `kubectl`:

```bash
# Desencriptar
sops -d stores/tienda-1/secrets.sops.yaml > secrets.yaml

# Aplicar
kubectl apply -f secrets.yaml -n poslite
```

**Nota:** Esta opción no es GitOps puro, pero funciona para empezar.

---

## ✅ Verificar Despliegue

### Paso 1: Verificar Bundles en Fleet

1. En Rancher, ve a **Fleet > Bundles**
2. Deberías ver bundles como:
   - `db`
   - `core`
   - `pam` (o `horustech`)
   - `cloudflared`

3. Haz clic en un bundle para ver:
   - Clusters donde se aplica
   - Estado del despliegue
   - Errores (si los hay)

### Paso 2: Verificar Recursos en el Cluster

**Desde Rancher UI:**
1. Ve al cluster
2. Ve a **"Workloads"** o **"Deployments"**
3. Filtra por namespace `poslite`
4. Deberías ver:
   - Deployments (portal, webapi, workers, etc.)
   - StatefulSets (postgres)
   - Services
   - ConfigMaps
   - Secrets

**Desde kubectl:**
```bash
# Conectarse al cluster
kubectl get pods -n poslite

# Ver deployments
kubectl get deployments -n poslite

# Ver servicios
kubectl get services -n poslite

# Ver configmaps
kubectl get configmaps -n poslite

# Ver secrets
kubectl get secrets -n poslite
```

### Paso 3: Verificar Puertos

```bash
# Verificar que los pods tengan hostPort configurado
kubectl get pods -n poslite -o yaml | grep hostPort

# Verificar que los puertos estén escuchando en el host
netstat -tulpn | grep -E "7014|7012|5432|9010|9014"
```

### Paso 4: Verificar Logs

```bash
# Logs de un pod específico
kubectl logs -n poslite <nombre-pod>

# Logs de todos los pods de un deployment
kubectl logs -n poslite -l app=poslite-pam-portal

# Logs de Fleet controller
kubectl logs -n fleet-system -l app=fleet-controller
```

---

## 🔍 Troubleshooting

### Problema: Fleet no detecta el repositorio

**Síntomas:**
- Repositorio aparece con estado "Error"
- No se crean bundles

**Soluciones:**
1. Verificar URL del repositorio
2. Verificar credenciales (si es privado)
3. Verificar acceso de red desde Rancher
4. Revisar logs de Fleet:
   ```bash
   kubectl logs -n fleet-system -l app=fleet-controller
   ```

### Problema: Bundles no se aplican a clusters

**Síntomas:**
- Bundles existen pero no se despliegan
- Clusters no aparecen en "Target Clusters"

**Soluciones:**
1. Verificar labels del cluster:
   ```bash
   kubectl get clusters.fleet.cattle.io -n fleet-default -o yaml
   ```
2. Verificar `clusterSelector` en el bundle:
   - Debe coincidir con los labels del cluster
3. Verificar que el cluster esté en el namespace correcto de Fleet

### Problema: Pods no inician

**Síntomas:**
- Pods en estado "Pending" o "Error"
- Pods se reinician constantemente

**Soluciones:**
1. Verificar eventos:
   ```bash
   kubectl describe pod <nombre-pod> -n poslite
   ```
2. Verificar recursos (CPU/memoria):
   ```bash
   kubectl top pods -n poslite
   ```
3. Verificar imágenes:
   - ¿La imagen existe?
   - ¿Las credenciales de ACR son correctas?
4. Verificar PVCs:
   ```bash
   kubectl get pvc -n poslite
   ```

### Problema: Puertos no están expuestos

**Síntomas:**
- Servicios no accesibles desde el host
- Cloudflared no puede conectar

**Soluciones:**
1. Verificar `hostPort` en el deployment:
   ```bash
   kubectl get deployment <nombre> -n poslite -o yaml | grep hostPort
   ```
2. Verificar que el puerto no esté en uso:
   ```bash
   netstat -tulpn | grep <puerto>
   ```
3. Verificar permisos del pod (hostPort requiere privilegios en algunos casos)

### Problema: Secretos no se aplican

**Síntomas:**
- Pods fallan por falta de secretos
- Variables de entorno vacías

**Soluciones:**
1. Verificar que el Secret exista:
   ```bash
   kubectl get secrets -n poslite
   ```
2. Verificar que el Secret tenga las claves correctas:
   ```bash
   kubectl get secret <nombre-secret> -n poslite -o yaml
   ```
3. Si usas SOPS, verificar que esté desencriptado correctamente
4. Verificar que el deployment referencia el Secret correctamente

---

## 📊 Monitoreo y Gestión

### Ver Estado de Despliegue

**En Rancher UI:**
1. Ve a **Fleet > Bundles**
2. Selecciona un bundle
3. Verás:
   - Clusters donde se aplica
   - Estado de cada cluster
   - Errores (si los hay)

### Actualizar Configuración

1. Editar archivos en Git (values.yaml, etc.)
2. Hacer commit y push
3. Fleet detecta automáticamente
4. Fleet aplica cambios (rolling update)

### Rollback

**Opción 1: Git Revert**
```bash
git revert <commit-hash>
git push
# Fleet revierte automáticamente
```

**Opción 2: Desde Rancher**
1. Ve al bundle
2. Selecciona versión anterior
3. Aplica manualmente

### Agregar Nueva Tienda

1. **Crear configuración en Git:**
   ```bash
   cd atlas-stores
   mkdir -p stores/nueva-tienda
   cp stores/ejemplo-tienda/* stores/nueva-tienda/
   # Editar values-*.yaml
   # Crear secrets.sops.yaml
   git add stores/nueva-tienda/
   git commit -m "Agregar nueva tienda"
   git push
   ```

2. **Registrar cluster en Rancher:**
   - Seguir pasos de "Registrar Clusters RKE2"

3. **Aplicar labels:**
   ```yaml
   atlas: "true"
   store: "nueva-tienda"
   poslite: "pam"  # o "horustech"
   wave: "pilot"
   ```

4. **Fleet despliega automáticamente**

---

## 🎯 Checklist de Configuración

Usa este checklist para asegurarte de que todo esté configurado:

- [ ] Repositorio `atlas-apps` registrado en Fleet
- [ ] Repositorio `atlas-stores` registrado en Fleet
- [ ] Clusters RKE2 registrados en Rancher
- [ ] Labels aplicados a cada cluster:
  - [ ] `atlas: "true"`
  - [ ] `store: "<nombre>"`
  - [ ] `poslite: "pam"` o `"horustech"`
  - [ ] `wave: "pilot"` o `"wave1"` o `"wave2"`
- [ ] SOPS configurado (si se usan secretos encriptados)
- [ ] Bundles aparecen en Fleet
- [ ] Bundles se aplican a clusters correctos
- [ ] Pods están corriendo en namespace `poslite`
- [ ] Puertos expuestos correctamente (hostPort)
- [ ] Cloudflared conectado y funcionando
- [ ] Secretos aplicados correctamente
- [ ] Logs sin errores críticos

---

## 📚 Recursos Adicionales

### Documentación Oficial

- [Rancher Fleet Documentation](https://fleet.rancher.io/)
- [RKE2 Documentation](https://docs.rke2.io/)
- [SOPS Documentation](https://github.com/mozilla/sops)

### Comandos Útiles

```bash
# Ver todos los clusters de Fleet
kubectl get clusters.fleet.cattle.io -A

# Ver bundles
kubectl get bundles.fleet.cattle.io -A

# Ver gitrepos
kubectl get gitrepos.fleet.cattle.io -A

# Describir un bundle
kubectl describe bundle.fleet.cattle.io <nombre> -n fleet-default

# Ver logs de Fleet
kubectl logs -n fleet-system -l app=fleet-controller --tail=100
```

---

## 🆘 Soporte

Si encuentras problemas:

1. Revisar logs de Fleet
2. Verificar configuración de repositorios
3. Verificar labels de clusters
4. Consultar documentación de Rancher/Fleet
5. Contactar al equipo de DevOps

---

**¡Configuración completada!** 🎉

Ahora Fleet debería estar desplegando automáticamente PosLite en tus clusters RKE2 según la configuración en Git.
