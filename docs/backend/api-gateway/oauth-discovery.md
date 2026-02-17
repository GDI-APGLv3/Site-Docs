# OAuth Discovery

Endpoints de descubrimiento OAuth 2.0 que permiten a clientes MCP descubrir automaticamente como autenticarse.

**Ubicacion:** `api_gateway/http_server.py`

## RFCs Implementados

| RFC | Endpoint | Proposito |
|-----|----------|-----------|
| RFC 9728 | `/.well-known/oauth-protected-resource` | Descubrir authorization server |
| RFC 8414 | `/.well-known/oauth-authorization-server` | Metadata OAuth completa |
| MCP Spec | `/.well-known/mcp.json` | Manifest del servidor MCP |
| OpenAPI 3.0 | `/.well-known/openapi.json` | Spec REST API (ChatGPT Actions) |

---

## Flujo de Discovery

```
Cliente MCP (Claude Code, ChatGPT, Gemini)
    |
    1. POST /mcp (sin auth)
    |
    v
Server responde 401 + WWW-Authenticate:
    Bearer resource_metadata="/.well-known/oauth-protected-resource"
    |
    2. GET /.well-known/oauth-protected-resource
    |
    v
Server responde:
    {
        "resource": "https://mcp.tu-dominio.com",
        "authorization_servers": ["https://tu-tenant.us.auth0.com"],
        "scopes_supported": ["openid", "profile", "email", "offline_access"],
        "bearer_methods_supported": ["header"]
    }
    |
    3. GET /.well-known/oauth-authorization-server  (ChatGPT especifico)
    |
    v
Server hace proxy del openid-configuration de Auth0:
    {
        "issuer": "https://tu-tenant.us.auth0.com/",
        "authorization_endpoint": "https://tu-tenant.us.auth0.com/authorize",
        "token_endpoint": "https://tu-tenant.us.auth0.com/oauth/token",
        "registration_endpoint": "https://tu-tenant.us.auth0.com/oidc/register",
        ...
    }
    |
    4. Login en Auth0 (navegador)
    |
    5. POST /mcp con Authorization: Bearer <jwt>
    |
    v
Server valida JWT, construye MCPContext, procesa tool call
```

---

## Endpoints de Discovery

### `/.well-known/oauth-protected-resource` (RFC 9728)

Indica a clientes MCP **donde autenticarse**.

```json
{
    "resource": "https://mcp.tu-dominio.com",
    "authorization_servers": [
        "https://tu-tenant.us.auth0.com"
    ],
    "scopes_supported": [
        "openid",
        "profile",
        "email",
        "offline_access"
    ],
    "bearer_methods_supported": ["header"]
}
```

### `/.well-known/oauth-authorization-server` (RFC 8414)

Metadata completa del authorization server. **Proxy** del endpoint de Auth0:

```
GET /.well-known/oauth-authorization-server
    -> Proxy a https://tu-tenant.us.auth0.com/.well-known/openid-configuration
    -> Modifica registration_endpoint para DCR
```

!!! info "ChatGPT"
    ChatGPT requiere especificamente `/.well-known/oauth-authorization-server` (no el path de OpenID). Tambien acepta el alias `/.well-known/oauth-authorization-server/mcp`.

### `/.well-known/mcp.json`

Manifest del servidor MCP:

```json
{
    "name": "GDI-Latam MCP Server",
    "version": "1.0.0",
    "description": "Servidor MCP para gestion documental gubernamental",
    "tools": ["search_cases", "get_case", "search_documents", ...],
    "authentication": {
        "type": "oauth2",
        "discovery": "/.well-known/oauth-protected-resource"
    }
}
```

### `/.well-known/openapi.json`

Especificacion OpenAPI 3.0 para la REST API. Usado por ChatGPT Actions:

```json
{
    "openapi": "3.0.0",
    "info": {
        "title": "GDI-Latam REST API",
        "version": "1.0.0"
    },
    "paths": {
        "/api/v1/cases/search": { ... },
        "/api/v1/documents/search": { ... }
    }
}
```

---

## Configuracion Auth0

Para que el flujo OAuth funcione, Auth0 debe tener:

| Requisito | Configuracion |
|-----------|--------------|
| API creada | Audience: `https://mcp.tu-dominio.com` |
| DCR habilitado | Settings > Advanced > "OIDC Dynamic Application Registration" |
| Conexion promovida | Database connection con "Domain Level" habilitado |
| Scopes | `openid profile email offline_access` |

### Audiences JWT Soportados

```python
VALID_AUDIENCES = [
    os.getenv('AUTH0_AUDIENCE'),       # Backend principal
    os.getenv('MCP_RESOURCE_URI'),     # Variable configurable
    "https://mcp.tu-dominio.com"         # Produccion hardcoded
]
```

---

## Respuesta 401 del MCP

Cuando un cliente envia un request sin token, el server responde:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://mcp.tu-dominio.com/.well-known/oauth-protected-resource"
Content-Type: application/json

{
    "error": "Authorization required",
    "hint": "Use OAuth 2.0 to authenticate"
}
```

El header `WWW-Authenticate` con `resource_metadata` es la clave para que clientes MCP descubran automaticamente el flujo OAuth.

---

## Troubleshooting

| Error | Causa | Solucion |
|-------|-------|----------|
| "Authorization required" | No hay token OAuth | Usar cliente MCP con OAuth |
| "Token invalido" | JWT expirado o mal formado | Re-autenticar via OAuth |
| "multi_tenant_selection_required" | Usuario con multiples municipalidades | Especificar `tenant_id` |
| "Audience no valido" | Token con audience incorrecto | Verificar audience en Auth0 |
| "Usuario no encontrado" | Email no existe en BD | Crear usuario en municipalidad |
| Discovery no funciona | Endpoint no accesible | Verificar CORS y rutas en http_server.py |
