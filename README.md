# atlas-stores

Repositorio de **definiciones de tiendas** para [Rancher Fleet](https://fleet.rancher.io/). Define qué se despliega en cada tienda (clusters Kubernetes etiquetados por `store`) usando Helm charts desde Azure Container Registry.

## ¿Qué hace este repo?

- **Fleet** lee solo la carpeta `stores/` (gracias a `.fleetignore`) y trata cada subcarpeta como un conjunto de **bundles**.
- Cada tienda tiene uno o más `fleet.yaml` que despliegan charts Helm (poslite-core, poslite-db, poslite-ht, poslite-pam) en los clusters cuyo label `store` coincide con el ID de la tienda.
- Las imágenes se obtienen de ACR; los Secrets para pull se gestionan en cada tienda dentro de `secret/`.

## Estructura

```
atlas-stores/
├── .fleetignore          # Ignora todo excepto stores/** → Fleet solo despliega desde stores/
├── README.md             # Este archivo
├── templates/
│   └── tienda-ejemplo/   # Plantilla para crear nuevas tiendas (no la despliega Fleet)
│       ├── core/         # poslite-core (POS principal)
│       ├── db/            # poslite-db (PostgreSQL + PgAdmin)
│       ├── horus/         # poslite-ht (Horustech)
│       ├── pam/           # poslite-pam (PAM)
│       ├── secret/        # Secrets (ACR, cloudflared)
│       └── README.md      # Instrucciones detalladas de la plantilla
└── stores/
    └── <id-tienda>/      # Una carpeta por tienda; Fleet crea bundles desde aquí
        ├── core/         # Opcional según tipo de tienda
        ├── db/
        ├── horus/
        ├── pam/
        ├── secret/
        └── ...
```

- **`templates/`**: excluida por `.fleetignore`. Solo sirve como plantilla; no se despliega.
- **`stores/`**: único contenido que Fleet usa. Cada `stores/<id-tienda>/` con sus `fleet.yaml` se despliega en los clusters con `store: <id-tienda>`.

## Tipos de tienda

| Tipo            | Carpetas típicas     | Descripción breve                          |
|-----------------|----------------------|--------------------------------------------|
| Solo Core       | db, core, secret     | POS principal (Portal, WebAPI, Workers)    |
| Solo Horustech  | db, horus, secret    | Stack Horustech (portal, webapi, workers)  |
| Solo PAM        | db, pam, secret      | PAM (portal, tcpconnector, scraper, etc.)  |
| Horustech + PAM | db, horus, pam, secret | Ambos stacks en el mismo cluster        |

Opcionalmente se pueden añadir carpetas para **Cloudflare** (cloudflared) según la plantilla.

## Crear una nueva tienda

1. **Copiar la plantilla** desde la raíz del repo:
   ```bash
   cp -r templates/tienda-ejemplo stores/<id-tienda>
   ```
   Ejemplo: `cp -r templates/tienda-ejemplo stores/mi-tienda`

2. **Sustituir placeholders** en todos los `fleet.yaml`:
   - `<id-tienda>` → ID de la tienda (ej: `mi-tienda`)
   - En horus/pam: `<ip-horustech>`, `<tag-imagen>` según corresponda
   - En `secret/fleet.yaml`: mismo `store` que el resto

3. **Elegir tipo de tienda**: borrar las carpetas que no uses (core, horus o pam) según el tipo.

4. **Configurar Secrets**: en `secret/acr-secret.yaml` usar las credenciales correctas. **No** subir contraseñas en claro a Git; en producción usar SOPS, Sealed Secrets o similar.

5. **Etiquetar el cluster en Rancher**:
   - `atlas: "true"`
   - `store: "<id-tienda>"`

Fleet desplegará en ese cluster todos los bundles bajo `stores/<id-tienda>/`.

## Documentación adicional

- **[templates/tienda-ejemplo/README.md](templates/tienda-ejemplo/README.md)** — Uso de la plantilla, placeholders, tipos de tienda y referencias.
- **stores/posdemos/** — Ejemplo de tienda real (Horustech + DB).

## Resumen técnico

- **Fleet**: GitOps; aplica los YAML bajo `stores/<id-tienda>/` según `clusterSelector.matchLabels`.
- **Helm (OCI)**: Charts en `atlashelmrepo.azurecr.io` (poslite-core, poslite-db, poslite-ht, poslite-pam).
- **Namespace**: `poslite` en todos los despliegues.
- **ACR**: Imágenes desde el registro configurado en `secret/acr-secret.yaml` (por ejemplo aspposlite.azurecr.io).
