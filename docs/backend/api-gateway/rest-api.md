# REST API

API REST publica con autenticacion por API Key para integraciones programaticas.

**Ubicacion:** `api_gateway/rest_api.py`

## Autenticacion

Requiere **dos headers** en cada request:

| Header | Descripcion | Ejemplo |
|--------|-------------|---------|
| `X-API-Key` | API Key del sistema externo | `sk-gdi-abc123...` |
| `X-User-ID` | UUID del usuario que ejecuta | `550e8400-e29b-41d4-...` |

```bash
curl -H "X-API-Key: sk-gdi-abc123" \
     -H "X-User-ID: 550e8400-e29b-41d4-a716-446655440000" \
     https://mcp.tu-dominio.com/api/v1/cases/search
```

### Validacion (`auth_rest.py`)

```python
async def validate_rest_api_key(api_key: str, user_id: str) -> MCPContext:
    """
    1. Buscar api_key en public.api_keys (activa, no expirada)
    2. Verificar que user_id esta en public.api_key_users para esa key
    3. Obtener datos del usuario (municipalidad, sector)
    4. Construir MCPContext
    """
```

**Errores:**

| Error | Causa |
|-------|-------|
| 401 "X-API-Key header requerido" | Falta header |
| 401 "X-User-ID header requerido" | Falta header |
| 401 "API Key invalida o expirada" | Key no encontrada o expirada |
| 401 "Usuario no autorizado" | user_id no asociado a la API Key |

### Tablas de BD

```sql
-- Tabla de API Keys (schema: public)
public.api_keys (
    id UUID PRIMARY KEY,
    key_hash VARCHAR,        -- Hash de la API Key
    name VARCHAR,            -- Nombre descriptivo
    is_active BOOLEAN,
    expires_at TIMESTAMP,
    created_at TIMESTAMP
)

-- Usuarios autorizados por API Key
public.api_key_users (
    api_key_id UUID REFERENCES public.api_keys,
    user_id UUID,
    municipality_id UUID
)
```

---

## Endpoints

### Expedientes

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/api/v1/cases/search` | Buscar expedientes |
| GET | `/api/v1/cases/{case_id}` | Detalle de expediente |
| GET | `/api/v1/cases/{case_id}/history` | Historial de movimientos |
| GET | `/api/v1/cases/{case_id}/documents` | Documentos del expediente |
| GET | `/api/v1/cases/{case_id}/permissions` | Permisos del usuario |

### Documentos

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/api/v1/documents/search` | Buscar documentos |
| GET | `/api/v1/documents/pending-signatures` | Firmas pendientes |
| GET | `/api/v1/documents/{document_id}` | Detalle de documento |
| GET | `/api/v1/documents/{document_id}/content` | Contenido HTML |

### Sistema

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/api/v1/system/document-types` | Tipos de documentos |
| GET | `/api/v1/system/sectors` | Sectores y departamentos |
| GET | `/api/v1/system/users/{user_id}` | Info de usuario |
| GET | `/api/v1/system/case-templates` | Templates de expedientes |

---

## Implementacion de Handlers

Cada handler valida auth, extrae parametros y llama al tool correspondiente:

```python
async def api_search_cases(request: Request) -> JSONResponse:
    """GET /api/v1/cases/search"""

    # 1. Extraer headers
    api_key = request.headers.get("X-API-Key")
    user_id = request.headers.get("X-User-ID")

    # 2. Validar autenticacion
    if not api_key:
        return _error_response("X-API-Key header requerido", 401)
    if not user_id:
        return _error_response("X-User-ID header requerido para REST API", 401)

    ctx = await validate_rest_api_key(api_key, user_id)

    # 3. Extraer query params
    params = dict(request.query_params)
    page = int(params.get("page", 1))
    search = params.get("search", "")

    # 4. Llamar al tool (misma logica que MCP)
    result = await cases.search_cases(ctx, search=search, page=page, ...)

    # 5. Retornar respuesta
    return _success_response(result)
```

### Serializacion

Los handlers usan un serializador custom para `datetime` y `UUID`:

```python
def _json_serializer(obj):
    if isinstance(obj, (datetime, date)):
        return obj.isoformat()
    if isinstance(obj, UUID):
        return str(obj)
    raise TypeError(f"Object of type {type(obj).__name__} is not JSON serializable")
```

---

## Query Parameters Comunes

### Busqueda de Expedientes

| Parametro | Tipo | Default | Descripcion |
|-----------|------|---------|-------------|
| `page` | int | 1 | Numero de pagina |
| `page_size` | int | 20 | Items por pagina (max 100) |
| `search` | string | "" | Buscar por numero o referencia |
| `status` | string | null | `active`, `inactive`, `archived` |
| `date_filter` | string | null | `today`, `week`, `month`, `year` |
| `sector_filter` | string | null | Acronimo del sector |

### Busqueda de Documentos

| Parametro | Tipo | Default | Descripcion |
|-----------|------|---------|-------------|
| `page` | int | 1 | Numero de pagina |
| `page_size` | int | 20 | Items por pagina |
| `search` | string | "" | Buscar por referencia o numero |
| `status` | string | null | `draft`, `sent_to_sign`, `signed` |
| `doc_type` | string | null | Acronimo de tipo (INF, DICT, etc.) |

---

## Ejemplo Completo

```bash
# Buscar expedientes de habilitacion
curl -s \
  -H "X-API-Key: sk-gdi-abc123" \
  -H "X-User-ID: 550e8400-e29b-41d4-a716-446655440000" \
  "https://mcp.tu-dominio.com/api/v1/cases/search?search=habilitacion&status=active&page=1"

# Respuesta
{
  "cases": [
    {
      "case_id": "uuid",
      "case_number": "EE-2025-00001-SMG-ADGEN",
      "reference": "Habilitacion comercial - Panaderia",
      "status": "active",
      "admin_sector": "ADGEN",
      "last_modified_at": "2025-01-15T10:30:00"
    }
  ],
  "total": 1,
  "page": 1,
  "page_size": 20
}
```
