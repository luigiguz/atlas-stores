# Estructura Completa de Migración RKE2

## Resumen

Se ha creado la estructura completa para migrar PosLite de Docker Compose a RKE2 con GitOps mediante Fleet.

## Estructura de Archivos Creados

```
Migracion/
├── ATLAS-RKE2-MIGRACION.md          # Especificaciones técnicas originales
├── MIGRACION-GUIDE.md               # Guía completa de migración
├── ESTRUCTURA-COMPLETA.md            # Este archivo
│
├── atlas-apps/                       # Repositorio de plantillas
│   ├── README.md
│   ├── charts/
│   │   ├── poslite-db/              # ✅ COMPLETO
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/
│   │   │       ├── statefulset.yaml
│   │   │       ├── configmap.yaml
│   │   │       ├── secret.yaml
│   │   │       ├── pvc-pgadmin.yaml
│   │   │       └── _helpers.tpl
│   │   │
│   │   ├── poslite-core/            # ✅ COMPLETO
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/
│   │   │       ├── deployment-portal.yaml
│   │   │       ├── deployment-webapi.yaml
│   │   │       ├── deployment-workers.yaml
│   │   │       ├── configmap.yaml
│   │   │       ├── secret.yaml
│   │   │       ├── pvc-uploads.yaml
│   │   │       └── _helpers.tpl
│   │   │
│   │   ├── poslite-horustech/        # ✅ COMPLETO
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/
│   │   │       ├── deployment-guard-api.yaml
│   │   │       ├── deployment-portal.yaml
│   │   │       ├── deployment-webapi.yaml
│   │   │       ├── deployment-core-webapi.yaml
│   │   │       ├── deployment-core-webevents.yaml
│   │   │       ├── deployment-core-loyalty-config-worker.yaml
│   │   │       ├── deployment-workers.yaml
│   │   │       ├── configmap.yaml
│   │   │       ├── secret.yaml
│   │   │       ├── pvc-uploads.yaml
│   │   │       ├── pvc-licenses.yaml
│   │   │       ├── pvc-pointer.yaml
│   │   │       └── _helpers.tpl
│   │   │
│   │   ├── poslite-pam/              # ✅ COMPLETO
│   │   │   ├── Chart.yaml
│   │   │   ├── values.yaml
│   │   │   └── templates/
│   │   │       ├── deployment-guard-api.yaml
│   │   │       ├── deployment-portal.yaml
│   │   │       ├── deployment-tcpconnector.yaml
│   │   │       ├── deployment-scraper.yaml
│   │   │       ├── deployment-playwright.yaml
│   │   │       ├── deployment-core-webapi.yaml
│   │   │       ├── deployment-core-webevents.yaml
│   │   │       ├── deployment-workers.yaml
│   │   │       ├── configmap.yaml
│   │   │       ├── secret.yaml
│   │   │       ├── pvc-uploads.yaml
│   │   │       ├── pvc-licenses.yaml
│   │   │       └── _helpers.tpl
│   │   │
│   │   └── poslite-cloudflared/      # ✅ COMPLETO
│   │       ├── Chart.yaml
│   │       ├── values.yaml
│   │       └── templates/
│   │           ├── deployment.yaml
│   │           ├── configmap.yaml
│   │           └── _helpers.tpl
│   │
│   └── fleet/
│       └── bundles/
│           ├── db/
│           │   └── fleet.yaml        # ✅ COMPLETO
│           ├── core/
│           │   └── fleet.yaml        # ✅ COMPLETO
│           ├── horustech/
│           │   └── fleet.yaml        # ✅ COMPLETO
│           ├── pam/
│           │   └── fleet.yaml        # ✅ COMPLETO
│           └── cloudflared/
│               └── fleet.yaml        # ✅ COMPLETO
│
└── atlas-stores/                     # Repositorio de configuración por tienda
    ├── README.md
    ├── stores/
    │   └── ejemplo-tienda/
    │       ├── values-common.yaml    # ✅ COMPLETO
    │       ├── values-db.yaml        # ✅ COMPLETO
    │       ├── values-pam.yaml       # ✅ COMPLETO
    │       ├── values-horustech.yaml # ✅ COMPLETO
    │       └── secrets.sops.yaml.example # ✅ COMPLETO
    └── groups/
        ├── pilot.yaml                # ✅ COMPLETO
        ├── wave1.yaml                # ✅ COMPLETO
        └── wave2.yaml                # ✅ COMPLETO
```

## Estado de Completitud

### ✅ Completado

1. **PostgreSQL Chart (poslite-db)**: 100%
   - StatefulSet con hostPort 5432
   - PgAdmin Deployment con hostPort 5050
   - ConfigMaps, Secrets, PVCs

2. **Core Chart (poslite-core)**: 100%
   - Portal (hostPort 10014)
   - WebAPI (hostPort 10012)
   - Todos los workers (sin puertos)
   - ConfigMaps, Secrets, PVCs

3. **Cloudflared Chart (poslite-cloudflared)**: 100%
   - Deployment con hostNetwork: true
   - ConfigMap para configuración
   - Apunta a localhost:<PUERTO>

4. **Fleet Bundles**: 100%
   - Todos los bundles creados
   - Configuración de targetCustomizations

5. **Estructura atlas-stores**: 100%
   - Templates de configuración por tienda
   - Grupos de despliegue (pilot, wave1, wave2)
   - Documentación

6. **Documentación**: 100%
   - Guía de migración completa
   - READMEs para cada repositorio
   - Ejemplos y mejores prácticas

### ✅ Completado (Actualizado)

4. **Horustech Chart (poslite-horustech)**: 100%
   - Guard API (hostPort 9015)
   - Portal (hostPort 9014)
   - WebAPI (hostPort 9010)
   - Core WebAPI (hostPort 9012)
   - Core WebEvents (hostPort 9020)
   - Core Loyalty Config Worker (hostPort 9021)
   - Todos los workers (sin puertos)
   - ConfigMaps, Secrets, PVCs

5. **PAM Chart (poslite-pam)**: 100%
   - Guard API (hostPort 7015)
   - Portal (hostPort 7014)
   - TCP Connector (hostPort 7010)
   - Scraper (hostPort 7016)
   - Playwright (hostPort 7011)
   - Core WebAPI (hostPort 7012)
   - Core WebEvents (hostPort 7020)
   - Todos los workers (sin puertos)
   - ConfigMaps, Secrets, PVCs

## Charts Completados

### Horustech Chart

Todos los templates han sido creados basándose en `new-compose.yaml`:
- ✅ Todos los servicios principales con `hostPort` correcto
- ✅ Todos los workers sin puertos expuestos
- ✅ Variables de entorno mapeadas desde Docker Compose
- ✅ Volúmenes persistentes configurados
- ✅ Services ClusterIP para comunicación interna

### PAM Chart

Todos los templates han sido creados basándose en `new-compose.yaml`:
- ✅ Todos los servicios principales con `hostPort` correcto
- ✅ Playwright con configuración especial (usuario pwuser, puerto 3320)
- ✅ Todos los workers sin puertos expuestos
- ✅ Variables de entorno mapeadas desde Docker Compose
- ✅ Volúmenes persistentes configurados
- ✅ Services ClusterIP para comunicación interna

## Matriz de Puertos (CONGELADA)

| Servicio | hostPort | containerPort | Chart |
|----------|----------|---------------|-------|
| postgres | 5432 | 5432 | poslite-db |
| pgadmin | 5050 | 80 | poslite-db |
| core-webapi | 10012 | 8080 | poslite-core |
| core-portal | 10014 | 8080 | poslite-core |
| horustech-webapi | 9010 | 8080 | poslite-horustech |
| horustech-core-webapi | 9012 | 8080 | poslite-horustech |
| horustech-core-portal | 9014 | 8080 | poslite-horustech |
| horustech-guard-api | 9015 | 8080 | poslite-horustech |
| horustech-core-webevents | 9020 | 8080 | poslite-horustech |
| horustech-core-loyalty-config-worker | 9021 | 8080 | poslite-horustech |
| pam-tcpconnector | 7010 | 8080 | poslite-pam |
| pam-playwright | 7011 | 3320 | poslite-pam |
| pam-core-webapi | 7012 | 8080 | poslite-pam |
| pam-core-portal | 7014 | 8080 | poslite-pam |
| pam-guard-api | 7015 | 8080 | poslite-pam |
| pam-scraper | 7016 | 8080 | poslite-pam |
| pam-core-webevents | 7020 | 8080 | poslite-pam |

## Próximos Pasos

1. **Validar charts**:
   ```bash
   helm lint ./charts/poslite-horustech
   helm lint ./charts/poslite-pam
   helm template poslite-horustech ./charts/poslite-horustech
   helm template poslite-pam ./charts/poslite-pam
   ```

4. **Probar despliegue local**:
   - Instalar charts en un cluster de prueba
   - Verificar que los puertos se expongan correctamente
   - Verificar que Cloudflared se conecte

5. **Configurar Fleet en Rancher**:
   - Registrar repositorio `atlas-apps` en Fleet
   - Registrar repositorio `atlas-stores` en Fleet
   - Aplicar labels a clusters de prueba

6. **Migración gradual**:
   - Empezar con tiendas piloto
   - Validar funcionamiento
   - Expandir a wave1 y wave2

## Notas Importantes

1. **NO cambiar puertos**: La matriz de puertos está congelada.

2. **hostPort obligatorio**: Todos los servicios expuestos deben usar hostPort.

3. **Cloudflared siempre en K8s**: Con hostNetwork: true y apuntando a localhost:<PUERTO>.

4. **PostgreSQL como StatefulSet**: Con PVC local para persistencia.

5. **Workers sin puertos**: Los workers NO exponen puertos al host.

6. **GitOps obligatorio**: Todos los cambios deben pasar por Git.

## Contacto

Para completar los charts faltantes o resolver dudas, contactar al equipo de DevOps.

---

**Fecha**: Enero 2026  
**Versión**: 1.0
