# Models y Schemas

El backend usa **Pydantic v2** para validacion de requests y responses. Los modelos estan organizados en `models/` por dominio.

## Convencion de Nombres

| Elemento | Patron | Ejemplo |
|----------|--------|---------|
| Request | `{Action}{Domain}Request` | `CreateDocumentRequest` |
| Response | `{Action}{Domain}Response` | `CreateDocumentResponse` |
| Item | `{Domain}Item` | `UserListItem` |
| Lista | `{Domain}ListResponse` | `UserListResponse` |

## Modelos Principales

### AuthenticatedUser

Modelo que representa al usuario autenticado en cada request. Se obtiene via `Depends(get_current_user)`.

```python
# models/schemas.py
class AuthenticatedUser(BaseModel):
    user_id: str
    auth_id: str
    full_name: str
    email: str
    permissions: List[SectorPermission] = []
```

### SectorPermission

Permisos del usuario sobre un sector especifico.

```python
# models/schemas.py
class SectorPermission(BaseModel):
    sector_id: str
    sector_acronym: str
    department_id: str
    department_name: str
    department_acronym: str
    can_view: bool
    can_edit: bool
    is_primary: bool
```

## Schemas de Documentos

Ubicados en `models/documents/`.

### Creacion

```python
# models/documents/creation.py
class CreateDocumentRequest(BaseModel):
    document_type_acronym: str    # "INF", "DICT", "NOTA", etc.
    reference: str                # Asunto (max 250 chars)
    recipients: Optional[RecipientsSchema] = None  # Solo para NOTA

class CreateDocumentResponse(BaseModel):
    document_id: str
    status: str    # "draft"
    message: str
```

### Edicion

```python
# models/documents/editing.py
class SaveDocumentRequest(BaseModel):
    reference: Optional[str] = None          # Asunto
    content: Optional[str] = None            # HTML
    signers: Optional[List[SignerSchema]] = None
    recipients: Optional[RecipientsSchema] = None  # Solo NOTA
    proposed_case_ids: Optional[List[str]] = None

class SaveDocumentResponse(BaseModel):
    success: bool
    message: str
    document_id: str
    last_modified_at: Optional[datetime] = None
```

### Firma

```python
# models/documents/signing.py
class StartSigningProcessResponse(BaseModel):
    success: bool
    message: str

# models/documents/unified_signing.py
class SuperSignResponse(BaseModel):
    success: bool
    message: str
    document_id: str
    signature_id: str
    document_status: str          # "sent_to_sign" o "signed"
    signed_at: datetime
    is_numerator: bool
    official_number: Optional[str] = None
    signed_pdf_url: Optional[str] = None
```

### Rechazo y Eliminacion

```python
# models/documents/rejection.py
class RejectDocumentRequest(BaseModel):
    reason: str

class RejectDocumentResponse(BaseModel):
    success: bool
    message: str
    rejection_id: str

# models/documents/deletion.py
class DeleteDocumentResponse(BaseModel):
    success: bool
    message: str
    unlinked_cases: int
```

### Detalles Unificados

```python
# models/documents/unified.py
class UnifiedDocumentDetailsResponse(BaseModel):
    state_category: str    # "editing" o "signing"
    status: str
    details: dict
```

### Busqueda

```python
# models/documents/official_search.py
class OfficialDocumentSearchResponse(BaseModel):
    found: bool
    document: Optional[dict] = None
    search_term: str
```

## Schemas de Usuarios

Ubicados en `models/users/`.

```python
# models/users/list.py
class UserListItem(BaseModel):
    user_id: str
    full_name: str
    email: str

class UserListResponse(BaseModel):
    users: List[UserListItem]
    total: int
```

```python
# models/schemas.py
class User(BaseModel):
    user_id: str
    auth_id: Optional[str]
    full_name: str
    email: str
    cuit: Optional[str]
    profile_picture_url: Optional[str]
    sector_id: Optional[str]
    sector_acronym: Optional[str]
    department_id: Optional[str]
    department_name: Optional[str]
    estado: int    # 1=activo, 0=inactivo
```

## Schemas de Notas

Ubicados en `models/notes/`.

```python
# models/notes/recipients.py
class RecipientsSchema(BaseModel):
    to: List[str]              # UUIDs de sectores (requerido)
    cc: Optional[List[str]]    # Copia
    bcc: Optional[List[str]]   # Copia oculta
```

## Tags para Swagger

Los endpoints se agrupan en Swagger UI mediante tags definidos en `models/tags.py`:

```python
class Tags(str, Enum):
    DOCUMENTOS = "documentos"
    SECTORS = "sectors"
    USERS = "users"
    SISTEMA = "sistema"
```

## Jerarquia de Excepciones

Definida en `shared/exceptions.py`:

```
GDIBaseException
├── ValidationError              -> 400
├── AuthorizationError           -> 403
│   ├── DocumentPermissionError
│   └── CasePermissionError
├── NotFoundError                -> 404
│   ├── DocumentNotFoundError
│   ├── UserNotFoundError
│   └── CaseNotFoundError
├── ConflictError                -> 409
│   ├── DocumentAlreadySignedError
│   └── DocumentAlreadyRejectedError
├── BusinessLogicError           -> 422
│   ├── DocumentStateError
│   ├── InvalidSignatureOrderError
│   ├── NumeratorRequiredError
│   └── UserInactiveError
├── DatabaseError                -> 500
└── ExternalServiceError         -> 502
```

La funcion `exception_to_http_exception()` convierte automaticamente estas excepciones a `HTTPException` de FastAPI con el codigo HTTP correspondiente.
