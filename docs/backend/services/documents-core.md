# Documents Core

Modulos base del sistema de documentos: queries SQL, repository de datos, constructor de respuestas y validador de reglas de negocio.

**Ubicacion:** `services/documents/core/`

## Arquitectura

```
core/
├── queries.py      # SQL centralizado (~740 lineas, ~30 queries)
├── repository.py   # Patron Repository para acceso a datos
├── builder.py      # Constructor de respuestas (Single Responsibility)
└── validator.py    # Reglas de negocio y validaciones
```

---

## Queries (`core/queries.py`)

Centraliza **todas** las consultas SQL del modulo de documentos. Cada query es una funcion que retorna un string SQL.

### Queries por Categoria

| Categoria | Funciones | Proposito |
|-----------|-----------|-----------|
| **Creacion** | `insert_document_draft_query`, `insert_document_signer_query` | INSERT de drafts y firmantes |
| **Edicion** | `update_document_reference_query`, `update_document_content_query` | UPDATE de campos |
| **Eliminacion** | `soft_delete_document_query`, `unlink_document_from_cases_query` | Soft delete + desvincular |
| **Rechazo** | `update_document_to_rejected_query`, `insert_rejection_record_query` | Cambiar estado + registrar |
| **Firma** | `get_signer_role_and_document_status_query`, `update_signer_status_to_signed_query` | Proceso de firma |
| **Consulta** | `get_document_details_for_editing_query`, `search_official_document_by_number_query` | Lectura de datos |
| **Catalogo** | `get_all_document_types_query`, `get_all_display_states_query` | Datos maestros |

### Ejemplo: Query de Detalles para Edicion

```python
def get_document_details_for_editing_query() -> str:
    return """
        SELECT
            d.id, d.reference, d.content, d.status,
            d.created_by as creator_id, d.last_modified_at, d.resume,
            dt.name as document_type_name,
            dt.acronym as document_type_acronym,
            dt.type as document_type_source,
            u.full_name as creator_name,
            u.profile_picture_url as creator_profile_picture_url,
            dep.acronym as creator_department_acronym,
            sec.acronym as creator_sector_acronym
        FROM document_draft d
            LEFT JOIN document_types dt ON d.document_type_id = dt.id
            LEFT JOIN users u ON d.created_by = u.id
            LEFT JOIN sectors sec ON u.sector_id = sec.id
            LEFT JOIN departments dep ON sec.department_id = dep.id
        WHERE d.id = %s
    """
```

### Tipos Excluidos

Algunos tipos de documentos se excluyen de listados generales (son automaticos):

```python
def get_excluded_types_clause() -> str:
    # Excluye CAEX (caratulas) y PV (pases) de busquedas
    types_list = ', '.join(f"'{t}'" for t in EXCLUDED_DOCUMENT_TYPES)
    return f"AND dt.acronym NOT IN ({types_list})"
```

---

## Repository (`core/repository.py`)

Patron Repository que centraliza acceso a datos. Todas las funciones son `@staticmethod` con `schema_name` keyword-only.

```python
class DocumentRepository:
    """
    Repository para acceso a datos de documentos.
    Compatible con PgBouncer transaction mode (SET LOCAL).
    """

    @staticmethod
    def get_basic_details(document_id: str, *, schema_name: str) -> Dict:
        """Obtiene datos basicos del documento."""
        with get_db_connection(schema_name) as conn:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT d.id, d.reference, d.content, d.status,
                           d.created_by as creator_id, d.last_modified_at,
                           dt.name as document_type_name,
                           dt.acronym as document_type_acronym,
                           u.full_name as creator_name
                    FROM document_draft d
                        LEFT JOIN document_types dt ON d.document_type_id = dt.id
                        LEFT JOIN users u ON d.created_by = u.id
                    WHERE d.id = %s
                """, (document_id,))
                document = cursor.fetchone()
                if not document:
                    raise DocumentNotFoundError(document_id)
                return document

    @staticmethod
    def get_signers(document_id: str, *, schema_name: str) -> List[Dict]:
        """Obtiene firmantes con info de sello y departamento."""

    @staticmethod
    def get_rejection_info(document_id: str, *, schema_name: str) -> Optional[Dict]:
        """Obtiene informacion del ultimo rechazo."""

    @staticmethod
    def get_status(document_id: str, *, schema_name: str) -> Optional[str]:
        """Obtiene solo el estado del documento (operacion rapida)."""

    @staticmethod
    def exists(document_id: str, *, schema_name: str) -> bool:
        """Verifica si el documento existe."""
```

---

## Builder (`core/builder.py`)

Constructor de respuestas que aplica Single Responsibility. Solo formateo de datos, sin acceso a BD.

```python
class DocumentBuilder:

    @staticmethod
    def build_complete_response(
        document: Dict, signers: List[Dict],
        rejection_info: Optional[Dict] = None
    ) -> Dict:
        return {
            "document_id": document['id'],
            "reference": document['reference'],
            "content": DocumentBuilder._extract_content(document['content']),
            "status": document['status'],
            "document_type": DocumentBuilder._build_document_type(document),
            "created_by": document['creator_id'],
            "creator_name": document['creator_name'],
            "signers": DocumentBuilder._format_signers(signers),
            "rejection_info": DocumentBuilder._format_rejection_info(rejection_info),
            "updated_at": document['last_modified_at'].isoformat() if document['last_modified_at'] else None
        }
```

### Extraccion de Contenido

Soporta multiples formatos de contenido para migracion gradual:

| Prioridad | Formato | Clave JSON |
|-----------|---------|------------|
| 1 | Estandar nuevo | `{"html": "contenido"}` |
| 2 | Legacy | `{"detalle": "contenido"}` |
| 3 | Alternativo | `{"body": "contenido"}` |
| 4 | TipTap | `{"type": "doc", "content": [...]}` |

---

## Validator (`core/validator.py`)

Reglas de negocio centralizadas para documentos.

```python
class DocumentValidator:

    EDITABLE_STATES = EDITABLE_DOCUMENT_STATES  # ['draft', 'rejected']

    @classmethod
    def validate_can_be_edited(cls, document_id: str, *, schema_name: str):
        """Valida que documento existe y puede ser editado."""
        status = DocumentRepository.get_status(document_id, schema_name=schema_name)
        if status is None:
            raise DocumentNotFoundError(document_id)
        if status not in cls.EDITABLE_STATES:
            raise DocumentStateError(...)

    @classmethod
    def validate_update_data(cls, reference, content, signers):
        """Valida datos de actualizacion."""
        if reference is None and content is None and signers is None:
            raise ValidationError("Debe proporcionar al menos un campo")

    @classmethod
    def _validate_signers_list(cls, signers: List[Dict]):
        """Valida lista de firmantes."""
        # Exactamente 1 numerador requerido
        numerators = [s for s in signers if s.get("is_numerator")]
        if len(numerators) != 1:
            raise ValidationError("Debe haber exactamente un numerador")
```

### Reglas de Firmantes

| Regla | Validacion |
|-------|------------|
| Identificacion | `user_id` O `email`, nunca ambos |
| Numerador | Exactamente 1 por documento |
| UUID | Formato valido si se proporciona `user_id` |
| Email | Debe contener `@` si se proporciona |
| is_numerator | Debe ser `bool` |
