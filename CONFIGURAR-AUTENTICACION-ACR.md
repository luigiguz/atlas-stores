# Configurar Autenticación ACR para Fleet

## 🔐 Problema

El ACR `atlashelmrepo.azurecr.io` es privado y requiere autenticación. Fleet necesita credenciales para descargar los charts.

**Error actual:**
```
401: unauthorized: authentication required
```

---

## ✅ Solución: Crear Secret de Autenticación

### Paso 1: Crear Secret en Kubernetes

Ejecuta este comando en el cluster donde está Fleet (normalmente el cluster de Rancher):

```bash
# Crear Secret de tipo docker-registry para ACR
kubectl create secret docker-registry helm-acr-credentials \
  --docker-server=atlashelmrepo.azurecr.io \
  --docker-username=atlashelmrepo \
  --docker-password=<TU_PASSWORD_ACR> \
  -n fleet-default
```

**Obtener el password del ACR:**

```bash
# Opción 1: Desde Azure Portal
# Ve a: Azure Portal > Container Registries > atlashelmrepo > Access keys
# Copia el "password" (puede ser password1 o password2)

# Opción 2: Con Azure CLI
az acr credential show --name atlashelmrepo --query "passwords[0].value" -o tsv
```

### Paso 2: Configurar Autenticación en Fleet

Tienes **dos opciones**:

---

## Opción A: Configurar en el GitRepo (Recomendado)

Esta opción aplica la autenticación a **todos los bundles** del repositorio.

### En Rancher UI:

1. Ve a **Fleet > Git Repos**
2. Selecciona el repositorio `atlas-stores`
3. Click en **"Edit Config"** o **"⋮" → Edit**
4. En la sección **"Helm Chart Credentials"** o **"Secrets"**, agrega:
   - **Secret Name**: `helm-acr-credentials`
   - **Namespace**: `fleet-default`

### O con kubectl:

```bash
# Editar el GitRepo
kubectl edit gitrepo.fleet.cattle.io atlas-stores -n fleet-default
```

Agregar en la sección `spec:`:

```yaml
spec:
  # ... otras configuraciones ...
  helmSecretName: helm-acr-credentials  # ← Agregar esto
```

---

## Opción B: Configurar en cada fleet.yaml

Si prefieres configurarlo por bundle, agrega en cada `fleet.yaml`:

```yaml
helm:
  chart: oci://atlashelmrepo.azurecr.io/helm/poslite-db
  version: 1.0.0
  repoCredentialSecretName: helm-acr-credentials  # ← Agregar esto
  values:
    # ... valores ...
```

**Nota:** Para OCI registries, Fleet puede usar el secret a nivel del GitRepo, así que la Opción A es más simple.

---

## 🔍 Verificar que Funciona

### 1. Verificar que el Secret existe:

```bash
kubectl get secret helm-acr-credentials -n fleet-default

# Deberías ver:
# NAME                     TYPE                  DATA   AGE
# helm-acr-credentials     kubernetes.io/dockerconfigjson   1   1m
```

### 2. Verificar que Fleet puede acceder:

```bash
# Ver logs de Fleet
kubectl logs -n fleet-system -l app=fleet-controller --tail=50 | grep -i "atlashelmrepo"

# No debería haber errores 401
```

### 3. Verificar bundles:

```bash
kubectl get bundles.fleet.cattle.io -n fleet-default

# Los bundles deberían estar en estado "Ready" (no "Error")
```

---

## 🚨 Si el Secret no Funciona

### Verificar el tipo de Secret:

Para OCI registries, el Secret debe ser de tipo `docker-registry`:

```bash
# Ver el tipo del Secret
kubectl get secret helm-acr-credentials -n fleet-default -o yaml | grep type

# Debe mostrar:
# type: kubernetes.io/dockerconfigjson
```

### Si necesitas recrear el Secret:

```bash
# Eliminar el Secret existente
kubectl delete secret helm-acr-credentials -n fleet-default

# Crear de nuevo con el tipo correcto
kubectl create secret docker-registry helm-acr-credentials \
  --docker-server=atlashelmrepo.azurecr.io \
  --docker-username=atlashelmrepo \
  --docker-password=<PASSWORD> \
  -n fleet-default
```

---

## 📝 Comando Completo (Copia y Pega)

```bash
# 1. Obtener password del ACR
ACR_PASSWORD=$(az acr credential show --name atlashelmrepo --query "passwords[0].value" -o tsv)

# 2. Crear Secret
kubectl create secret docker-registry helm-acr-credentials \
  --docker-server=atlashelmrepo.azurecr.io \
  --docker-username=atlashelmrepo \
  --docker-password="$ACR_PASSWORD" \
  -n fleet-default

# 3. Verificar
kubectl get secret helm-acr-credentials -n fleet-default

# 4. Configurar en GitRepo (si usas Opción A)
kubectl patch gitrepo.fleet.cattle.io atlas-stores -n fleet-default \
  --type merge -p '{"spec":{"helmSecretName":"helm-acr-credentials"}}'
```

---

## ✅ Checklist

- [ ] Secret `helm-acr-credentials` creado en `fleet-default`
- [ ] Secret es de tipo `docker-registry` o `kubernetes.io/dockerconfigjson`
- [ ] GitRepo configurado con `helmSecretName` (Opción A) O cada fleet.yaml tiene `repoCredentialSecretName` (Opción B)
- [ ] Charts publicados en el ACR
- [ ] Fleet puede acceder sin errores 401

---

## 🔄 Actualizar Password del ACR

Si cambias el password del ACR:

```bash
# Obtener nuevo password
NEW_PASSWORD=$(az acr credential show --name atlashelmrepo --query "passwords[0].value" -o tsv)

# Actualizar Secret
kubectl delete secret helm-acr-credentials -n fleet-default

kubectl create secret docker-registry helm-acr-credentials \
  --docker-server=atlashelmrepo.azurecr.io \
  --docker-username=atlashelmrepo \
  --docker-password="$NEW_PASSWORD" \
  -n fleet-default
```

---

## 🆘 Troubleshooting

### Error: "secret not found"
**Solución:** Verificar que el Secret existe: `kubectl get secret helm-acr-credentials -n fleet-default`

### Error: "unauthorized" después de crear el Secret
**Solución:** 
1. Verificar que el password es correcto
2. Verificar que el Secret es de tipo `docker-registry`
3. Verificar que el GitRepo tiene `helmSecretName` configurado

### Error: "wrong secret type"
**Solución:** Usar `docker-registry` en lugar de `generic`:
```bash
kubectl create secret docker-registry ...  # ← No "generic"
```
