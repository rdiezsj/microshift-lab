
# MicroShift Lab: Full-Stack GitOps (App of Apps)

Este repositorio contiene una prueba de concepto (PoC) para desplegar una arquitectura Full-Stack segura sobre **Red Hat MicroShift**, gestionada mediante **ArgoCD** bajo el patrón **App of Apps**.

## 🏗 Arquitectura del Repositorio

Hemos evolucionado el laboratorio de un despliegue monolítico a una estructura jerárquica gestionada por una aplicación raíz.

### Jerarquía de Aplicaciones (ArgoCD)
1.  **Root App (`microshift-lab-root`):** La aplicación principal que orquesta el despliegue. Monitorea la carpeta `argocd-apps/`.
2.  **Aplicaciones Hijas:**
    * `lab-database`: Despliega PostgreSQL.
    * `lab-iam`: Despliega Keycloak.
    * `lab-backend`: Despliega PostgREST.
    * `lab-frontend`: Despliega Swagger UI.

### Estructura de Directorios

```
.
├── argocd-apps/           # Definiciones de Application para ArgoCD
├── apps/
│   └── full-stack/        # Manifiestos K8s organizados por componente
│       ├── database/      # PostgreSQL + Init Scripts
│       ├── iam/           # Keycloak (OIDC IdP)
│       ├── backend/       # PostgREST API
│       └── frontend/      # Swagger UI con OAuth2
└── microshift-ca.crt      # CA del clúster para confianza local

````


## 🚀 Despliegue con ArgoCD

Para iniciar el laboratorio, solo necesitas aplicar la aplicación raíz en el namespace donde reside ArgoCD (en este entorno: `argocd`):

```sh
oc apply -f argocd-apps/root-app.yaml -n argocd
```

ArgoCD detectará automáticamente los componentes en `argocd-apps/` y creará el namespace `full-stack` si no existe gracias a la política `CreateNamespace=true`.

---

## ⚙️ Configuración Manual (Post-Despliegue)

Debido a la naturaleza del laboratorio, se requieren pasos manuales tras la primera sincronización:

### 1. Confianza en Certificados

Extrae e instala la CA para evitar errores de CORS:

```sh
oc get secret ingress-ca -n kube-system -o jsonpath='{.data.tls\.crt}' | base64 -d > microshift-ca.crt
sudo cp microshift-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### 2. Roles de Base de Datos

Accede al pod de Postgres y configura los roles de seguridad:

```sh
oc rsh -n full-stack deployment/postgres psql -U app_user -d app_db
```

Ejecuta:

```sql
CREATE ROLE todo_user NOLOGIN;
GRANT USAGE ON SCHEMA public TO todo_user;
GRANT ALL ON tasks TO todo_user;
GRANT USAGE, SELECT ON SEQUENCE tasks_id_seq TO todo_user;
```

### 3. Sincronización de JWT Secret

PostgREST necesita la clave pública de Keycloak.

1. Obtén las llaves en: `https://keycloak-route-full-stack.apps.lab.local/realms/lab-realm/protocol/openid-connect/certs`.
    
2. Actualiza el campo `PGRST_JWT_SECRET` en `apps/full-stack/backend/deployment.yaml` con el JSON obtenido.
    
3. Haz push de los cambios; ArgoCD sincronizará solo el backend automáticamente.
    
---

## ✅ Validación

1. Accede al Frontend: `https://frontend-route-full-stack.apps.lab.local`.
    
2. Autentícate con `testuser` mediante el botón **Authorize** (Implicit Flow).
    
3. Ejecuta `GET /tasks` para verificar la conexión backend-DB.