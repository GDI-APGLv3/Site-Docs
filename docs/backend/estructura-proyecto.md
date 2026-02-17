# Estructura del Proyecto

Organizacion de carpetas y archivos del backend GDI.

## Arbol de Carpetas

```
GDI-Backend/
├── main.py                          # Entry point FastAPI
├── server.py                        # Server produccion (Gunicorn/Uvicorn)
├── auth.py                          # Auth0 JWT validation
├── database.py                      # Pool de conexiones, queries, multi-tenant
│
├── endpoints/                       # API REST (controladores thin)
│   ├── __init__.py
│   ├── auth/                        # Autenticacion
│   │   └── onboarding.py            # Registro de usuarios nuevos
│   ├── documents/                   # Documentos
│   │   ├── router.py                # Router principal (agrupa sub-routers)
│   │   ├── create_document.py       # POST /documents
│   │   ├── create_imported.py       # POST /documents/import
│   │   ├── save_document.py         # PATCH /documents/{id}/save
│   │   ├── preview_document.py      # POST /documents/{id}/preview-*
│   │   ├── start_signing.py         # POST /documents/{id}/start-signing-process
│   │   ├── super_sign.py            # POST /documents/{id}/super-sign
│   │   ├── reject_document.py       # POST /documents/{id}/reject
│   │   ├── delete_document.py       # DELETE /documents/{id}
│   │   ├── unified_details.py       # GET /documents/{id}/details
│   │   ├── editor_details.py        # GET /documents/{id}/editor
│   │   ├── signature_details.py     # GET /documents/{id}/signature-details
│   │   ├── search_official.py       # GET /api/v1/documents/search-official/{num}
│   │   ├── autocomplete.py          # GET /documents/autocomplete
│   │   ├── content.py               # GET /documents/{id}/content
│   │   ├── get_types.py             # GET /document-types
│   │   ├── get_states.py            # GET /document-states
│   │   ├── geturl_officialdoc.py    # GET /documents/{id}/official-url
│   │   ├── replace_imported_pdf.py  # PUT /documents/{id}/replace-pdf
│   │   └── check_signer_permissions.py
│   ├── cases/                       # Expedientes
│   │   ├── router.py                # Router principal
│   │   ├── list_cases.py            # GET /api/v1/cases/
│   │   ├── create_case.py           # POST /api/v1/cases/
│   │   ├── get_case_detail.py       # GET /api/v1/cases/{id}
│   │   ├── transfer_case.py         # POST /api/v1/cases/{id}/transfer
│   │   ├── link_document.py         # POST /api/v1/cases/{id}/link-document
│   │   ├── prepare_actions.py       # GET /api/v1/cases/{id}/actions
│   │   ├── get_by_number.py         # GET /api/v1/cases/by-number/{num}
│   │   ├── subsanar_document.py     # POST /api/v1/cases/{id}/subsanar
│   │   └── proposed_documents.py    # Documentos propuestos
│   ├── users/                       # Usuarios
│   │   ├── list_users.py            # GET /users
│   │   ├── profile.py               # GET /users/{id}/profile
│   │   ├── search_users.py          # GET /users/search
│   │   └── get_documents.py         # GET /users/{id}/documents
│   ├── sectors/                     # Sectores
│   │   ├── router.py                # Router principal
│   │   └── list_sectors.py          # GET /api/v1/sectors/
│   ├── notes/                       # Notas internas
│   │   ├── router.py                # Router principal
│   │   ├── received.py              # GET /notes/received
│   │   ├── sent.py                  # GET /notes/sent
│   │   ├── archived.py              # GET /notes/archived
│   │   ├── archive.py               # POST /notes/{id}/archive
│   │   ├── detail.py                # GET /notes/{id}
│   │   └── helpers.py               # Funciones helper
│   ├── dashboard/                   # Dashboard
│   │   ├── router.py                # Router principal
│   │   ├── feed.py                  # GET /api/v1/dashboard/feed
│   │   └── stats.py                 # GET /api/v1/dashboard/stats
│   └── system/                      # Sistema
│       ├── health.py                # GET /health
│       ├── auth_test.py             # GET /auth/test
│       └── login.py                 # Login helpers
│
├── services/                        # Logica de negocio
│   ├── documents/                   # Servicio de documentos (modular)
│   │   ├── core/                    # Base: queries, repository, builder, validator
│   │   ├── catalog/                 # Datos maestros: types, states, metadata
│   │   ├── lifecycle/               # Ciclo de vida: creation, editing, deletion, rejection
│   │   ├── signing/                 # Firma: signing, numerator, unified_signing
│   │   ├── retrieval/               # Consulta: unified_details, content, official_search
│   │   ├── importing/              # PDFs externos: import_service
│   │   ├── pdf/                     # Generacion de PDF
│   │   └── preview/                 # Preview: core, auto_save, data_fetcher
│   ├── cases/                       # Servicio de expedientes
│   │   ├── core.py                  # Funciones base
│   │   ├── creation.py              # Crear expediente + caratula
│   │   ├── cover_creator.py         # Generar CAEX (caratula)
│   │   ├── transfer.py              # Transferir expediente
│   │   ├── transfer_document_creator.py  # Generar PV (pase)
│   │   ├── _document_creator_base.py     # Base para CAEX y PV
│   │   ├── history.py               # Historial de movimientos
│   │   ├── permissions.py           # Permisos sobre expedientes
│   │   ├── documents.py             # Vincular documentos
│   │   ├── retrieval.py             # Consultar expedientes
│   │   ├── subsanacion.py           # Subsanacion de documentos
│   │   ├── queries.py               # Queries SQL
│   │   └── validation.py            # Validaciones
│   ├── notes/                       # Servicio de notas
│   │   ├── queries.py               # Queries SQL
│   │   ├── recipients.py            # Manejo de destinatarios
│   │   ├── retrieval.py             # Consultar notas
│   │   ├── archiving.py             # Archivar notas
│   │   ├── tracking.py              # Seguimiento
│   │   ├── validation.py            # Validaciones
│   │   ├── header_builder.py        # Construir header de nota
│   │   └── save_recipients.py       # Guardar destinatarios
│   ├── users/                       # Servicio de usuarios
│   │   ├── list.py                  # Listar usuarios
│   │   ├── profile.py               # Perfil de usuario
│   │   ├── search.py                # Buscar usuarios
│   │   ├── management.py            # Gestion de usuarios
│   │   ├── user_documents.py        # Documentos del usuario
│   │   ├── document_queries.py      # Queries de documentos
│   │   ├── document_filters.py      # Filtros
│   │   ├── document_mappers.py      # Mappers
│   │   ├── case_filters.py          # Filtros de expedientes
│   │   └── case_validators.py       # Validadores
│   ├── shared/                      # Servicios compartidos
│   │   ├── pdfcomposer_api.py       # Cliente GDI-PDFComposer
│   │   ├── notary_api.py            # Cliente GDI-Notary
│   │   ├── settings_utils.py        # Config de settings (city, etc.)
│   │   ├── pdf_utils.py             # Utilidades PDF
│   │   ├── sector_utils.py          # Utilidades de sectores
│   │   ├── signer_data.py           # Datos de firmantes
│   │   ├── user_data.py             # Datos de usuario
│   │   ├── user_queries.py          # Queries de usuario
│   │   ├── retry.py                 # Retry logic
│   │   ├── resume_trigger.py        # Resume trigger
│   │   └── external_api.py          # API externa base
│   ├── storage/                     # Storage
│   │   └── cloudflare.py            # Cliente Cloudflare R2
│   ├── sectors/
│   │   └── queries.py               # Queries de sectores
│   ├── case_service.py              # CaseService (facade)
│   ├── case_queries.py              # Queries de expedientes
│   ├── document_service.py          # DocumentService legacy
│   ├── user_service.py              # UserService
│   ├── sector_service.py            # SectorService
│   ├── dashboard_service.py         # DashboardService
│   ├── dashboard_queries.py         # Queries de dashboard
│   └── cache.py                     # Cache en memoria
│
├── models/                          # Pydantic schemas
│   ├── schemas.py                   # Schemas principales (User, Document, Auth)
│   ├── tenant_models.py             # Modelos multi-tenant
│   ├── tags.py                      # Tags para Swagger
│   ├── documents/                   # Schemas de documentos
│   │   ├── creation.py              # CreateDocumentRequest/Response
│   │   ├── editing.py               # SaveDocumentRequest/Response
│   │   ├── deletion.py              # DeleteDocumentResponse
│   │   ├── rejection.py             # RejectDocumentRequest/Response
│   │   ├── signing.py               # StartSigningProcessResponse
│   │   ├── unified_signing.py       # SuperSignResponse
│   │   ├── unified.py               # UnifiedDocumentDetailsResponse
│   │   ├── preview.py               # PreviewInfoResponse
│   │   ├── states.py                # DocumentState schemas
│   │   ├── types.py                 # DocumentType schemas
│   │   ├── metadata.py              # Metadata schemas
│   │   ├── official_search.py       # OfficialDocumentSearchResponse
│   │   └── official_url.py          # OfficialDocumentUrlResponse
│   ├── users/                       # Schemas de usuarios
│   │   ├── list.py                  # UserListItem, UserListResponse
│   │   ├── search.py                # UserSearchResponse
│   │   ├── onboarding.py            # OnboardingRequest/Response
│   │   └── user_documents.py        # UserDocumentsResponse
│   ├── notes/                       # Schemas de notas
│   │   ├── recipients.py            # RecipientSchema
│   │   └── responses.py             # NoteResponse
│   └── shared/
│       └── base.py                  # Modelos base compartidos
│
├── shared/                          # Utilidades compartidas
│   ├── exceptions.py                # Excepciones personalizadas
│   ├── validation.py                # Validaciones (UUID, etc.)
│   ├── utils.py                     # Helpers generales
│   ├── dependencies.py              # FastAPI Dependencies
│   ├── numbering.py                 # Numeracion con advisory lock
│   ├── logging.py                   # Configuracion de logging
│   ├── context.py                   # Correlation ID (contextvars)
│   ├── config.py                    # Configuracion
│   ├── tenant_validation.py         # Validacion de tenant
│   └── audit_context.py             # Contexto de auditoria
│
├── middleware/                      # Middleware
│   └── tenant_middleware.py         # TenantMiddleware (multi-tenant)
│
├── api_gateway/                     # MCP Server + REST API
│   ├── http_server.py               # Server HTTP + OAuth discovery
│   ├── server.py                    # MCP Server (stdio transport)
│   ├── run_server.py                # Entry point del gateway
│   ├── auth_mcp.py                  # Auth JWT multi-audience
│   ├── auth_rest.py                 # Auth API Key + X-User-ID
│   ├── context.py                   # MCPContext (multi-tenant)
│   ├── rest_api.py                  # Handlers REST API
│   └── tools/                       # MCP Tools
│       ├── cases.py                 # Tools de expedientes
│       ├── documents.py             # Tools de documentos
│       ├── system.py                # Tools de sistema
│       └── notes.py                 # Tools de notas
│
├── config/
│   └── constants.py                 # Constantes del sistema
│
├── schemas/
│   └── dashboard_schemas.py         # Schemas de dashboard
│
├── scripts/                         # Scripts de utilidad
│   ├── create_api_key.py            # Crear API Keys
│   ├── run_migration.py             # Ejecutar migraciones
│   ├── migrate_profile_picture.py   # Migrar fotos de perfil
│   └── monitor_connections.py       # Monitoreo de conexiones
│
└── tests/                           # Tests
    └── __init__.py
```

## Patron de Organizacion

### Endpoints (Controladores)

Cada modulo de endpoint sigue el patron **thin controller**:

1. Recibe request con validacion Pydantic
2. Extrae `schema_name` via `Depends(get_tenant_schema)`
3. Extrae usuario via `Depends(get_current_user)`
4. Delega a service
5. Retorna response tipado

### Services (Logica de Negocio)

Los services estan organizados por dominio funcional. El modulo mas complejo es `services/documents/`, que esta subdividido en:

| Submmodulo | Responsabilidad |
|-----------|----------------|
| `core/` | Queries base, repositorio, builder, validador |
| `catalog/` | Tipos de documentos, estados, metadata |
| `lifecycle/` | Creacion, edicion, eliminacion, rechazo |
| `signing/` | Firma digital, numeracion, firma unificada |
| `retrieval/` | Busqueda, detalles, contenido |
| `preview/` | Preview PDF, auto-guardado |
| `importing/` | Importar PDFs externos |

### Models (Schemas Pydantic)

Organizados por dominio con convencion `{Model}{Action}`:

- `CreateDocumentRequest` / `CreateDocumentResponse`
- `SaveDocumentRequest` / `SaveDocumentResponse`
- `AuthenticatedUser` / `SectorPermission`
