# Documents Retrieval

Servicios de consulta, busqueda y obtencion de detalles de documentos.

**Ubicacion:** `services/documents/retrieval/`

## Modulos

```
retrieval/
├── unified_details.py    # Detalles segun estado del documento
├── content.py            # Contenido HTML completo
├── official_search.py    # Busqueda por numero oficial
└── official_url.py       # URL de PDF oficial firmado
```

---

## Detalles Unificados (`retrieval/unified_details.py`)

### `get_unified_document_details()`

Punto de entrada unico que delega al servicio apropiado segun el estado del documento.

```python
async def get_unified_document_details(
    document_id: str,
    user_id: Optional[str] = None,
    *,
    schema_name: str
) -> Dict[str, Any]:
```

**Categorias de estado:**

| Estado | Categoria | Servicio delegado |
|--------|-----------|-------------------|
| `draft` | `editing` | `get_document_details_for_editing()` |
| `rejected` | `editing` | `get_document_details_for_editing()` |
| `sent_to_sign` | `signing` | `build_signature_details_response()` |
| `signed` | `signing` | `build_signature_details_response()` |

**Flujo:**

```python
# 1. Obtener estado actual
status = _get_document_status(document_id, schema_name=schema_name)

# 2. Mapear a categoria
category = STATE_CATEGORY_MAP.get(status)  # 'editing' o 'signing'

# 3. Delegar
if category == 'editing':
    details = get_document_details_for_editing(document_id, schema_name=schema_name)
else:
    # Para signing: verificar permisos ANTES de obtener detalles
    _check_user_can_view_document(document_id, user_id, schema_name=schema_name)
    details = await build_signature_details_response(document_id, user_id, schema_name=schema_name)
```

### Control de Acceso

Para documentos en estado `signing`, se verifica que el usuario tenga permiso:

| Condicion | Acceso |
|-----------|--------|
| Es creador (`created_by`) | Permitido |
| Es firmante (`document_signers`) | Permitido |
| Tiene `can_view` en sector del creador | Permitido |
| Ninguna de las anteriores | 403 `AuthorizationError` |

**Respuesta:**

```python
{
    "state_category": "editing",      # o "signing"
    "document_id": "uuid",
    "status": "draft",
    "details": {                       # Datos del servicio delegado
        "document_id": "uuid",
        "reference": "Informe sobre ...",
        "content": "<p>HTML</p>",
        "signers": [...],
        ...
    },
    "timestamp": "2025-01-15T10:30:00"
}
```

---

## Busqueda Oficial (`retrieval/official_search.py`)

### `search_official_document_by_number()`

Busca un documento oficial por su numero exacto en `official_documents`.

```python
def search_official_document_by_number(
    doc_number: str,
    *,
    user_id: str = None,
    schema_name: str
) -> Dict[str, Any]:
```

**Modos de busqueda:**

| Modo | `user_id` | Comportamiento |
|------|-----------|----------------|
| Global | `None` | Sin restricciones de acceso |
| Filtrado | UUID | Solo si `signer_sector_ids` intersecta con sectores del usuario |

**Respuesta:**

```python
# Encontrado:
{
    "found": True,
    "document": {
        "id": "uuid",
        "reference": "IF-2025-0000157-SMG-ADGEN",
        "display_status": "Firmado",
        "document_type": {"name": "Informe", "acronym": "IF"},
        "official_number": "IF-2025-0000157-SMG-ADGEN",
        "last_editor_name": "Juan Perez"
    },
    "search_term": "IF-2025-0000157-SMG-ADGEN"
}

# No encontrado:
{
    "found": False,
    "document": None,
    "search_term": "IF-2025-0000157-SMG-ADGEN"
}
```

---

## URL Oficial (`retrieval/official_url.py`)

Genera URLs firmadas temporales para descargar PDFs oficiales desde R2.

| Bucket | Filename | Expiracion |
|--------|----------|------------|
| `oficial` | `{official_number}.pdf` | 600s (configurable) |
| `tosign` | `{uuid_sin_guiones}.pdf` | 600s (configurable) |

---

## Contenido (`retrieval/content.py`)

Obtiene el contenido HTML completo de un documento, soportando multiples formatos de almacenamiento.

**Formatos soportados:**

| Formato | JSON | Extraccion |
|---------|------|------------|
| Estandar | `{"html": "..."}` | `content['html']` |
| Legacy | `{"detalle": "..."}` | `content['detalle']` |
| Body | `{"body": "..."}` | `content['body']` |
| TipTap | `{"type": "doc", ...}` | `json.dumps(content)` |
