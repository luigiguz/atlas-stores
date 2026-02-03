# Plantillas de tienda para atlas-stores (un bundle por chart)

Cada **tienda** es una carpeta con **una subcarpeta por chart**. Fleet descubre un bundle por cada subcarpeta que tenga `fleet.yaml`, así se despliegan todos los charts de la tienda en el mismo cluster.

El label **`store` es obligatorio**: cada cluster debe tener `atlas: "true"` y `store: "<nombre-tienda>"` para recibir el despliegue de esa tienda.

## Estructura de una tienda

```
stores/
└── <nombre-tienda>/              # ej: posdemos
    ├── db/
    │   └── fleet.yaml            # chart poslite-db
    ├── cloudflared-<tipo>/       # cloudflared-core, cloudflared-horustech o cloudflared-pam
    │   └── fleet.yaml
    └── <pos>/                    # core, horus (Horustech) o pam (según tipo de tienda)
        └── fleet.yaml
```

Cada `fleet.yaml` tiene **un solo chart** y el mismo `clusterSelector` (`atlas: "true"`, `store: "<nombre-tienda>"`). Los valores van **inline** en cada `fleet.yaml`; no se usan archivos `values-*.yaml` separados.

## Tipos de tienda

| Tipo | Subcarpetas (charts) | Labels del cluster |
|------|----------------------|--------------------|
| **Solo Core** | `db/`, `core/`, `cloudflared-core/` | `atlas: "true"`, `store: "<nombre-tienda>"` |
| **Horustech** | `db/`, `cloudflared-horustech/`, `horus/` | `atlas: "true"`, `store: "<nombre-tienda>"` |
| **PAM** | `db/`, `cloudflared-pam/`, `pam/` | `atlas: "true"`, `store: "<nombre-tienda>"` |

PAM y Horustech **no desplegan poslite-core** (solo db + cloudflared-xxx + pam/horustech).

**Hostnames:** `<nombre-tienda>-<puerto>.asptienda.com`. En cada `fleet.yaml` de cloudflared hay que actualizar los hostnames al renombrar la tienda.

## Ejemplo: tienda Horustech (posdemos)

```
stores/posdemos/
├── db/fleet.yaml
├── cloudflared-horustech/fleet.yaml
└── horus/fleet.yaml
```

La carpeta del chart poslite-horustech se llama **horus** (no horustech) para que los nombres de recursos no superen el límite de 63 caracteres de Kubernetes.

Fleet crea 3 bundles: `stores/posdemos/db`, `stores/posdemos/cloudflared-horustech`, `stores/posdemos/horus`. Los tres se aplican al cluster con label `store: "posdemos"`.

## Uso

1. Asegúrate de que los charts estén publicados en ACR.
2. Copia la plantilla de tienda (ej: `tienda-horustech-ejemplo`) y renómbrala al ID de la tienda (ej: `posdemos`).
3. Dentro de cada subcarpeta, en el `fleet.yaml` sustituye el nombre de la plantilla por el nombre de la tienda (`clusterSelector.store` y hostnames en cloudflared).
4. Ajusta contraseñas/secretos (SOPS/Sealed Secrets); no commitear en claro.
5. En Rancher, asigna al cluster las labels `atlas: "true"` y `store: <id-tienda>`.

## Por qué una carpeta por chart

Fleet genera **un bundle por cada directorio que contiene un fleet.yaml**. Si en un solo `fleet.yaml` se definen varios charts (`targetCustomizations`), en algunas versiones solo se despliega el primero. Con **una subcarpeta por chart** (cada una con su `fleet.yaml`), Fleet crea un bundle por chart y todos se despliegan en el cluster que coincida con el selector.
