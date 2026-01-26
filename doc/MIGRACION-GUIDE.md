# Guía de Migración Docker Compose → RKE2 con Fleet

## Resumen

Esta guía documenta la migración completa del sistema PosLite desde Docker Compose hacia RKE2 (Rancher Kubernetes Engine v2) gestionado con Rancher y GitOps mediante Fleet.

## Estructura de Repositorios

### Repositorio: `atlas-apps` (Plantillas)

Contiene todos los recursos reutilizables:
- **Helm Charts**: Plantillas parametrizables para cada componente
- **Fleet Bundles**: Configuraciones de despliegue por tipo de servicio

```
atlas-apps/
├── charts/
│   ├── poslite-db/          # PostgreSQL + PgAdmin
│   ├── poslite-core/        # Core (Portal, WebAPI, Workers)
│   ├── poslite-horustech/   # Servicios Horustech
│   ├── poslite-pam/         # Servicios PAM
│   └── poslite-cloudflared/ # Cloudflared tunnel
└── fleet/
    └── bundles/
        ├── db/
        ├── core/
        ├── horustech/
        ├── pam/
        └── cloudflared/
```

### Repositorio: `atlas-stores` (Configuración por Tienda)

Contiene la configuración específica de cada tienda:

```
atlas-stores/
├── stores/
│   └── <nombre-tienda>/
│       ├── values-common.yaml
│       ├── values-db.yaml
│       ├── values-pam.yaml  (o values-horustech.yaml)
│       └── secrets.sops.yaml
└── groups/
    ├── pilot.yaml
    ├── wave1.yaml
    └── wave2.yaml
```

## Puertos (CONGELADOS - NO MODIFICAR)

### Base de Datos
- **PostgreSQL**: `hostPort: 5432` → `containerPort: 5432`
- **PgAdmin**: `hostPort: 5050` → `containerPort: 80`

### Core
- **core-webapi**: `hostPort: 10012` → `containerPort: 8080`
- **core-portal**: `hostPort: 10014` → `containerPort: 8080`

### Horustech
- **horustech-webapi**: `hostPort: 9010` → `containerPort: 8080`
- **horustech-core-webapi**: `hostPort: 9012` → `containerPort: 8080`
- **horustech-core-portal**: `hostPort: 9014` → `containerPort: 8080`
- **horustech-guard-api**: `hostPort: 9015` → `containerPort: 8080`
- **horustech-core-webevents**: `hostPort: 9020` → `containerPort: 8080`
- **horustech-core-loyalty-config-worker**: `hostPort: 9021` → `containerPort: 8080`

### PAM
- **pam-tcpconnector**: `hostPort: 7010` → `containerPort: 8080`
- **pam-playwright**: `hostPort: 7011` → `containerPort: 3320`
- **pam-core-webapi**: `hostPort: 7012` → `containerPort: 8080`
- **pam-core-portal**: `hostPort: 7014` → `containerPort: 8080`
- **pam-guard-api**: `hostPort: 7015` → `containerPort: 8080`
- **pam-scraper**: `hostPort: 7016` → `containerPort: 8080`
- **pam-core-webevents**: `hostPort: 7020` → `containerPort: 8080`

## Reglas Técnicas Obligatorias

### 1. Uso de hostPort

**TODOS** los servicios expuestos deben usar `hostPort` en lugar de NodePort o LoadBalancer:

```yaml
ports:
  - containerPort: 8080
    hostPort: 9010
    protocol: TCP
```

### 2. Cloudflared dentro de Kubernetes

Cloudflared **DEBE** correr como Deployment dentro del cluster con:
- `hostNetwork: true`
- Apuntar a `localhost:<PUERTO>` (ej: `http://localhost:9010`)

```yaml
spec:
  hostNetwork: true
  containers:
  - name: cloudflared
    command:
    - cloudflared
    - tunnel
    - run
```

### 3. PostgreSQL como StatefulSet

PostgreSQL debe correr como StatefulSet con:
- PVC local para persistencia
- `hostPort: 5432`
- Migraciones mediante Job o initContainer

### 4. Labels de Fleet

Los clusters deben tener los siguientes labels:

```yaml
labels:
  atlas: "true"
  store: "<nombre-tienda>"
  poslite: "pam" | "horustech"
  wave: "pilot" | "wave1" | "wave2"
```

## Proceso de Migración

### Paso 1: Preparar RKE2 Cluster

1. Instalar RKE2 en el Raspberry Pi
2. Registrar el cluster en Rancher
3. Aplicar labels al cluster:
   ```bash
   kubectl label cluster <cluster-name> atlas=true store=<tienda> poslite=<pam|horustech>
   ```

### Paso 2: Configurar GitOps

1. Crear repositorio `atlas-apps` con los charts
2. Crear repositorio `atlas-stores` con configuración por tienda
3. Registrar repositorios en Rancher Fleet

### Paso 3: Crear Configuración por Tienda

En `atlas-stores/stores/<tienda>/`:

**values-db.yaml:**
```yaml
postgresql:
  password: "<password>"
  timezone: "America/Panama"
```

**values-pam.yaml** (o values-horustech.yaml):
```yaml
config:
  ierpUrl: "http://ierpgateway.azurewebsites.net/"
  horustechIp: "192.168.1.100"
  # ... otras configuraciones
```

**secrets.sops.yaml:**
```yaml
# Encriptado con SOPS
db-connection-string: "Host=..."
smtp-password: "..."
```

### Paso 4: Desplegar con Fleet

Fleet desplegará automáticamente según los labels del cluster.

## Estructura de Helm Charts

### Chart: poslite-db

- **StatefulSet** para PostgreSQL
- **Deployment** para PgAdmin (opcional)
- **ConfigMap** para configuración PostgreSQL
- **Secret** para credenciales
- **PVC** para persistencia

### Chart: poslite-core

- **Deployment** para portal (hostPort: 10014)
- **Deployment** para webapi (hostPort: 10012)
- **Deployments** para workers (sin puertos)
- **Service** ClusterIP para comunicación interna
- **PVC** para uploads

### Chart: poslite-horustech / poslite-pam

Similar a core pero con servicios específicos de Horustech/PAM.

### Chart: poslite-cloudflared

- **Deployment** con `hostNetwork: true`
- **ConfigMap** para configuración del tunnel
- **Secret** para credenciales de Cloudflare

## Volúmenes Persistentes

### Requeridos

1. **PostgreSQL Data**: `ReadWriteOnce`, 20Gi
2. **Uploads**: `ReadWriteMany`, 10Gi
3. **Licenses**: `ReadOnlyMany`, 1Gi (si aplica)
4. **Pointer Data**: `ReadWriteOnce`, 5Gi (solo Horustech/PAM)

### StorageClass

Usar `local-path` para almacenamiento local en RPI.

## Migración de Datos

### PostgreSQL

1. Hacer backup de la base de datos actual:
   ```bash
   docker exec poslite-postgres-db pg_dump -U sa poslite > backup.sql
   ```

2. Desplegar PostgreSQL en Kubernetes

3. Restaurar backup:
   ```bash
   kubectl exec -it <postgres-pod> -- psql -U sa -d poslite < backup.sql
   ```

### Volúmenes

1. Copiar datos de `/datos/uploads` al PVC
2. Copiar licencias de `/datos/.licenses` al PVC de licenses

## Verificación Post-Migración

### Checklist

- [ ] Todos los servicios están corriendo
- [ ] Puertos expuestos correctamente (verificar con `netstat -tulpn`)
- [ ] Cloudflared conectado y funcionando
- [ ] Base de datos accesible
- [ ] Workers procesando correctamente
- [ ] Integración con IERP funcionando
- [ ] Licencias validadas (si aplica)

### Comandos de Verificación

```bash
# Verificar pods
kubectl get pods -n poslite

# Verificar servicios
kubectl get svc -n poslite

# Verificar puertos en el host
sudo netstat -tulpn | grep -E '5432|5050|10012|10014|9010|7010'

# Ver logs de Cloudflared
kubectl logs -n poslite deployment/poslite-cloudflared

# Verificar conectividad a base de datos
kubectl exec -it <postgres-pod> -n poslite -- psql -U sa -d poslite -c "SELECT version();"
```

## Rollback

En caso de problemas:

1. **Rollback de Helm Release:**
   ```bash
   helm rollback <release-name> -n poslite
   ```

2. **Rollback de Git (Fleet):**
   ```bash
   git revert <commit-hash>
   git push
   ```

3. **Restaurar Docker Compose:**
   ```bash
   # Detener servicios Kubernetes
   kubectl delete namespace poslite
   
   # Iniciar Docker Compose
   docker-compose up -d
   ```

## Troubleshooting

### Problema: Puerto ya en uso

**Causa**: Docker Compose aún corriendo o conflicto de puertos.

**Solución**:
```bash
# Verificar qué está usando el puerto
sudo lsof -i :9010

# Detener Docker Compose
docker-compose down

# Verificar que el puerto esté libre
sudo netstat -tulpn | grep 9010
```

### Problema: Cloudflared no conecta

**Causa**: Configuración incorrecta o credenciales inválidas.

**Solución**:
1. Verificar ConfigMap de Cloudflared
2. Verificar Secret de credenciales
3. Verificar logs: `kubectl logs -n poslite deployment/poslite-cloudflared`

### Problema: Workers no procesan

**Causa**: Variables de entorno incorrectas o conexión a BD fallando.

**Solución**:
1. Verificar Secret de conexión a BD
2. Verificar logs del worker: `kubectl logs -n poslite deployment/<worker-name>`
3. Verificar conectividad: `kubectl exec -it <worker-pod> -- env | grep DB`

## Notas Importantes

1. **NO cambiar puertos**: Los puertos están congelados y deben mantenerse exactamente iguales.

2. **Cloudflared SIEMPRE en K8s**: No sacar Cloudflared del cluster.

3. **Una tienda = Un cluster**: Cada Raspberry Pi es un cluster RKE2 single-node.

4. **PAM O Horustech**: Nunca ambos en la misma tienda.

5. **GitOps obligatorio**: Todos los cambios deben pasar por Git.

## Contacto y Soporte

Para problemas o consultas sobre la migración, contactar al equipo de DevOps.

---

**Última actualización**: Enero 2026  
**Versión**: 1.0
