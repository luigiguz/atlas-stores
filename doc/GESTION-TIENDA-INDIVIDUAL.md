# Gestión de una Tienda Individual

Esta guía explica cómo instalar o actualizar **solo una tienda específica** sin afectar a las demás.

---

## 🎯 Concepto Clave

Fleet usa **labels en los clusters** para determinar qué desplegar. Cada tienda tiene su propio cluster con labels únicos, por lo que puedes trabajar con una tienda sin afectar a las demás.

---

## 📝 Actualizar una Tienda Existente

### Paso 1: Editar solo los archivos de esa tienda

```bash
# Navegar a la carpeta de la tienda
cd stores/<nombre-tienda>/

# Editar los archivos que necesites
vim values-common.yaml      # Configuración común
vim values-db.yaml          # Configuración de BD
vim values-horustech.yaml   # O values-pam.yaml
vim secrets.yaml            # Secretos
```

### Paso 2: Commit y Push

```bash
# Solo agregar los archivos de esa tienda
git add stores/<nombre-tienda>/

# Commit
git commit -m "Actualizar configuración de <nombre-tienda>"

# Push
git push origin <rama>
```

### Paso 3: Fleet aplica automáticamente

Fleet detectará los cambios y aplicará **solo a esa tienda** porque:
- Los valores específicos de la tienda (`stores/<nombre-tienda>/*.yaml`) tienen mayor precedencia
- Solo el cluster con el label `store: "<nombre-tienda>"` recibirá esos valores

**Tiempo estimado**: 2-5 minutos

---

## 🆕 Instalar una Nueva Tienda

### Paso 1: Crear configuración en Git

```bash
# Crear directorio de la tienda
mkdir -p stores/<nombre-tienda>

# Copiar templates
cp stores/ejemplo-tienda/values-*.yaml stores/<nombre-tienda>/
cp stores/ejemplo-tienda/secrets.sops.yaml.example stores/<nombre-tienda>/secrets.yaml

# Editar configuración
vim stores/<nombre-tienda>/values-common.yaml
vim stores/<nombre-tienda>/values-db.yaml
vim stores/<nombre-tienda>/values-horustech.yaml  # o values-pam.yaml
vim stores/<nombre-tienda>/secrets.yaml
```

### Paso 2: Commit y Push

```bash
git add stores/<nombre-tienda>/
git commit -m "Agregar nueva tienda: <nombre-tienda>"
git push origin <rama>
```

### Paso 3: Configurar el Cluster en Rancher

1. **Registrar el cluster** en Rancher (si aún no está registrado)
2. **Aplicar labels al cluster**:

```yaml
Labels del cluster:
  atlas: "true"
  store: "<nombre-tienda>"        # ← IMPORTANTE: Nombre único de la tienda
  poslite: "horustech"            # o "pam"
  wave: "pilot"                   # o "wave1", "wave2"
```

**Cómo aplicar labels en Rancher:**
1. Ve al cluster
2. Click en **"⋮" (menú)** → **"Edit Config"**
3. En la sección **"Labels"**, agrega los labels arriba
4. Guardar

### Paso 4: Fleet despliega automáticamente

Una vez que el cluster tenga los labels correctos:
- Fleet detectará que el cluster coincide con el `clusterSelector` del grupo (ej: `wave: "pilot"`)
- Fleet aplicará los bundles del grupo (db, core, cloudflared)
- Fleet combinará con los valores específicos de `stores/<nombre-tienda>/*.yaml`
- Solo ese cluster recibirá la configuración de esa tienda

---

## 🔍 Cómo Funciona el Sistema de Labels

### Labels del Cluster

Cada cluster tiene labels que determinan:
- **Qué grupo de bundles aplicar** (`wave: "pilot"` → aplica bundles de `groups/pilot.yaml`)
- **Qué configuración de tienda usar** (`store: "atlasposlitepilot"` → usa `stores/atlasposlitepilot/*.yaml`)
- **Qué tipo de POS** (`poslite: "horustech"` → aplica bundles de horustech)

### Ejemplo: Tienda "atlasposlitepilot"

```yaml
# Labels del cluster
atlas: "true"
store: "atlasposlitepilot"
poslite: "horustech"
wave: "pilot"
```

**Fleet hace:**
1. Busca bundles que coincidan con `wave: "pilot"` → Encuentra `groups/pilot.yaml`
2. Aplica bundles: `pilot-stores`, `pilot-core`, `pilot-cloudflared`
3. Busca configuración específica con `store: "atlasposlitepilot"` → Encuentra `stores/atlasposlitepilot/*.yaml`
4. Combina valores: chart defaults + grupo pilot + tienda atlasposlitepilot
5. Despliega solo en ese cluster

---

## ✅ Verificar que Solo se Actualizó una Tienda

### En Rancher UI

1. Ve a **Fleet > Bundles**
2. Selecciona un bundle (ej: `pilot-stores`)
3. Verás **"Target Clusters"** → Debería mostrar solo los clusters del grupo `pilot`
4. Cada cluster tiene su propia configuración según `store: "<nombre>"`

### Con kubectl

```bash
# Ver qué clusters tienen el label store
kubectl get clusters.fleet.cattle.io -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.store}{"\n"}{end}'

# Ver configuración aplicada a un cluster específico
kubectl describe bundle.fleet.cattle.io <nombre-bundle> -n fleet-default | grep -A 20 "Target Clusters"
```

---

## 🎯 Casos de Uso Comunes

### Caso 1: Cambiar versión de imagen para una tienda

```bash
# Editar solo esa tienda
vim stores/<nombre-tienda>/values-common.yaml

# Cambiar imageTag
imageTag: stable  # o latest, unstable

# Commit y push
git add stores/<nombre-tienda>/values-common.yaml
git commit -m "Actualizar imagen a stable para <nombre-tienda>"
git push
```

**Resultado**: Solo esa tienda se actualiza, las demás siguen con su versión actual.

### Caso 2: Cambiar configuración de BD para una tienda

```bash
vim stores/<nombre-tienda>/values-db.yaml
# Editar configuración de PostgreSQL

git add stores/<nombre-tienda>/values-db.yaml
git commit -m "Actualizar BD para <nombre-tienda>"
git push
```

**Resultado**: Solo el StatefulSet de PostgreSQL de esa tienda se actualiza.

### Caso 3: Actualizar secretos de una tienda

```bash
vim stores/<nombre-tienda>/secrets.yaml
# Editar secretos

git add stores/<nombre-tienda>/secrets.yaml
git commit -m "Actualizar secretos para <nombre-tienda>"
git push
```

**Resultado**: Solo los Secrets de esa tienda se actualizan.

### Caso 4: Agregar nueva tienda sin afectar existentes

```bash
# Crear configuración
mkdir stores/nueva-tienda
cp stores/ejemplo-tienda/* stores/nueva-tienda/
# Editar archivos...

# Commit
git add stores/nueva-tienda/
git commit -m "Agregar nueva tienda: nueva-tienda"
git push

# En Rancher: aplicar labels al cluster nuevo
# store: "nueva-tienda", wave: "pilot", etc.
```

**Resultado**: La nueva tienda se despliega, las existentes no se tocan.

---

## ⚠️ Precauciones

### ❌ NO hacer esto:

1. **Editar `groups/*.yaml`** si solo quieres cambiar una tienda
   - Esto afectaría a TODAS las tiendas del grupo

2. **Cambiar labels del cluster incorrectamente**
   - Podría hacer que Fleet aplique configuración incorrecta

3. **Editar archivos de otras tiendas por error**
   - Siempre verifica que estás en la carpeta correcta

### ✅ SÍ hacer esto:

1. **Siempre trabajar en la carpeta específica de la tienda**
   ```bash
   cd stores/<nombre-tienda>/
   ```

2. **Verificar antes de commit**
   ```bash
   git status
   git diff stores/<nombre-tienda>/
   ```

3. **Usar nombres descriptivos en commits**
   ```bash
   git commit -m "Actualizar <componente> para tienda <nombre>"
   ```

---

## 🔄 Orden de Precedencia de Valores

Cuando Fleet despliega, combina valores en este orden (mayor a menor precedencia):

1. **Valores específicos de la tienda** (`stores/<tienda>/*.yaml`) ← **MAYOR PRECEDENCIA**
2. Valores del grupo (`groups/pilot.yaml`)
3. Valores del bundle en `atlas-apps`
4. Valores por defecto del chart

**Esto significa**: Los valores en `stores/<tienda>/*.yaml` siempre sobrescriben los del grupo.

---

## 📊 Ejemplo Completo: Actualizar Solo "atlasposlitepilot"

```bash
# 1. Editar configuración
cd stores/atlasposlitepilot/
vim values-horustech.yaml

# Cambiar algo, por ejemplo:
# imageTag: latest → imageTag: stable

# 2. Verificar cambios
git diff stores/atlasposlitepilot/values-horustech.yaml

# 3. Commit
git add stores/atlasposlitepilot/values-horustech.yaml
git commit -m "Actualizar imagen a stable para atlasposlitepilot"

# 4. Push
git push origin develop

# 5. Verificar en Rancher (2-5 minutos después)
# - Fleet > Bundles > pilot-horustech
# - Ver que solo el cluster "atlasposlitepilot" se actualiza
# - Ver pods reiniciándose con la nueva imagen
```

---

## 🆘 Troubleshooting

### Problema: Los cambios se aplican a todas las tiendas

**Causa**: Probablemente editaste `groups/*.yaml` en lugar de `stores/<tienda>/*.yaml`

**Solución**: 
- Verifica qué archivos editaste: `git status`
- Si editaste un grupo, revierte: `git checkout groups/pilot.yaml`
- Edita solo los archivos de la tienda específica

### Problema: Los cambios no se aplican a ninguna tienda

**Causa**: El cluster no tiene el label `store: "<nombre-tienda>"` correcto

**Solución**:
1. Verificar labels del cluster en Rancher
2. Verificar que el nombre en `stores/<nombre-tienda>/` coincida con `store: "<nombre-tienda>"` en los labels

### Problema: Fleet no detecta los cambios

**Solución**:
```bash
# Forzar actualización del repositorio
kubectl annotate gitrepo.fleet.cattle.io atlas-stores -n fleet-default fleet.cattle.io/force-update="$(date +%s)"
```

---

## 📝 Resumen

- ✅ **Para actualizar una tienda**: Edita solo `stores/<nombre-tienda>/*.yaml` y haz push
- ✅ **Para instalar nueva tienda**: Crea `stores/<nombre-tienda>/` y aplica labels al cluster
- ✅ **Fleet aplica automáticamente** solo a la tienda correcta según el label `store`
- ✅ **No afecta otras tiendas** porque cada una tiene su propia configuración
