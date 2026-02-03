# Tienda tipo: PAM + BD + Cloudflare

Plantilla para una tienda que despliega **PAM** (stack autosuficiente: portal, coreWebapi, workers, ierp), **PostgreSQL** y **Cloudflared**. No se despliega poslite-core (PAM ya incluye todo lo necesario).

## Labels del cluster (en Rancher)

- `atlas: "true"`
- `store: "<id-tienda>"` (obligatorio, ej: `tienda-pam-ejemplo`)

Fleet desplegará: **poslite-db**, **poslite-cloudflared-pam**, **poslite-pam**. No se despliega poslite-core.

## Archivos de valores

- **values-db.yaml**: overrides para PostgreSQL/PgAdmin.
- **values-cloudflared.yaml**: overrides para **poslite-cloudflared-pam** (hostnames PAM + BD).
- **values-pam.yaml**: overrides para PAM (Guard API, Portal, TCP Connector, Scraper, Playwright, coreWebapi, workers).

En atlas-stores: el **fleet.yaml** de esta carpeta ya define db + cloudflared-pam + pam (sin core). Copiar la carpeta a `atlas-stores/fleet/bundles/stores/`.

## Cómo llevar esta plantilla a atlas-stores

1. Copia la carpeta `tienda-pam-ejemplo` a `atlas-stores/fleet/bundles/stores/`.
2. Renómbrala al ID de la tienda (ej: `gasparhernandez`) y en `fleet.yaml` sustituye `tienda-pam-ejemplo` por ese ID (clusterSelector y hostnames).
3. Asigna al cluster las labels `atlas: "true"` y `store: <id-tienda>`.
