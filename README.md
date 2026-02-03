# Atlas Stores

Configuración **por tienda** para desplegar **PosLite** en clusters RKE2 con **Fleet** (Rancher). Este repositorio no contiene código de aplicación: solo YAML que Fleet lee desde Git y aplica en cada cluster según el nombre de la tienda.

---

## Nueva estructura del repositorio

El modelo es **una tienda = una carpeta** dentro de `stores/`. No hay grupos ni plantillas globales: todo el despliegue se define por tienda.

```
atlas-stores/
├── README.md
├── doc/                              # Documentación técnica
│   ├── ESTRUCTURA-COMPLETA.md
│   ├── FUNCIONAMIENTO-SISTEMA.md
│   ├── GESTION-TIENDA-INDIVIDUAL.md
│   ├── CONFIGURACION-RANCHER.md
│   ├── HELM-REPOSITORY-SETUP.md
│   ├── CONFIGURAR-AUTENTICACION-ACR.md
│   ├── VERIFICAR-DESPLIEGUE.md
│   ├── MIGRACION-GUIDE.md
│   └── ATLAS-RKE2-MIGRACION.md
│
└── stores/                           # Una carpeta por tienda
    │
    ├── atlasposlitepilot/             # Tienda real (piloto)
    │   ├── fleet.yaml                 # Qué charts y con qué valores
    │   ├── values-common.yaml         # Común: registry, timezone, recursos
    │   ├── values-db.yaml             # PostgreSQL
    │   ├── values-core.yaml           # Core (Portal, WebAPI)
    │   ├── values-horustech.yaml      # Horustech (esta tienda es tipo Horustech)
    │   └── secrets.yaml               # Secretos (BD, ACR, Cloudflared, etc.)
    │
    ├── ejemplo-tienda-core/          # Plantilla: solo Core + BD + Cloudflared
    ├── ejemplo-tienda-horustech/      # Plantilla: Horustech + Core + BD + Cloudflared
    └── ejemplo-tienda-pam/            # Plantilla: PAM + Core + BD + Cloudflared
```

### Qué hay dentro de cada tienda

| Archivo | Uso |
|--------|-----|
| **fleet.yaml** | Define los bundles de Fleet: qué Helm charts se instalan y qué archivos de valores usan. El `clusterSelector` usa el label `store: "<nombre-tienda>"`. |
| **values-common.yaml** | Valores compartidos: registry de imágenes, timezone, recursos, `storageClass`. |
| **values-db.yaml** | Configuración de PostgreSQL. |
| **values-core.yaml** | Configuración de Core (Portal, WebAPI). |
| **values-horustech.yaml** o **values-pam.yaml** | Solo si la tienda es Horustech o PAM. Configuración específica del POS. |
| **secrets.yaml** | Secretos: contraseñas BD, cadena de conexión, ACR, Cloudflared, SMTP. Opcional: SOPS en `secrets.sops.yaml`. |

Las carpetas `ejemplo-tienda-*` son **plantillas para copiar**. No se despliegan: sirven para crear una nueva carpeta con el mismo esquema (sustituyendo `<nombre-tienda>` por el nombre real).

---

## Estrategia de ramas en Git

Se usan **dos ramas** con roles bien definidos:

| Rama | Contenido | Uso |
|------|-----------|-----|
| **main** | Solo tiendas reales + documentación (sin plantillas) | **Rama de despliegue.** Fleet en Rancher debe apuntar a esta rama. Es la fuente de verdad de producción. |
| **develop** | Solo las plantillas (`ejemplo-tienda-*`) + documentación | **Rama de trabajo** para evolucionar plantillas. No contiene tiendas reales; al crear una nueva tienda se copia desde aquí y se añade en `main`. |

### Flujo de trabajo

- **Cambiar una plantilla** (p. ej. `ejemplo-tienda-horustech`): editar en `develop`; las plantillas solo existen en esta rama.
- **Añadir una tienda nueva**: copiar la plantilla desde `develop` (o desde la doc), crear la carpeta en `main` (o en una rama desde `main`, p. ej. `feature/<nombre-tienda>`), rellenar valores y secretos, commit y merge a `main`. La tienda solo existe en `main`.
- **Cambiar una tienda ya existente**: editar en `main` (o en una rama desde `main`), porque las tiendas reales solo viven en `main`.

Fleet lee siempre de **main** (solo tiendas reales). Las plantillas viven únicamente en **develop** para consulta y evolución.

---

## Cómo funciona el despliegue

1. **Fleet** tiene registrado este repositorio Git (por ejemplo en Rancher).
2. Fleet **descubre** cada `stores/<nombre-tienda>/fleet.yaml`.
3. Para cada **cluster** registrado, Fleet mira sus **labels**.
4. Si el cluster tiene el label **`store: "<nombre-tienda>"`** (y `atlas: "true"`), Fleet aplica el bundle de esa carpeta.
5. Cada bundle indica qué **charts** instalar (desde el registry OCI) y qué **valuesFiles** usar (los YAML de esa misma carpeta).

Resultado: **un cluster = una tienda = una carpeta en `stores/`**. No hay despliegues “por tipo” separados; todo va por nombre de tienda.

---

## Tipos de tienda

Hay tres variantes según el sistema POS. La carpeta de la tienda incluye los charts y values que correspondan.

| Tipo | Charts desplegados | Carpeta plantilla | Label extra en cluster |
|------|--------------------|-------------------|------------------------|
| **Solo Core** | BD, Core, Cloudflared | `ejemplo-tienda-core/` | — |
| **Horustech** | BD, Core, Horustech, Cloudflared | `ejemplo-tienda-horustech/` | `poslite: "horustech"` |
| **PAM** | BD, Core, PAM, Cloudflared | `ejemplo-tienda-pam/` | `poslite: "pam"` |

El label **`store: "<nombre-tienda>"`** es obligatorio y debe coincidir con el nombre de la carpeta en `stores/`.

---

## Cómo crear una nueva tienda

1. **Copiar la plantilla** desde la rama **develop** (en `main` no hay plantillas). Según el tipo (Core, Horustech o PAM), en `develop` ejecutar:
   ```bash
   mkdir -p stores/<nombre-tienda>
   cp -r stores/ejemplo-tienda-horustech/* stores/<nombre-tienda>/
   ```
   Luego crear esa carpeta en `main` (o en una rama desde `main`) con los archivos copiados y adaptados.

2. **Sustituir `<nombre-tienda>`** por el nombre real en `stores/<nombre-tienda>/fleet.yaml` (en `store: "..."`, en `name: ...-db`, `name: ...-core`, etc.). En PowerShell:
   ```powershell
   (Get-Content stores/<nombre-tienda>/fleet.yaml) -replace '<nombre-tienda>', '<nombre-real>' | Set-Content stores/<nombre-tienda>/fleet.yaml
   ```

3. **Editar los values** de esa carpeta: `values-common.yaml`, `values-db.yaml`, `values-core.yaml` y, si aplica, `values-horustech.yaml` o `values-pam.yaml`.

4. **Configurar secretos**: renombrar o copiar `secrets.sops.yaml.example` a `secrets.yaml` y rellenar con los valores reales (BD, ACR, Cloudflared, etc.).

5. **Etiquetar el cluster** en Rancher con `atlas: "true"` y `store: "<nombre-tienda>"` (y `poslite: "horustech"` o `poslite: "pam"` si aplica).

6. **Commit y push** de la carpeta `stores/<nombre-tienda>/` en la rama **main** (o merge a `main` si trabajaste en una rama `feature/`). Fleet aplicará el bundle en el cluster que tenga ese `store`.

---

## Secretos

- **secrets.yaml** (sin encriptar): Fleet lo usa directamente. Adecuado si el repo es privado.
- **secrets.sops.yaml** (con SOPS): Fleet no desencripta SOPS; hace falta otro mecanismo (Sealed Secrets, External Secrets o desencriptar antes).

Los ejemplos usan `secrets.sops.yaml.example` como plantilla; al crear la tienda se copia/renombra a `secrets.yaml` y se rellenan los valores.

---

## Variables de referencia

- **BD:** `postgresql.password`, `postgresql.user`, `postgresql.database`, `db-connection-string`
- **Core:** `portal.enabled`, `webapi.enabled`, `config.coreApiKey`, `config.ierpUrl`
- **Horustech:** `horustech.ip`, `config.horustechIp`, puertos WebAPI/Workers
- **PAM:** `pam.ip`, `pam.password`, `config.pamIp`
- **Cloudflared:** `cloudflared.tunnel-id`, `cloudflared.credentials` (JSON)

El orden de precedencia de valores es: defaults del chart → valores inline en `fleet.yaml` → archivos en `valuesFiles`.

---

## Documentación en `doc/`

| Archivo | Contenido |
|---------|-----------|
| ESTRUCTURA-COMPLETA.md | Estructura del repo: directorios, ramas, contenido por tienda |
| FUNCIONAMIENTO-SISTEMA.md | Arquitectura, Fleet, Helm, flujo GitOps |
| GESTION-TIENDA-INDIVIDUAL.md | Actualizar o instalar una sola tienda |
| CONFIGURACION-RANCHER.md | Configuración en Rancher |
| HELM-REPOSITORY-SETUP.md | Repositorio Helm OCI |
| CONFIGURAR-AUTENTICACION-ACR.md | Autenticación Azure Container Registry |
| VERIFICAR-DESPLIEGUE.md | Comprobar que el despliegue es correcto |
| MIGRACION-GUIDE.md | Migración Docker Compose → RKE2 |
| ATLAS-RKE2-MIGRACION.md | Especificaciones de migración RKE2 |

---

## Problemas frecuentes

- **Fleet no aplica:** Comprobar que el repo esté registrado, que el cluster tenga `atlas: "true"` y `store: "<nombre-tienda>"`, y revisar logs: `kubectl logs -n fleet-system -l app=fleet-controller`.
- **Secretos no se aplican:** Verificar que exista `secrets.yaml` en la carpeta de la tienda y que el YAML sea válido.
- **Valores no se aplican:** Revisar que los paths en `valuesFiles` sean relativos al `fleet.yaml` y que los archivos existan en esa carpeta.

Para más detalle: `kubectl get bundles.fleet.cattle.io -n fleet-default` y `kubectl describe bundle.fleet.cattle.io <nombre-bundle> -n fleet-default`.

---

Contacto: equipo de DevOps.
