# Autenticacion

El backend usa **Auth0** con tokens **JWT RS256** para autenticacion. Definido en `auth.py`.

## Configuracion Auth0

| Variable | Valor Default |
|----------|--------------|
| `AUTH0_DOMAIN` | `tu-tenant.us.auth0.com` |
| `AUTH0_AUDIENCE` | Configurado en `.env` |
| `AUTH0_ALGORITHMS` | `["RS256"]` |

## Flujo de Autenticacion

### Modo Produccion

```
1. Frontend obtiene JWT de Auth0 (login)
2. Frontend envia request con header:
   Authorization: Bearer <jwt_token>
   X-Tenant-Schema: 100_san_miguel
3. TenantMiddleware valida email del JWT contra el schema
4. auth.py valida JWT y obtiene AuthenticatedUser
5. Endpoint recibe current_user con permisos
```

### Modo Testing

Si `TESTING_MODE=true` en `.env`, el backend acepta autenticacion simplificada:

| Metodo | Header | Descripcion |
|--------|--------|-------------|
| API Key fija | `X-API-Key: <TESTING_API_KEY>` | Usa primer usuario activo |
| UUID directo | `X-User-ID: <uuid>` | UUID del usuario |
| Bearer UUID | `Authorization: Bearer <uuid>` | UUID en lugar de JWT |
| JWT normal | `Authorization: Bearer <jwt>` | Funciona igual que produccion |

!!! warning "Seguridad"
    `TESTING_MODE` se desactiva automaticamente si se detecta entorno de produccion en Railway.

## Funciones Principales

### get_current_user

Dependency principal de FastAPI. Se usa en todos los endpoints protegidos.

```python
# auth.py
def get_current_user(
    request: Request,
    credentials: Optional[HTTPAuthorizationCredentials] = Depends(security),
    x_user_id: Optional[str] = Header(None, alias="X-User-ID")
) -> AuthenticatedUser:
```

**Retorna** un `AuthenticatedUser` con:

- `user_id`: UUID del usuario en la BD
- `auth_id`: ID de Auth0 (`sub` del JWT)
- `full_name`: Nombre completo
- `email`: Email
- `permissions`: Lista de `SectorPermission`

**Uso en endpoints:**

```python
@router.get("/documents")
async def list_documents(
    current_user: AuthenticatedUser = Depends(get_current_user),
    schema_name: str = Depends(get_tenant_schema)
):
    # current_user.user_id, current_user.permissions, etc.
    ...
```

### verify_token

Verifica y decodifica un JWT de Auth0.

```python
def verify_token(token: str) -> Dict[str, Any]:
```

Proceso:

1. Obtiene la clave RSA publica de Auth0 (JWKS) con cache de 30 minutos
2. Busca la clave correspondiente al `kid` del token
3. Decodifica y verifica el JWT con audience e issuer

**Errores:**

| HTTP | Detalle |
|------|---------|
| 401 | Token expirado |
| 401 | Claims incorrectos (audience/issuer) |
| 401 | Token invalido |
| 503 | No se pudo obtener claves de Auth0 |

### verify_auth0_token

Valida JWT sin requerir que el usuario exista en BD. Util para onboarding.

```python
def verify_auth0_token(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> Dict[str, Any]:
```

### load_user_permissions

Carga permisos de sectores del usuario desde la BD.

```python
def load_user_permissions(user_id: str, *, schema_name: str) -> list:
```

Retorna lista de `SectorPermission` con informacion de cada sector donde el usuario tiene acceso.

### decode_jwt_from_request

Extrae y valida JWT del header Authorization. Usado internamente por el TenantMiddleware.

```python
def decode_jwt_from_request(request) -> Dict[str, Any]:
```

## Cache de JWKS

Las claves publicas de Auth0 se cachean por **30 minutos** para evitar consultas frecuentes:

```python
_jwks_cache = None
_jwks_cache_expiry = None  # datetime.now() + timedelta(minutes=30)
```

## Permisos de Sectores

El sistema de permisos se basa en sectores. Cada usuario puede tener acceso a multiples sectores con diferentes niveles:

| Permiso | Descripcion |
|---------|-------------|
| `can_view` | Ver documentos y expedientes del sector |
| `can_edit` | Crear y editar documentos en el sector |
| `is_primary` | Sector principal del usuario |

Los permisos se cargan desde la tabla `user_sector_permissions` al autenticar al usuario y se incluyen en el `AuthenticatedUser`.

## Dependencies de FastAPI

Definidas en `shared/dependencies.py`:

```python
def get_tenant_schema(request: Request) -> str:
    """Extrae schema_name del request.state (seteado por TenantMiddleware)."""
    return request.state.schema_name

def get_auth_source(request: Request) -> str:
    """Extrae auth_source del request.state."""
    return getattr(request.state, 'auth_source', 'unknown')
```
