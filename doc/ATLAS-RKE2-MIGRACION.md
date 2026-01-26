# Atlas – Migración de Docker Compose a RKE2 con GitOps

## 1. Objetivo

Migrar la plataforma PosLite, actualmente desplegada con Docker Compose
y scripts Bash, hacia una arquitectura basada en:

- RKE2 (Kubernetes certificado por Rancher)
- Rancher (Atlas Core) como plano de control
- Fleet como motor GitOps
- Cloudflared SIEMPRE dentro de Kubernetes
- PostgreSQL local por tienda

La migración NO debe cambiar:
- Puertos
- Hostnames
- Forma de acceso externo
- Integraciones existentes

---

## 2. Modelo operativo

### 2.1 Topología

- 1 tienda = 1 Raspberry Pi
- 1 Raspberry Pi = 1 cluster RKE2 (single-node)
- Cada tienda ejecuta:
  - PostgreSQL local
  - Core (opcional)
  - PAM O HORUSTECH
  - Cloudflared

---

## 3. Repositorios GitOps (OBLIGATORIO – Opción 2)

### 3.1 Repositorio: atlas-apps (plantillas)

Contiene todo lo reutilizable y estándar:

```
atlas-apps/
├── charts/
│   ├── poslite-common
│   ├── poslite-db
│   ├── poslite-core
│   ├── poslite-horustech
│   ├── poslite-pam
│   └── poslite-cloudflared
└── fleet/
    └── bundles/
        ├── common
        ├── db
        ├── core
        ├── horustech
        ├── pam
        └── cloudflared
```

---

### 3.2 Repositorio: atlas-stores (configuración por tienda)

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

---

## 4. Reglas técnicas obligatorias

### 4.1 Kubernetes / RKE2

- Todos los recursos deben ser compatibles con RKE2
- No usar Docker Compose
- No usar NodePort
- No modificar rangos de puertos del cluster

---

## 5. Exposición de servicios (PUERTOS CONGELADOS)

LOS PUERTOS DEBEN SER EXACTAMENTE LOS MISMOS QUE DOCKER COMPOSE.

### Base de Datos
- postgres: hostPort **5432** → containerPort 5432
- pgadmin: hostPort **5050** → containerPort 80

### Core
- core-webapi: **10012 → 8080**
- core-portal: **10014 → 8080**

### Horustech
- horustech-webapi: **9010 → 8080**
- horustech-core-webapi: **9012 → 8080**
- horustech-core-portal: **9014 → 8080**
- horustech-guard-api: **9015 → 8080** (solo new-compose)
- horustech-core-webevents: **9020 → 8080**
- horustech-core-loyalty-config-worker: **9021 → 8080** (solo new-compose)

### Pam
- pam-tcpconnector: **7010 → 8080**
- pam-playwright: **7011 → 3320**
- pam-core-webapi: **7012 → 8080**
- pam-core-portal: **7014 → 8080**
- pam-guard-api: **7015 → 8080** (solo new-compose)
- pam-scraper: **7016 → 8080**
- pam-core-webevents: **7020 → 8080**

Los workers NO exponen puertos.

---

## 6. Uso obligatorio de hostPort

Todos los servicios expuestos deben usar `hostPort`.

Ejemplo:

```yaml
ports:
  - containerPort: 8080
    hostPort: 9010
```

---

## 7. Cloudflared (OBLIGATORIO dentro de Kubernetes)

Reglas:

- Cloudflared corre como Deployment
- Usa `hostNetwork: true`
- 1 réplica
- Configuración vía ConfigMap
- Credenciales vía Secret
- Debe apuntar a `localhost:<PUERTO>` exactamente igual que hoy

Ejemplos:
- http://localhost:9010
- http://localhost:7010
- tcp://127.0.0.1:5432
- tcp://localhost:22

---

## 8. PostgreSQL

- PostgreSQL corre como StatefulSet
- Usa PVC local
- Expone `hostPort: 5432`
- Migraciones mediante Job o initContainer

---

## 9. Fleet / GitOps

- Fleet despliega por labels:
  - atlas=true
  - store=<tienda>
  - poslite=pam | horustech
  - wave=pilot | wave1 | wave2
- Deploy = commit en Git
- Rollback = git revert

---

## 10. Principio final

Git define el estado deseado.  
RKE2 ejecuta exactamente ese estado.  
Cloudflared mantiene la conectividad sin cambios externos.

NO cambiar puertos.  
NO sacar Cloudflared del cluster.  
NO alterar el comportamiento funcional.
