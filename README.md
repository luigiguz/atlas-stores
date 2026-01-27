# Atlas Stores - Configuración por Tienda

Este repositorio contiene la configuración específica de cada tienda para el despliegue con Fleet. Cada tienda tiene su propio bundle de Fleet que se despliega automáticamente según los labels del cluster.

## Estructura

```
atlas-stores/
├── stores/
│   ├── <nombre-tienda>/
│   │   ├── fleet.yaml              # Bundle de Fleet para esta tienda
│   │   ├── values-common.yaml      # Configuración común
│   │   ├── values-db.yaml          # Configuración de base de datos
│   │   ├── values-core.yaml        # Configuración Core
│   │   ├── values-pam.yaml         # Configuración PAM (si aplica)
│   │   ├── values-horustech.yaml   # Configuración Horustech (si aplica)
│   │   └── secrets.yaml            # Secretos (puede usar SOPS opcionalmente)
│   └── ejemplo-tienda/             # Template completo para nuevas tiendas
│       ├── fleet.yaml              # Template de bundle Fleet
│       ├── values-common.yaml      # Template de valores comunes
│       ├── values-db.yaml          # Template de valores BD
│       ├── values-core.yaml        # Template de valores Core
│       ├── values-pam.yaml         # Template de valores PAM
│       ├── values-horustech.yaml   # Template de valores Horustech
│       └── secrets.sops.yaml.example  # Template de secretos
└── groups/
    ├── core/                       # Template genérico para Core
    ├── horustech/                  # Template genérico para Horustech
    └── pam/                        # Template genérico para PAM
```

## Crear Configuración para una Nueva Tienda

### 1. Crear directorio de la tienda

```bash
mkdir -p stores/<nombre-tienda>
```

### 2. Copiar templates completos

```bash
# Copiar todos los archivos del template
cp -r stores/ejemplo-tienda/* stores/<nombre-tienda>/

# O copiar archivos individuales
cp stores/ejemplo-tienda/fleet.yaml stores/<nombre-tienda>/
cp stores/ejemplo-tienda/values-*.yaml stores/<nombre-tienda>/
cp stores/ejemplo-tienda/secrets.sops.yaml.example stores/<nombre-tienda>/secrets.yaml
```

### 3. Editar fleet.yaml

Editar `stores/<nombre-tienda>/fleet.yaml` y reemplazar todas las ocurrencias de `<nombre-tienda>` con el nombre real de la tienda:

```bash
# Reemplazar en el archivo (Linux/macOS)
sed -i 's/<nombre-tienda>/<nombre-real>/g' stores/<nombre-tienda>/fleet.yaml

# O en Windows PowerShell
(Get-Content stores/<nombre-tienda>/fleet.yaml) -replace '<nombre-tienda>', '<nombre-real>' | Set-Content stores/<nombre-tienda>/fleet.yaml
```

**Reemplazos necesarios:**
- `store: "<nombre-tienda>"` → `store: "<nombre-real>"`
- `name: <nombre-tienda>-db` → `name: <nombre-real>-db`
- `name: <nombre-tienda>-core` → `name: <nombre-real>-core`
- Y así para todos los bundles

**Importante:** Si la tienda solo usa Core (sin Horustech ni PAM):
- Eliminar las secciones `horustech` y `pam` del `fleet.yaml`
- Eliminar los archivos `values-horustech.yaml` y `values-pam.yaml` (opcional)

Si usa Horustech, eliminar la sección `pam` y el archivo `values-pam.yaml`.  
Si usa PAM, eliminar la sección `horustech` y el archivo `values-horustech.yaml`.

### 4. Editar valores

Editar los archivos `values-*.yaml` con la configuración específica de la tienda:

- `values-common.yaml`: Configuración común (registry, timezone, recursos, etc.)
- `values-db.yaml`: Configuración de PostgreSQL
- `values-core.yaml`: Configuración de Core (Portal y WebAPI)
- `values-horustech.yaml` o `values-pam.yaml`: Configuración específica del sistema POS (si aplica)

### 5. Crear secretos

**Opción A: Sin encriptación (Recomendado para empezar)**

```bash
# Copiar template
cp stores/ejemplo-tienda/secrets.sops.yaml.example stores/<nombre-tienda>/secrets.yaml

# Editar secretos directamente
vim stores/<nombre-tienda>/secrets.yaml

# ¡Listo! Fleet leerá el archivo directamente
```

**Opción B: Con SOPS (Opcional - solo si necesitas encriptación)**

```bash
# Copiar template
cp stores/ejemplo-tienda/secrets.sops.yaml.example stores/<nombre-tienda>/secrets.sops.yaml

# Editar secretos (se desencripta automáticamente)
sops stores/<nombre-tienda>/secrets.sops.yaml

# Guardar (se encripta automáticamente)
```

**Nota:** Fleet NO desencripta SOPS automáticamente. Si usas SOPS, necesitarás desencriptar manualmente o usar Sealed Secrets/External Secrets Operator.

### 6. Aplicar labels al cluster

En Rancher, aplicar los siguientes labels al cluster:

```yaml
atlas: "true"
store: "<nombre-tienda>"
poslite: "horustech"  # o "pam" o "core"
```

**Cómo aplicar labels en Rancher:**
1. Ve al cluster
2. Click en **"⋮" (menú)** → **"Edit Config"**
3. En la sección **"Labels"**, agrega los labels arriba
4. Guardar

### 7. Commit y Push

```bash
git add stores/<nombre-tienda>/
git commit -m "Agregar nueva tienda: <nombre-tienda>"
git push
```

Fleet detectará automáticamente el nuevo bundle y lo desplegará en el cluster con los labels correspondientes.

## Gestión de Secretos

### Opción 1: Archivos YAML sin Encriptar (Recomendado)

Fleet puede leer archivos `secrets.yaml` directamente sin necesidad de encriptación:

```bash
# Crear archivo de secretos
cp stores/ejemplo-tienda/secrets.sops.yaml.example stores/<tienda>/secrets.yaml

# Editar directamente
vim stores/<tienda>/secrets.yaml

# Commit y push - Fleet lo aplicará automáticamente
git add stores/<tienda>/secrets.yaml
git commit -m "Agregar secretos para tienda"
git push
```

**Ventajas:**
- ✅ Simple y directo
- ✅ Fleet lo maneja automáticamente
- ✅ No requiere configuración adicional

**Desventajas:**
- ⚠️ Secretos visibles en el repositorio Git
- ⚠️ Cualquiera con acceso al repo puede verlos

### Opción 2: SOPS (Opcional - Solo si necesitas encriptación)

Si necesitas encriptar los secretos en Git, puedes usar SOPS:

#### Instalación de SOPS

```bash
# Linux
wget https://github.com/mozilla/sops/releases/download/v3.8.0/sops-v3.8.0.linux
sudo mv sops-v3.8.0.linux /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# macOS
brew install sops
```

#### Configuración de SOPS

Crear archivo `.sops.yaml` en la raíz del repositorio:

```yaml
creation_rules:
  - path_regex: stores/.*/secrets\.sops\.yaml$
    pgp: >-
      FINGERPRINT_DE_LA_CLAVE_PGP
```

#### Uso de SOPS

```bash
# Editar archivo encriptado
sops stores/<tienda>/secrets.sops.yaml

# Encriptar archivo
sops -e -i stores/<tienda>/secrets.sops.yaml

# Desencriptar para ver contenido
sops -d stores/<tienda>/secrets.sops.yaml
```

**⚠️ Importante:** Fleet NO desencripta SOPS automáticamente. Si usas SOPS, necesitarás:
- Desencriptar manualmente antes de aplicar, O
- Usar Sealed Secrets Operator, O
- Usar External Secrets Operator

**Recomendación:** Para empezar, usa archivos `secrets.yaml` sin encriptar. Puedes migrar a SOPS más adelante si es necesario.

## Variables Importantes

### Base de Datos

- `postgresql.password`: Contraseña de PostgreSQL
- `postgresql.user`: Usuario de PostgreSQL (por defecto: `sa`)
- `postgresql.database`: Nombre de la base de datos (por defecto: `poslite`)
- `db-connection-string`: Cadena de conexión completa

### Core

- `portal.enabled`: Habilitar Portal (hostPort: 10014)
- `webapi.enabled`: Habilitar WebAPI (hostPort: 10012)
- `config.coreApiKey`: API Key para Core
- `config.ierpUrl`: URL del gateway iERP

### PAM

- `pam.password`: Contraseña de acceso al sistema PAM
- `pam.ip`: IP del sistema PAM
- `config.pamIp`: IP del sistema PAM (alternativa)

### Horustech

- `horustech.ip`: IP principal del sistema Horustech
- `horustech.portWebapi`: Puerto WebAPI
- `horustech.portWorkers`: Puerto Workers
- `horustech.tanksIp`: IP para tanques (opcional)
- `config.horustechIp`: IP principal (alternativa)

### Cloudflared

- `cloudflared.tunnel-id`: ID del tunnel de Cloudflare
- `cloudflared.credentials`: Credenciales del tunnel (JSON)

## Integración con Fleet

Fleet leerá automáticamente los archivos de este repositorio y aplicará la configuración según los labels de los clusters.

### Cómo Funciona

1. **Cada tienda tiene su propio bundle**: `stores/<tienda>/fleet.yaml`
2. **Fleet detecta automáticamente** los bundles en subdirectorios
3. **Los bundles se despliegan** cuando el cluster tiene el label `store: "<tienda>"`
4. **Los valores se cargan** desde archivos relativos en el mismo directorio

### Orden de Precedencia

1. Valores del chart (defaults)
2. Valores del bundle en `stores/<tienda>/fleet.yaml` (valores inline)
3. Valores de `valuesFiles` en el bundle (archivos YAML de la tienda)

## Mejores Prácticas

1. **Gestionar secretos según tu necesidad**
   - Si el repositorio es privado y confías en el equipo: usar `secrets.yaml` sin encriptar
   - Si necesitas encriptación: usar SOPS o Sealed Secrets

2. **Usar valores por defecto cuando sea posible**
   - Solo sobrescribir valores que sean específicos de la tienda

3. **Documentar cambios importantes**
   - Comentar en commits cuando se cambian configuraciones críticas

4. **Validar antes de mergear**
   - Verificar que los valores YAML sean válidos
   - Si usas SOPS, verificar que los secretos estén encriptados

5. **Usar branches para cambios grandes**
   - Crear branches para cambios que afecten múltiples tiendas

## Troubleshooting

### Problema: Fleet no aplica cambios

**Solución**:
1. Verificar que el repositorio esté registrado en Rancher
2. Verificar que los labels del cluster sean correctos (`atlas: "true"`, `store: "<tienda>"`)
3. Verificar logs de Fleet: `kubectl logs -n fleet-system -l app=fleet-controller`
4. Verificar bundles: `kubectl get bundles.fleet.cattle.io -n fleet-default`

### Problema: Secretos no se aplican

**Solución**:
1. Verificar que el archivo `secrets.yaml` exista en la carpeta de la tienda
2. Verificar que el archivo tenga sintaxis YAML válida
3. Verificar que Fleet tenga permisos para leer el Secret
4. Si usas SOPS: verificar que el archivo esté desencriptado o que uses Sealed Secrets

### Problema: Valores no se aplican correctamente

**Solución**:
1. Verificar orden de precedencia
2. Verificar sintaxis YAML
3. Verificar que los paths en `valuesFiles` sean relativos al `fleet.yaml`
4. Verificar que los archivos de valores existan en el directorio de la tienda

### Problema: Bundle no se despliega

**Solución**:
1. Verificar que el `fleet.yaml` esté en `stores/<tienda>/fleet.yaml`
2. Verificar que el cluster tenga el label `store: "<tienda>"`
3. Verificar que el `clusterSelector` en el bundle coincida con los labels del cluster
4. Ver logs: `kubectl describe bundle.fleet.cattle.io <nombre-bundle> -n fleet-default`

## Contacto

Para problemas o consultas, contactar al equipo de DevOps.
