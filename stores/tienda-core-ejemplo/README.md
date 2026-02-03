# Tienda tipo: Core + BD + Cloudflare

Plantilla para una tienda que solo despliega **Core**, **PostgreSQL** y **Cloudflared** (solo puertos Core + BD).

## Labels del cluster (en Rancher)

Aplica estas labels al cluster RKE2:

- `atlas: "true"`
- `store: <id-tienda>` (obligatorio con un bundle por tienda, ej: `tienda-core-ejemplo`)

Con `atlas: "true"` Fleet desplegará: **poslite-db**, **poslite-core**, **poslite-cloudflared-core** (chart específico: puertos 10012, 10014, 5050, 5432).

## Archivos de valores

- **values-db.yaml**: overrides para PostgreSQL/PgAdmin (timezone, tamaño PVC, etc.).
- **values-core.yaml**: overrides para Portal, WebAPI, workers.
- **values-cloudflared.yaml**: overrides para el chart **poslite-cloudflared-core** (replicas, config.ingress con hostnames, etc.).

En atlas-stores: copia esta carpeta a `fleet/bundles/stores/<id-tienda>/`. El `fleet.yaml` incluye db, core y cloudflared-core (chart `poslite-cloudflared-core` desde ACR).

## Cómo llevar esta plantilla a atlas-stores

1. Copia la carpeta `tienda-core-ejemplo` a `atlas-stores/fleet/bundles/stores/`.
2. Renómbrala al ID de tu tienda (ej: `mi-tienda-core`) y en `fleet.yaml` sustituye `tienda-core-ejemplo` por ese ID en `clusterSelector` y en los hostnames de cloudflared.
3. Asigna al cluster las labels `atlas: "true"` y `store: <id-tienda>`.
