# Tienda tipo: Horustech + BD + Cloudflare

Plantilla para una tienda que despliega **Horustech** (stack autosuficiente: portal, webapi, coreWebapi, workers, ierp), **PostgreSQL** y **Cloudflared**. No se despliega poslite-core (Horustech ya incluye todo lo necesario).

## Labels del cluster (en Rancher)

- `atlas: "true"`
- `store: "<id-tienda>"` (obligatorio, ej: `tienda-horustech-ejemplo`)

Fleet desplegará: **poslite-db**, **poslite-cloudflared-horustech**, **poslite-horustech**. No se despliega poslite-core.

## Archivos de valores

- **values-db.yaml**: overrides para PostgreSQL/PgAdmin.
- **values-cloudflared.yaml**: overrides para **poslite-cloudflared-horustech** (hostnames Horustech + BD).
- **values-horustech.yaml**: overrides para Horustech (Guard API, Portal, WebAPI, coreWebapi, workers).

En atlas-stores: el **fleet.yaml** de esta carpeta ya define db + cloudflared-horustech + horustech (sin core). Copiar la carpeta a `atlas-stores/fleet/bundles/stores/`.

## Cómo llevar esta plantilla a atlas-stores

1. Copia la carpeta `tienda-horustech-ejemplo` a `atlas-stores/fleet/bundles/stores/`.
2. Renómbrala al ID de la tienda y en `fleet.yaml` sustituye `tienda-horustech-ejemplo` por ese ID (clusterSelector y hostnames).
3. Asigna al cluster las labels `atlas: "true"` y `store: <id-tienda>`.
