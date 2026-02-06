# MicroShift Lab: Full-Stack GitOps Demo

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/rdiezsj/microshift-lab)

Este repositorio contiene una prueba de concepto (PoC) para desplegar una arquitectura Full-Stack segura sobre **Red Hat MicroShift**, gestionada mediante **ArgoCD** e integrando autenticación **OIDC** con **Keycloak**.

El objetivo es demostrar cómo integrar servicios modernos en un entorno de borde (Edge) con recursos limitados, manejando retos comunes como certificados autofirmados, CORS y terminación TLS.

## 🏗 Arquitectura

La aplicación simula una lista de tareas (To-Do App) protegida por autenticación.

```mermaid
graph TD
    User((Usuario)) -->|HTTPS| Route["OpenShift Router (Edge TLS)"]
    
    subgraph "Namespace: full-stack"
        Route -->|HTTP| Front["Frontend: Swagger UI"]
        Route -->|HTTP| Auth["IdP: Keycloak"]
        Route -->|HTTP| Back["Backend: PostgREST"]
        
        Front -->|1. Auth Flow| Auth
        Front -->|2. JWT Request| Back
        Back -->|3. Query| DB[("Database: PostgreSQL")]
        Back -.->|Verify Token| Auth
    end
    
    subgraph "Storage"
        DB --> PVC["PVC: Topolvm Provisioner"]
    end
````

### 🧩 Componentes

1. **Frontend (Swagger UI):** Interfaz personalizada que gestiona la autenticación OAuth2 (Implicit Flow) y realiza peticiones a la API. Modificada para soportar entornos de laboratorio privados.

2. **Backend (PostgREST):** Convierte la base de datos PostgreSQL directamente en una API RESTful. Valida tokens JWT firmados por Keycloak.

3. **Identity Provider (Keycloak):** Servidor de autorización OIDC. Gestiona usuarios y emite tokens con roles específicos de base de datos.

4. **Base de Datos (PostgreSQL):** Persistencia de datos con roles a nivel de fila (RLS - Row Level Security).

5. **Infraestructura:** MicroShift con almacenamiento `topolvm`.

---

## 🚀 Despliegue

Este laboratorio está diseñado para ser desplegado mediante **ArgoCD** (GitOps), aunque puede desplegarse manualmente.

### Opción A: ArgoCD (Recomendada)

Apunta una Application de ArgoCD a la carpeta `apps/full-stack` de este repositorio.

### Opción B: Manual (OC CLI)

Bash

```sh
oc apply -k apps/full-stack/
```

---

## ⚙️ Configuración Manual (Post-Despliegue)

Debido a la naturaleza de seguridad de Keycloak y Postgres, existen pasos manuales necesarios tras el primer despliegue.

### 1. Confianza en Certificados (Crítico)

Dado que MicroShift genera certificados autofirmados, debes añadir la CA del clúster a tu sistema o navegador para evitar bloqueos de CORS y "Network Errors".

**En Linux:**

Bash

```sh
# 1. Extraer CA
oc get secret ingress-ca -n kube-system -o jsonpath='{.data.tls\.crt}' | base64 -d > microshift-ca.crt

# 2. Instalar
sudo cp microshift-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### 2. Configuración de Base de Datos

El script de inicialización crea la estructura básica, pero el rol de seguridad vinculado a Keycloak debe crearse manualmente (o via job).

Bash

```sh
# Entrar al pod de Postgres
oc rsh -n full-stack deployment/postgres psql -U app_user -d app_db
```

Ejecutar en SQL:

SQL

```sh
-- Rol que coincide con el claim de Keycloak
CREATE ROLE todo_user NOLOGIN;
GRANT USAGE ON SCHEMA public TO todo_user;
GRANT ALL ON tasks TO todo_user;
GRANT USAGE, SELECT ON SEQUENCE tasks_id_seq TO todo_user;
```

### 3. Configuración de Keycloak

Accede a `https://keycloak-route-full-stack.apps.lab.local` (Admin: `admin`/`admin`).

1. **Realm:** Crear nuevo realm llamado `lab-realm`.

2. **Client:** Crear cliente `swagger-client`.

- _Valid Redirect URIs:_ `https://frontend-route-full-stack.apps.lab.local/*`
- _Web Origins:_ `+`
- _Client Authentication:_ Off (Public).
- _Authentication Flow:_ Implicit Flow habilitado.

3. **Rol Mapper (El truco):**

- Ir a _Client Scopes_ -> `swagger-client-dedicated` -> _Add Mapper_ -> _Hardcoded claim_.
- _Name:_ `postgres-role`
- _Token Claim Name:_ `role` (Esencial para PostgREST).
- _Claim Value:_ `todo_user`.

4. **Usuario:** Crear un usuario `testuser` con credenciales permanentes.

### 4. Conectar Backend con Keycloak

PostgREST necesita la clave pública de Keycloak para validar los tokens. Como Keycloak genera una nueva en cada instalación limpia:

1. Ve a: `https://keycloak-route-full-stack.apps.lab.local/realms/lab-realm/protocol/openid-connect/certs`

2. Copia el JSON completo (`{"keys": [...]}`).

3. Edita `apps/full-stack/backend/deployment.yaml` en el repositorio:

```yaml
- name: PGRST_JWT_SECRET
  value: '...PEGA_EL_JSON_AQUI...'
```

4. Haz **Push** y sincroniza ArgoCD (o reinicia el pod del backend).

---

## ✅ Validación del Funcionamiento

1. Abre el Frontend: `https://frontend-route-full-stack.apps.lab.local`.
2. **Login:** Pulsa el botón **Authorize** (candado verde). Loguéate con `testuser`.
3. **Prueba:** Despliega `GET /tasks` y pulsa **Execute**.
4. **Éxito:** Deberías recibir un código `200 OK` y un JSON con las tareas.

> **Nota:** Si ves un "spinner" cargando infinitamente, revisa que hayas aceptado el certificado SSL de la URL del backend (`api-route...`) abriéndola en una pestaña nueva.

---

## 🔧 Solución de Problemas Comunes

- **Error "Failed to fetch" o Spinner infinito:**

- El navegador no confía en el certificado del Backend. Abre la URL de la API directamente y acepta el riesgo.
- Configuración CORS incorrecta (la URL en `PGRST_SERVER_CORS_ALLOWED_ORIGINS` no debe tener `/` final).

- **Error 401 Unauthorized:**

- El JWT Secret en el backend no coincide con el de Keycloak.
- El usuario no tiene el rol `role: todo_user` en el token (revisar Mapper).

- **Etiqueta "INVALID" en Swagger:**

- El frontend está intentando usar el validador online de Swagger. Asegúrate de que `validatorUrl: null` está configurado en el ConfigMap.
