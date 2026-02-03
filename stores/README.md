# Plantillas de tienda para atlas-stores (opción A: un bundle por tienda)

Cada subcarpeta es **una tienda** con su **fleet.yaml** incluido. El label **`store` es obligatorio**: cada cluster debe tener `atlas: "true"` y `store: "<nombre-carpeta>"` para recibir solo el despliegue de esa tienda.

## Dónde copiar en atlas-stores

Copia cada carpeta de tienda dentro de:

```
atlas-stores/fleet/bundles/stores/
```

Ejemplo: al copiar `tienda-pam-ejemplo` obtienes `atlas-stores/fleet/bundles/stores/tienda-pam-ejemplo/fleet.yaml`. Fleet descubre los bundles por esa ruta.

## Tipos de tienda

| Carpeta | Stack | Charts desplegados | Labels del cluster (obligatorios) |
|--------|--------|-------------------|------------------------------------|
| **tienda-core-ejemplo** | Solo Core | db, core, cloudflared-core | `atlas: "true"`, `store: "tienda-core-ejemplo"` |
| **tienda-pam-ejemplo** | PAM (autosuficiente, sin core) | db, cloudflared-pam, pam | `atlas: "true"`, `store: "tienda-pam-ejemplo"` |
| **tienda-horustech-ejemplo** | Horustech (autosuficiente, sin core) | db, cloudflared-horustech, horustech | `atlas: "true"`, `store: "tienda-horustech-ejemplo"` |

PAM y Horustech **no desplegar poslite-core** para evitar duplicar workers (p. ej. ierp).

**Hostnames:** `<nombre-tienda>-<puerto>.asptienda.com`. En cada `fleet.yaml` ya vienen con el nombre de la carpeta (ej: `tienda-pam-ejemplo-7012.asptienda.com`). Al renombrar la tienda, actualiza también los hostnames en el `fleet.yaml`.

Cada subcarpeta incluye:
- **fleet.yaml**: bundle con `clusterSelector` que exige `store`. Core solo para tienda-core; PAM/Horustech solo db + cloudflared-xxx + pam/horustech (sin core).
- **README.md**, **values-db.yaml**, **values-*.yaml**: referencia; los valores usados por Fleet están ya en `fleet.yaml`.

## Uso

1. Asegúrate de que los charts estén publicados en ACR (véase [doc/PUBLICAR-CHARTS-ACR.md](../doc/PUBLICAR-CHARTS-ACR.md)).
2. Copia la carpeta de la tienda (ej: `tienda-pam-ejemplo`) a `atlas-stores/fleet/bundles/stores/`.
3. Para una tienda real: renombra la carpeta al ID (ej: `gasparhernandez`) y dentro de `fleet.yaml` sustituye todo `tienda-pam-ejemplo` por `gasparhernandez` (clusterSelector y hostnames).
4. Ajusta contraseñas/secretos (SOPS/Sealed Secrets); no commitear en claro.
5. En Rancher, asigna al cluster las labels `atlas: "true"` y `store: <id-tienda>`.
