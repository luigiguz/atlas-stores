# Cómo Verificar el Despliegue en Fleet

## 🚀 Después de hacer Push

Una vez que hagas `git push`, Fleet debería detectar los cambios automáticamente. Aquí te muestro cómo verificar que todo se está desplegando correctamente.

---

## 📊 Opción 1: Verificar en Rancher UI (Recomendado)

### Paso 1: Verificar que Fleet detectó el repositorio

1. Ve a **Rancher > Fleet > Git Repos**
2. Busca el repositorio `atlas-stores`
3. Verifica que el estado sea **"Active"** (verde) y no "Error"
4. Si hay error, haz clic para ver los detalles

### Paso 2: Verificar Bundles

1. Ve a **Fleet > Bundles**
2. Deberías ver bundles como:
   - `pilot-stores` (o `wave1-stores`, `wave2-stores`)
   - `pilot-core` (o `wave1-core`, `wave2-core`)
   - `pilot-cloudflared` (o `wave1-cloudflared`, `wave2-cloudflared`)
   - Y si aplica: bundles de `horustech` o `pam`

3. Haz clic en un bundle para ver:
   - **Target Clusters**: Clusters donde se aplica
   - **Status**: Estado (Ready, Pending, Error)
   - **Resources**: Recursos creados
   - **Errors**: Si hay errores, aparecerán aquí

### Paso 3: Verificar en el Cluster

1. Ve al cluster específico (ej: `atlasposlitepilot`)
2. Ve a **Workloads > Deployments** (o **StatefulSets**)
3. Filtra por namespace: `poslite`
4. Deberías ver:
   - ✅ Deployments corriendo (portal, webapi, workers, etc.)
   - ✅ StatefulSet de PostgreSQL
   - ✅ Todos con estado "Active" o "Running"

### Paso 4: Verificar Recursos

1. En el cluster, ve a **Service Discovery > Services**
2. Filtra por namespace: `poslite`
3. Deberías ver Services ClusterIP para cada componente

4. Ve a **Config & Storage > ConfigMaps** y **Secrets**
5. Verifica que existan los ConfigMaps y Secrets necesarios

---

## 💻 Opción 2: Verificar con kubectl

### Conectarse al cluster

```bash
# Si usas Rancher, exporta el kubeconfig
# O usa: kubectl config use-context <nombre-cluster>
```

### Verificar estado de Fleet

```bash
# Ver todos los bundles de Fleet
kubectl get bundles.fleet.cattle.io -A

# Ver detalles de un bundle específico
kubectl describe bundle.fleet.cattle.io <nombre-bundle> -n fleet-default

# Ver gitrepos (repositorios registrados)
kubectl get gitrepos.fleet.cattle.io -A

# Ver clusters de Fleet
kubectl get clusters.fleet.cattle.io -A
```

### Verificar Pods y Deployments

```bash
# Ver todos los pods en el namespace poslite
kubectl get pods -n poslite

# Ver deployments
kubectl get deployments -n poslite

# Ver statefulsets
kubectl get statefulsets -n poslite

# Ver servicios
kubectl get services -n poslite

# Ver configmaps
kubectl get configmaps -n poslite

# Ver secrets
kubectl get secrets -n poslite
```

### Verificar estado detallado

```bash
# Ver estado de un pod específico
kubectl describe pod <nombre-pod> -n poslite

# Ver eventos del namespace
kubectl get events -n poslite --sort-by='.lastTimestamp'

# Ver logs de un pod
kubectl logs <nombre-pod> -n poslite

# Ver logs de todos los pods de un deployment
kubectl logs -n poslite -l app=<nombre-app> --tail=50
```

### Verificar puertos

```bash
# Verificar que los pods tengan hostPort configurado
kubectl get pods -n poslite -o yaml | grep -A 5 hostPort

# Verificar que los puertos estén escuchando en el host
# (ejecutar en el nodo del cluster)
netstat -tulpn | grep -E "7014|7012|5432|9010|9014|10012|10014"
```

---

## 📝 Verificar Logs de Fleet

Si algo no funciona, revisa los logs de Fleet:

```bash
# Logs del Fleet Controller
kubectl logs -n fleet-system -l app=fleet-controller --tail=100

# Logs del Fleet Agent (en cada cluster)
kubectl logs -n fleet-system -l app=fleet-agent --tail=100
```

---

## ✅ Checklist de Verificación

Usa este checklist para verificar que todo está desplegando:

- [ ] **Git Repo**: Repositorio `atlas-stores` aparece como "Active" en Fleet
- [ ] **Bundles**: Los bundles aparecen en Fleet > Bundles
- [ ] **Target Clusters**: Los bundles tienen clusters asignados
- [ ] **Status**: Los bundles muestran estado "Ready" (no "Pending" o "Error")
- [ ] **Pods**: Los pods están en estado "Running" (no "Pending" o "Error")
- [ ] **Deployments**: Los deployments muestran réplicas deseadas = réplicas disponibles
- [ ] **StatefulSet**: PostgreSQL está corriendo
- [ ] **Services**: Los services ClusterIP están creados
- [ ] **ConfigMaps**: Los ConfigMaps existen
- [ ] **Secrets**: Los Secrets existen
- [ ] **Puertos**: Los puertos están expuestos (hostPort)
- [ ] **Logs**: No hay errores críticos en los logs

---

## 🔍 Troubleshooting Rápido

### Si Fleet no detecta cambios:

```bash
# Forzar actualización del repositorio
kubectl annotate gitrepo.fleet.cattle.io atlas-stores -n fleet-default fleet.cattle.io/force-update="$(date +%s)"
```

### Si los bundles no se aplican:

1. Verificar labels del cluster:
   ```bash
   kubectl get clusters.fleet.cattle.io <nombre-cluster> -n fleet-default -o yaml | grep -A 10 labels
   ```

2. Verificar que los labels coincidan con el `clusterSelector` en `groups/*.yaml`

### Si los pods no inician:

```bash
# Ver eventos recientes
kubectl get events -n poslite --sort-by='.lastTimestamp' | tail -20

# Ver descripción del pod para ver errores
kubectl describe pod <nombre-pod> -n poslite
```

---

## 📊 Comandos Útiles de Monitoreo

```bash
# Ver estado general del namespace
kubectl get all -n poslite

# Ver uso de recursos
kubectl top pods -n poslite

# Ver historial de despliegues
kubectl rollout history deployment/<nombre> -n poslite

# Ver estado de un rollout
kubectl rollout status deployment/<nombre> -n poslite
```

---

## 🎯 Flujo Normal de Despliegue

1. **Push a Git** → Cambios en el repositorio
2. **Fleet detecta** → GitRepo se actualiza (30-60 segundos)
3. **Fleet procesa** → Bundles se crean/actualizan
4. **Fleet aplica** → Helm charts se despliegan en clusters
5. **Kubernetes crea** → Pods, Services, ConfigMaps, Secrets
6. **Pods inician** → Containers se ejecutan
7. **Health checks** → Kubernetes verifica que estén saludables

**Tiempo estimado**: 2-5 minutos desde el push hasta que los pods estén corriendo.

---

## 📞 Si algo falla

1. Revisar logs de Fleet (ver arriba)
2. Verificar eventos del namespace: `kubectl get events -n poslite`
3. Verificar descripción de recursos: `kubectl describe <tipo> <nombre> -n poslite`
4. Consultar la sección de Troubleshooting en `doc/CONFIGURACION-RANCHER.md`
