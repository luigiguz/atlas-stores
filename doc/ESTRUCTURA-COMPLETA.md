# Estructura de atlas-stores

Contexto de cómo está organizado el repositorio **atlas-stores**: directorios, ramas, contenido por tienda y relación con Fleet.

---

## Qué es atlas-stores

- Repositorio de **configuración por tienda** para desplegar **PosLite** en clusters RKE2 con **Fleet** (Rancher).
- **No contiene código de aplicación**: solo YAML que Fleet lee desde Git y aplica en cada cluster según el nombre de la tienda.
- Modelo: **una tienda = una carpeta** dentro de `stores/`. No hay grupos ni plantillas globales; todo el despliegue se define por tienda.

---

## Estructura del repositorio

```
atlas-stores/
├── README.md                    # Documentación principal y flujos
├── doc/                         # Documentación técnica
│   ├── ESTRUCTURA-COMPLETA.md   # Este archivo
│   ├── FUNCIONAMIENTO-SISTEMA.md
│   ├── GESTION-TIENDA-INDIVIDUAL.md
│   ├── CONFIGURACION-RANCHER.md
│   ├── HELM-REPOSITORY-SETUP.md
│   ├── CONFIGURAR-AUTENTICACION-ACR.md
│   ├── VERIFICAR-DESPLIEGUE.md
│   ├── MIGRACION-GUIDE.md
│   └── ATLAS-RKE2-MIGRACION.md
│
└── stores/                      # Una carpeta por tienda
    ├── <nombre-tienda>/         # Tienda real (solo en rama main)
    │   ├── fleet.yaml           # Bundles Fleet: charts y values
    │   ├── values-common.yaml   # Registry, timezone, recursos
    │   ├── values-db.yaml       # PostgreSQL
    │   ├── values-core.yaml     # Core (Portal, WebAPI)
    │   ├── values-horustech.yaml   # o values-pam.yaml si aplica
    │   ├── secrets.yaml         # Secretos (BD, ACR, Cloudflared, etc.)
    │   └── secrets.sops.yaml.example  # Plantilla de secretos (opcional)
    │
    └── ejemplo-tienda-*         # Plantillas (solo en rama develop/plantillas)
        ├── ejemplo-tienda-core/
        ├── ejemplo-tienda-horustech/
        └── ejemplo-tienda-pam/
```

- **Raíz:** `README.md` + carpeta `doc/` (documentación) y carpeta `stores/` (configuración por tienda).
- **stores/:** Cada subcarpeta es una tienda. El nombre de la carpeta debe coincidir con el label `store: "<nombre-tienda>"` del cluster en Rancher.

---

## Contenido de cada tienda

Cada carpeta `stores/<nombre-tienda>/` contiene:

| Archivo | Uso |
|--------|-----|
| **fleet.yaml** | Define los bundles de Fleet: qué Helm charts se instalan y qué archivos de valores usan. Incluye el `clusterSelector` con `store: "<nombre-tienda>"` (y opcionalmente `poslite: "horustech"` o `poslite: "pam"`). |
| **values-common.yaml** | Valores compartidos: registry de imágenes, timezone, recursos, `storageClass`. |
| **values-db.yaml** | Configuración de PostgreSQL. |
| **values-core.yaml** | Configuración de Core (Portal, WebAPI). |
| **values-horustech.yaml** o **values-pam.yaml** | Solo si la tienda es tipo Horustech o PAM. Configuración específica del POS. |
| **secrets.yaml** | Secretos: contraseñas BD, cadena de conexión, ACR, Cloudflared, SMTP. Fleet lo usa en el despliegue. Opcional: SOPS en `secrets.sops.yaml` (requiere mecanismo externo para desencriptar). |
| **secrets.sops.yaml.example** | Plantilla de secretos; se copia/renombra a `secrets.yaml` y se rellenan los valores reales. |

Las plantillas `ejemplo-tienda-*` tienen la misma estructura; sirven para copiar y crear una nueva carpeta de tienda con el mismo esquema.

---

## Estrategia de ramas en Git

| Rama | Contenido | Uso |
|------|-----------|-----|
| **main** | Solo tiendas reales + documentación (`doc/`, `README.md`). **Sin plantillas.** | **Rama de despliegue.** Fleet en Rancher apunta a esta rama. Los cambios entran por **Pull Request**; al mergear a `main` se despliega. |
| **develop** | Solo plantillas (`ejemplo-tienda-core/`, `ejemplo-tienda-horustech/`, `ejemplo-tienda-pam/`) + documentación. **Sin tiendas reales.** | Rama de trabajo para evolucionar plantillas. Al crear una nueva tienda se copia desde aquí y se añade en `main` (vía PR). |

- **Despliegue:** Solo lo que está en `main`. Entrada a `main` vía PR (revisión, merge).
- **Plantillas:** Solo en `develop`. En `main` no existen las carpetas `ejemplo-tienda-*`.

---

## Tipos de tienda

Tres variantes según el sistema POS. La carpeta de la tienda incluye los `values-*` que correspondan.

| Tipo | Charts desplegados | Plantilla | Label extra en cluster |
|------|--------------------|-----------|------------------------|
| **Solo Core** | BD, Core, Cloudflared | `ejemplo-tienda-core/` | — |
| **Horustech** | BD, Core, Horustech, Cloudflared | `ejemplo-tienda-horustech/` | `poslite: "horustech"` |
| **PAM** | BD, Core, PAM, Cloudflared | `ejemplo-tienda-pam/` | `poslite: "pam"` |

El label **`store: "<nombre-tienda>"`** es obligatorio y debe coincidir con el nombre de la carpeta en `stores/`.

---

## Cómo Fleet usa atlas-stores

1. Fleet tiene registrado este repositorio Git en Rancher, apuntando a la rama **main**.
2. Fleet descubre cada `stores/<nombre-tienda>/fleet.yaml` en esa rama.
3. Para cada cluster registrado, Fleet mira sus **labels**.
4. Si el cluster tiene `atlas: "true"` y **`store: "<nombre-tienda>"`** (y, si aplica, `poslite: "horustech"` o `poslite: "pam"`), Fleet aplica el bundle de esa carpeta.
5. Cada bundle indica qué Helm charts instalar (desde el registry OCI) y qué `valuesFiles` usar (los YAML de esa misma carpeta).

Resultado: **un cluster = una tienda = una carpeta en `stores/`** (en `main`).

---

## Resumen rápido

- **atlas-stores:** repo de configuración por tienda (YAML), sin código de aplicación.
- **Estructura:** `doc/` + `stores/<nombre-tienda>/` con `fleet.yaml`, `values-*.yaml`, `secrets.yaml`.
- **Ramas:** `main` = solo tiendas reales, despliegue vía PR; `develop` = solo plantillas.
- **Fleet:** lee `main`; cada cluster con label `store: X` recibe la carpeta `stores/X/`.

Para crear una nueva tienda, copiar la plantilla desde `develop`, adaptar en una rama desde `main`, y mergear a `main` por PR. Para más detalle: README.md y el resto de la documentación en `doc/`.
