# Plantilla: tienda ejemplo

Plantilla base para crear una **nueva tienda** en `atlas-stores`. Se copia esta carpeta a `stores/<id-tienda>/`, se reemplazan los placeholders y opcionalmente se eliminan las subcarpetas que no correspondan al tipo de tienda (solo Core, solo Horustech, solo PAM, etc.).

La carpeta `templates/` está excluida por **`.fleetignore`** en la raíz del repo: Fleet no crea bundles desde aquí; solo despliega lo que está bajo `stores/<id-tienda>/`.

## Estructura de la plantilla

```
templates/tienda-ejemplo/
├── core/
│   └── fleet.yaml          # Chart poslite-core (POS principal)
├── db/
│   └── fleet.yaml          # Chart poslite-db (PostgreSQL + PgAdmin)
├── horus/
│   └── fleet.yaml          # Chart poslite-ht (Horustech)
├── pam/
│   └── fleet.yaml          # Plantilla PAM (poslite-ht o chart PAM; ajustar según tienda)
├── secret/
│   ├── fleet.yaml          # Bundle que aplica los Secrets en el namespace poslite
│   └── acr-secret.yaml     # Secret para pull de imágenes desde ACR
└── README.md
```

Cada subcarpeta con `fleet.yaml` se convierte en **un bundle** de Fleet. No copies esta plantilla directamente al repo de Fleet: primero copia a `stores/<id-tienda>/` dentro de **atlas-stores**.

## Cómo usar la plantilla

### 1. Copiar la plantilla a stores

Desde la raíz del repo:

```bash
cp -r templates/tienda-ejemplo stores/<id-tienda>
```

Ejemplo: `cp -r templates/tienda-ejemplo stores/mi-tienda`

### 2. Sustituir placeholders en todos los `fleet.yaml`

En **cada** `fleet.yaml` (core, db, horus, pam, secret) ajusta el selector del cluster:

| Placeholder   | Dónde aparece        | Qué poner                                      |
|---------------|----------------------|-------------------------------------------------|
| `<id-tienda>` | core, db, horus, pam, secret | ID único de la tienda (ej: `mi-tienda`). Debe coincidir con el label `store` del cluster. |

En **horus/fleet.yaml** (y en **pam/fleet.yaml** si usas PAM con la misma lógica) sustituye también:

| Placeholder     | Dónde      | Qué poner                                      |
|-----------------|------------|-------------------------------------------------|
| `<ip-horustech>`| config     | IP del equipo/servidor Horustech (ej: `192.168.10.92`) |
| `<tag-imagen>`  | image.tag  | Tag de las imágenes (ej: `unstable`, `1.2.3`)   |

En **secret/fleet.yaml** el `store` debe ser el mismo `<id-tienda>` que en el resto.

### 3. Elegir tipo de tienda (qué carpetas dejar)

- **Solo Core:** deja `db/`, `core/`, `secret/`. Opcional: `cloudflared-core/` (crear con su `fleet.yaml`). Borra `horus/` y `pam/`.
- **Solo Horustech:** deja `db/`, `horus/`, `secret/`. Opcional: `cloudflared-horustech/`. Borra `core/` y `pam/`.
- **Solo PAM:** deja `db/`, `pam/`, `secret/`. Ajusta `pam/fleet.yaml` con el chart y values de PAM (poslite-pam u otro). Opcional: `cloudflared-pam/`. Borra `core/` y `horus/`.
- **Horustech + PAM:** deja db, horus, pam, secret; asegúrate de que los nombres/selectores no choquen (por ejemplo distintos `name` en cada target).

Si añades **Cloudflare**, crea una carpeta `cloudflared-core/`, `cloudflared-horustech/` o `cloudflared-pam/` con su `fleet.yaml` (ver `stores/README.md`). Hostnames: `<nombre-tienda>-<puerto>.asptienda.com`.

### 4. Configurar Secrets

En **secret/acr-secret.yaml**:

- Sustituye el registro y las credenciales por las de la tienda (o las de tu ACR).
- **No** subas contraseñas en claro a Git; en producción usa SOPS, Sealed Secrets o un sistema seguro.

Si usas cloudflared, añade en `secret/` el manifest del Secret con el JSON del tunnel (por ejemplo `horustech-cloudflared-credentials.yaml`).

### 5. Ajustar configuración de la tienda

En **horus/fleet.yaml** (o **pam/fleet.yaml**) revisa:

- `config`: timezone, companyId, coreApiKey, ierpUrl, horustechPortWebapi / horustechPortWorkers, etc.
- Puertos (`hostPort`) si hay conflictos con otras tiendas en el mismo cluster.
- Réplicas por componente si hace falta.

En **core/fleet.yaml** revisa values del chart poslite-core según la tienda.

### 6. Labels en el cluster (Rancher)

En Rancher, el cluster que recibirá esta tienda debe tener:

- `atlas: "true"`
- `store: "<id-tienda>"` (el mismo que usaste en los fleet.yaml)

Fleet desplegará en ese cluster todos los bundles de `stores/<id-tienda>/` que tengan `fleet.yaml`.

## Resumen de placeholders

| Archivo / carpeta    | Reemplazar |
|----------------------|------------|
| **core/fleet.yaml**  | `store: "<id-tienda>"` → `store: "mi-tienda"` |
| **db/fleet.yaml**    | `store: "<id-tienda>"` → `store: "mi-tienda"` |
| **horus/fleet.yaml** | `store: "<id-tienda>"`, `horustechIp: "<ip-horustech>"`, `tag: "<tag-imagen>"` |
| **pam/fleet.yaml**   | `store: "<id-tienda>"` (y chart/values si usas PAM distinto de horus) |
| **secret/fleet.yaml**| `store: "posdemos"` → `store: "mi-tienda"` (o el id que uses) |
| **secret/acr-secret.yaml** | Usuario/contraseña (y registro si aplica) del ACR |

## bundleVersion

En los `fleet.yaml` que usan Helm (core, db, horus, pam) hay un campo **`bundleVersion`** dentro de `values`. Si en el futuro quieres forzar un redeploy sin cambiar la versión del chart, sube ese número (por ejemplo de `"1.0.0"` a `"1.0.1"`), haz commit y push; Fleet reconciliará y Helm hará upgrade.

## Referencias

- **stores/README.md**: estructura general de tiendas, tipos (Core / Horustech / PAM), hostnames cloudflared.
- **stores/posdemos/**: ejemplo real de tienda Horustech ya configurada.
