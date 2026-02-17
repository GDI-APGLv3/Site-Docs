# MCP Server

Server MCP (Model Context Protocol) con 14 tools de solo lectura. Compatible con Claude Code, ChatGPT y Gemini.

**Ubicacion:** `api_gateway/server.py`

## Tools Disponibles

### Expedientes (Cases)

| Tool | Descripcion | Parametros principales |
|------|-------------|----------------------|
| `search_cases` | Busca en contenido completo de expedientes | `search`, `page`, `status`, `date_filter` |
| `get_case` | Detalle de expediente con documentos opcionales | `case_id`, `include_documents` |
| `get_case_history` | Historial de movimientos + ai_summary | `case_id` |
| `get_case_documents` | Lista documentos oficiales y propuestos | `case_id` |
| `get_case_permissions` | Permisos del usuario sobre un expediente | `case_id` |

### Documentos

| Tool | Descripcion | Parametros principales |
|------|-------------|----------------------|
| `search_documents` | Busca en contenido completo de documentos | `search`, `page`, `status`, `doc_type` |
| `get_document` | Detalle con ai_summary (resumen IA) | `document_id` |
| `get_document_content` | Contenido HTML completo (solo oficiales) | `document_id` |
| `get_pending_signatures` | Documentos esperando firma del usuario | (ninguno) |

### Sistema

| Tool | Descripcion |
|------|-------------|
| `get_document_types` | Tipos de documentos activos (INF, DICT, CAEX, etc.) |
| `get_sectors` | Sectores y departamentos del municipio |
| `get_user_info` | Informacion del usuario actual |
| `get_case_templates` | Templates de expedientes disponibles |

### Utilidades

| Tool | Descripcion |
|------|-------------|
| `get_agent_guide` | Guia completa del sistema (usar al conectar) |

---

## Flujos Recomendados

### "Que tengo para firmar?"

```
1. get_pending_signatures
2. Responder: "Tenes 2 docs esperando tu firma: INF-xxx, DICT-xxx"
```

### "Contame sobre el expediente de la panaderia"

```
1. search_cases(search="panaderia")
2. get_case_history(case_id=...)
3. Responder con RESUMEN NARRATIVO (no lista de pasos)
```

### "Busca documentos de Juan Perez"

```
1. search_documents(search="Juan Perez")
2. Responder con lista resumida
```

---

## Implementacion

### Definicion de Tools (`server.py`)

Cada tool se define con `@server.list_tools()` y se ejecuta con `@server.call_tool()`:

```python
# Definicion
Tool(
    name="search_cases",
    description="Busca expedientes por texto...",
    inputSchema={
        "type": "object",
        "properties": {
            "search": {"type": "string", "description": "Texto de busqueda"},
            "page": {"type": "integer", "default": 1},
            "page_size": {"type": "integer", "default": 20},
            "status": {"type": "string", "enum": ["active", "inactive", "archived"]},
        }
    }
)

# Ejecucion
@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "search_cases":
        result = await cases.search_cases(ctx, **arguments)
        return [TextContent(type="text", text=json.dumps(result))]
```

### Contexto Multi-Tenant

El contexto se inyecta automaticamente desde el JWT:

```python
# auth_mcp.py
async def validate_mcp_auth(token: str) -> MCPContext:
    # 1. Validar JWT con Auth0 JWKS
    # 2. Extraer email (del token o de /userinfo)
    # 3. Buscar usuario en BD por email
    # 4. Construir MCPContext con user_id, municipality_id, schema_name
    return MCPContext(
        user_id=user['id'],
        municipality_id=user['municipality_id'],
        schema_name=get_schema_from_municipality(municipality_id),
        user_full_name=user['full_name'],
        user_email=email
    )
```

---

## Tools Layer (`tools/`)

Los tools son funciones async que reciben `MCPContext` y parametros tipados.

### Ejemplo: `search_cases` (`tools/cases.py`)

```python
async def search_cases(
    ctx: MCPContext,
    search: str = "",
    page: int = 1,
    page_size: int = 20,
    status: str = None,
    date_filter: str = None,
    sector_filter: str = None
) -> Dict[str, Any]:
    """Busca expedientes accesibles por el usuario."""
    # Usa schema_name del contexto para multi-tenant
    # Filtra por sectores del usuario automaticamente
```

### Ejemplo: `get_document` (`tools/documents.py`)

```python
async def get_document(
    ctx: MCPContext,
    document_id: str
) -> Dict[str, Any]:
    """Obtiene detalle de documento con ai_summary."""
    # Incluye resumen IA si esta disponible
    # Resuelve linked_case si esta vinculado a expediente
```

---

## Autenticacion MCP (`auth_mcp.py`)

### JWT Multi-Audience

Soporta multiples audiences para flexibilidad de clientes:

```python
VALID_AUDIENCES = [
    os.getenv('AUTH0_AUDIENCE'),      # Backend principal
    os.getenv('MCP_RESOURCE_URI'),    # Variable configurable
    "https://mcp.gdilatam.com"        # Produccion hardcoded
]
```

### Flujo de Autenticacion

```
1. Cliente envia POST /mcp sin auth
2. Server responde 401 + WWW-Authenticate header
3. Cliente descubre Auth0 via /.well-known/oauth-protected-resource
4. Cliente hace login en Auth0 (navegador)
5. Cliente envia POST /mcp con Authorization: Bearer <jwt>
6. Server valida JWT con JWKS (cache 30 min)
7. Server extrae email (token o /userinfo fallback)
8. Server busca usuario en BD por email
9. Server construye MCPContext
```

### Multi-Tenant Selection

Si un usuario tiene acceso a multiples municipalidades:

```python
# Error: "multi_tenant_selection_required"
# El usuario debe especificar tenant_id

# Tools disponibles para seleccion:
# list_my_tenants -> retorna municipalidades del usuario
# Luego usar tenant_id en las llamadas
```
