# Atlas Stores - Configuración por Tienda

Este repositorio contiene la configuración específica de cada tienda para el despliegue con Fleet.

## Estructura

```
atlas-stores/
├── stores/
│   └── <nombre-tienda>/
│       ├── values-common.yaml      # Configuración común
│       ├── values-db.yaml          # Configuración de base de datos
│       ├── values-pam.yaml         # Configuración PAM (si aplica)
│       ├── values-horustech.yaml   # Configuración Horustech (si aplica)
│       └── secrets.yaml             # Secretos (puede usar SOPS opcionalmente)
└── groups/
    ├── pilot.yaml                  # Grupo de tiendas piloto
    ├── wave1.yaml                  # Primera ola de migración
    └── wave2.yaml                  # Segunda ola de migración
```

## Crear Configuración para una Nueva Tienda

### 1. Crear directorio de la tienda

```bash
mkdir -p stores/<nombre-tienda>
```

### 2. Copiar templates

```bash
cp stores/ejemplo-tienda/values-*.yaml stores/<nombre-tienda>/
```

### 3. Editar valores

Editar los archivos `values-*.yaml` con la configuración específica de la tienda.

### 4. Crear secretos

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

### 5. Aplicar labels al cluster

En Rancher, aplicar los siguientes labels al cluster:

```yaml
atlas: "true"
store: "<nombre-tienda>"
poslite: "pam"  # o "horustech"
wave: "pilot"   # o "wave1", "wave2"
```

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
- `db-connection-string`: Cadena de conexión completa

### PAM

- `pam.password`: Contraseña de acceso al sistema PAM
- `pam.ip`: IP del sistema PAM

### Horustech

- `horustech.ip`: IP principal del sistema Horustech
- `horustech.portWebapi`: Puerto WebAPI
- `horustech.portWorkers`: Puerto Workers
- `horustech.tanksIp`: IP para tanques (opcional)

### Cloudflared

- `cloudflared.tunnel-id`: ID del tunnel de Cloudflare
- `cloudflared.credentials`: Credenciales del tunnel (JSON)

## Integración con Fleet

Fleet leerá automáticamente los archivos de este repositorio y aplicará la configuración según los labels de los clusters.

### Orden de Precedencia

1. Valores del chart (defaults)
2. Valores del bundle en `atlas-apps`
3. Valores del grupo en `groups/*.yaml`
4. Valores específicos de la tienda en `stores/<tienda>/*.yaml`

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
2. Verificar que los labels del cluster sean correctos
3. Verificar logs de Fleet: `kubectl logs -n fleet-system -l app=fleet-controller`

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
3. Verificar que los paths en los valores sean correctos

## Contacto

Para problemas o consultas, contactar al equipo de DevOps.
