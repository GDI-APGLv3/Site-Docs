# API Gateway

El API Gateway de GDI-Backend expone dos interfaces externas: un **MCP Server** para clientes IA y una **REST API** para integraciones programaticas.

**Ubicacion:** `api_gateway/`

## Arquitectura

```
                  Clientes IA                    Aplicaciones Externas
              (Claude, ChatGPT, Gemini)           (Scripts, Bots)
                       |                                |
                   OAuth 2.0                     API Key + X-User-ID
                       |                                |
                       v                                v
              ┌────────────────┐               ┌────────────────┐
              │   MCP Server   │               │   REST API     │
              │  POST /mcp     │               │  /api/v1/*     │
              └───────┬────────┘               └───────┬────────┘
                      |                                |
                      v                                v
              ┌────────────────────────────────────────────┐
              │              Tools Layer                    │
              │  cases.py | documents.py | system.py       │
              └──────────────────┬─────────────────────────┘
                                 |
                                 v
              ┌────────────────────────────────────────────┐
              │          GDI-Backend Services               │
              │      (database, storage, etc.)              │
              └────────────────────────────────────────────┘
```

## Estructura de Archivos

```
api_gateway/
├── server.py          # MCP Server (stdio transport, 12 tools)
├── http_server.py     # HTTP Server (ASGI, OAuth discovery, REST routes)
├── auth_mcp.py        # Autenticacion MCP (JWT multi-audience)
├── auth_rest.py       # Autenticacion REST (API Key + X-User-ID)
├── context.py         # MCPContext (multi-tenant)
├── rest_api.py        # Handlers REST API
└── tools/             # Tools compartidos entre MCP y REST
    ├── cases.py       # search_cases, get_case, get_case_history, etc.
    ├── documents.py   # search_documents, get_document, etc.
    ├── system.py      # get_document_types, get_sectors, etc.
    └── notes.py       # (futuro) Tools de notas
```

## Comparacion MCP vs REST

| Aspecto | MCP Server | REST API |
|---------|------------|----------|
| **Protocolo** | MCP (JSON-RPC sobre SSE) | HTTP REST |
| **Endpoint** | `POST /mcp` | `GET /api/v1/*` |
| **Auth** | OAuth 2.0 (Auth0 JWT) | API Key + X-User-ID |
| **Clientes** | Claude Code, ChatGPT, Gemini | Scripts, bots, apps |
| **Discovery** | `/.well-known/oauth-protected-resource` | `/.well-known/openapi.json` |
| **Multi-tenant** | Automatico (JWT -> email -> usuario) | Manual (X-User-ID -> sector) |
| **Tools** | 14 tools con parametros tipados | 14 endpoints GET equivalentes |

## URLs de Produccion

| Servicio | URL |
|----------|-----|
| MCP Server | `https://mcp.tu-dominio.com/mcp` |
| REST API | `https://mcp.tu-dominio.com/api/v1/*` |
| OAuth Discovery | `https://mcp.tu-dominio.com/.well-known/oauth-protected-resource` |
| OpenAPI Spec | `https://mcp.tu-dominio.com/.well-known/openapi.json` |

## MCPContext

Contexto compartido entre tools y handlers REST:

```python
@dataclass
class MCPContext:
    user_id: str            # UUID del usuario autenticado
    municipality_id: str    # UUID de la municipalidad
    schema_name: str        # Schema de BD (ej: "200_muni")
    user_full_name: str     # Nombre completo
    user_email: str         # Email
```
