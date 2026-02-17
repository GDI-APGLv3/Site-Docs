# Middleware

El backend usa dos middleware principales: **CORS** y **TenantMiddleware**.

## Orden de Middleware

Los middleware se ejecutan en orden inverso al registro (el ultimo registrado se ejecuta primero):

```python
# main.py
app.add_middleware(CORSMiddleware, ...)   # Se ejecuta segundo
app.add_middleware(TenantMiddleware)       # Se ejecuta primero
```

## CORS Middleware

Configurado para permitir acceso desde frontends en desarrollo y produccion.

```python
# main.py
allowed_origins = (
    [f"http://localhost:{port}" for port in range(3000, 3051)] +
    [f"http://127.0.0.1:{port}" for port in range(3000, 3051)] +
    [f"http://localhost:{port}" for port in range(8000, 8051)] +
    [f"http://127.0.0.1:{port}" for port in range(8000, 8051)]
)

# Agregar frontend de Railway si existe
railway_frontend = os.getenv("FRONTEND_URL")
if railway_frontend:
    allowed_origins.append(railway_frontend)

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## TenantMiddleware

Middleware critico que valida el acceso multi-tenant en cada request. Definido en `middleware/tenant_middleware.py`.

### Flujo de Validacion

```
Request entrante
    |
    v
1. OPTIONS? -> Skip (CORS preflight)
    |
    v
2. Ruta publica? -> Skip (/health, /docs, /openapi.json)
    |
    v
3. TESTING_MODE?
    |  SI -> Leer X-Tenant-Schema (default: "100_test")
    |         Buscar usuario por X-User-ID, Bearer UUID, o JWT email
    |         Guardar en request.state
    |
    v
4. Extraer email del JWT (Auth0)
    |  Fallback 1: Header X-User-Email
    |  Fallback 2: Buscar por auth_id (sub) en el schema
    |
    v
5. Leer header X-Tenant-Schema
    |  Requerido para requests autenticados
    |
    v
6. Validar acceso del usuario al schema (con cache)
    |
    v
7. Validar schema contra whitelist (SQL injection prevention)
    |
    v
8. Buscar usuario en schema WHERE email=$1
    |  Activacion automatica si estado=0 y auth_id=PENDING_*
    |
    v
9. Guardar en request.state:
    - schema_name
    - tenant_user_id
    - tenant_email
    - auth_source ("jwt" o "testing")
    - correlation_id
```

### Rutas Excluidas

Las siguientes rutas no requieren validacion de tenant:

```python
EXCLUDED_PATHS = {
    "/health",
    "/api/v1/system/health",
    "/favicon.ico",
    "/docs",
    "/openapi.json",
    "/redoc",
    "/api/auth/onboarding",
}
```

### request.state

El TenantMiddleware inyecta estos valores en `request.state` para uso en endpoints:

| Atributo | Tipo | Descripcion |
|----------|------|-------------|
| `schema_name` | `str` | Schema PostgreSQL del tenant (ej: `"100_test"`) |
| `tenant_user_id` | `str` | UUID del usuario en el schema |
| `tenant_email` | `str` | Email del usuario |
| `auth_source` | `str` | `"jwt"` o `"testing"` |
| `correlation_id` | `str` | ID para trazabilidad entre servicios |

### Correlation ID

Cada request recibe un correlation ID unico para trazabilidad:

- Si viene en header `X-Correlation-ID`: se reutiliza (propagacion entre servicios)
- Si no viene: se genera uno nuevo automaticamente
- Se agrega al response header `X-Correlation-ID`

### Activacion Automatica de Usuarios

Cuando un usuario con `estado=0` y `auth_id=PENDING_*` hace login por primera vez:

1. El middleware detecta el estado pendiente
2. Extrae el `auth_id` real del JWT (`sub`)
3. Actualiza `estado=1`, `auth_id`, `full_name`, `profile_picture_url`
4. El usuario queda activo automaticamente

### Auto-completar auth_id

Si un usuario existente no tiene `auth_id` (migrado o creado manualmente) pero envia un JWT valido, el middleware auto-completa el `auth_id` en la BD.

### Seguridad

!!! warning "Validacion de Schema"
    El schema_name se valida contra una whitelist usando `is_valid_schema()` para prevenir SQL injection. Solo se permiten caracteres alfanumericos y guion bajo.

!!! warning "TESTING_MODE en Produccion"
    Si `TESTING_MODE=true` se detecta en un entorno de Railway con nombre "production" o "prod", se desactiva automaticamente por seguridad.
