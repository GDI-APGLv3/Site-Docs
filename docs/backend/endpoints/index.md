# Endpoints

Tabla resumen de todos los endpoints REST del backend. Organizados por dominio.

## Documentos

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| GET | `/document-types` | Listar tipos de documentos | Si |
| GET | `/document-states` | Listar estados visuales | Si |
| POST | `/documents` | Crear documento (draft) | Si |
| POST | `/documents/import` | Crear documento importado (PDF externo) | Si |
| GET | `/documents/autocomplete` | Autocompletado de documentos oficiales | Si |
| GET | `/api/v1/documents/search-official/{doc_number}` | Buscar documento oficial por numero | Si |
| GET | `/documents/{id}/details` | Detalles unificados del documento | Si |
| GET | `/documents/{id}/editor` | Detalles para editor | Si |
| GET | `/documents/{id}/signature-details` | Detalles de firma | Si |
| PATCH | `/documents/{id}/save` | Guardar cambios (contenido, firmantes, etc.) | Si |
| PUT | `/documents/{id}/replace-pdf` | Reemplazar PDF importado | Si |
| POST | `/documents/{id}/preview-info` | Informacion para preview | Si |
| POST | `/documents/{id}/preview-download` | Descargar PDF de preview | Si |
| POST | `/documents/{id}/start-signing-process` | Iniciar proceso de firma | Si |
| POST | `/documents/{id}/super-sign` | Firma unificada (comun o numerador) | Si |
| POST | `/documents/{id}/reject` | Rechazar documento | Si |
| DELETE | `/documents/{id}` | Eliminar documento (soft delete) | Si |
| GET | `/documents/{id}/official-url` | URL del PDF oficial | Si |
| GET | `/documents/{id}/content` | Contenido HTML del documento | Si |
| GET | `/documents/{id}/check-signer-permissions` | Verificar permisos de firma | Si |

## Expedientes

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| GET | `/api/v1/cases/` | Listar expedientes con filtros y paginacion | Si |
| POST | `/api/v1/cases/` | Crear expediente (genera CAEX automatica) | Si |
| GET | `/api/v1/cases/templates` | Plantillas de expedientes disponibles | Si |
| GET | `/api/v1/cases/{id}` | Detalle de expediente | Si |
| GET | `/api/v1/cases/by-number/{num}` | Buscar expediente por numero | Si |
| POST | `/api/v1/cases/{id}/transfer` | Transferir expediente a otro sector | Si |
| POST | `/api/v1/cases/{id}/assign` | Asignar tarea sin transferir propiedad | Si |
| POST | `/api/v1/cases/{id}/close-assign` | Cerrar asignacion | Si |
| GET | `/api/v1/cases/{id}/available-sectors` | Sectores disponibles para transferencia | Si |
| GET | `/api/v1/cases/sectors/{sector_id}/users` | Usuarios de un sector | Si |
| POST | `/api/v1/cases/{id}/link-document` | Vincular documento oficial a expediente | Si |
| GET | `/api/v1/cases/{id}/actions` | Acciones disponibles para el usuario | Si |
| POST | `/api/v1/cases/{id}/subsanar` | Subsanar documento en expediente | Si |
| GET | `/api/v1/cases/{id}/proposed-documents` | Documentos propuestos | Si |

## Usuarios

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| GET | `/users` | Listar usuarios activos | Si |
| GET | `/users/{id}/profile` | Perfil de usuario | Si |
| GET | `/users/search` | Buscar usuarios | Si |
| GET | `/users/{id}/documents` | Documentos del usuario con filtros | Si |

## Sectores

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| GET | `/api/v1/sectors/` | Listar sectores con departamentos | Si |

## Notas

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| GET | `/notes/received` | Notas recibidas | Si |
| GET | `/notes/sent` | Notas enviadas | Si |
| GET | `/notes/archived` | Notas archivadas | Si |
| GET | `/notes/{id}` | Detalle de nota | Si |
| POST | `/notes/{id}/archive` | Archivar nota | Si |

## Dashboard

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| GET | `/api/v1/dashboard/feed` | Feed de actividad reciente | Si |
| GET | `/api/v1/dashboard/stats` | Estadisticas del usuario | Si |

## Sistema

| Metodo | Path | Descripcion | Auth |
|--------|------|-------------|------|
| GET | `/health` | Health check | No |
| GET | `/auth/test` | Test de autenticacion | Si |

## Patron de Endpoint

Todos los endpoints siguen el mismo patron:

```python
@router.post("/documents/{document_id}/action")
async def action_endpoint(
    request: Request,
    document_id: str = Path(..., description="UUID del documento"),
    body: ActionRequest = None,
    current_user: AuthenticatedUser = Depends(get_current_user),
    schema_name: str = Depends(get_tenant_schema)
) -> ActionResponse:
    try:
        result = action_service(
            document_id,
            current_user.user_id,
            schema_name=schema_name
        )
        return ActionResponse(**result)
    except (ValidationError, DocumentNotFoundError) as e:
        raise exception_to_http_exception(e)
```
