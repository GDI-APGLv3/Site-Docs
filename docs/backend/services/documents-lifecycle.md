# Documents Lifecycle

Servicios del ciclo de vida de documentos: creacion, edicion, eliminacion y rechazo.

**Ubicacion:** `services/documents/lifecycle/`

## Flujo de Estados

```
             create_document()
                    |
                    v
               [ draft ] <------- save_document_changes()
                    |                     ^
                    |                     |
            start_signing()         reject / edit
                    |                     |
                    v                     |
           [ sent_to_sign ] -------> [ rejected ]
                    |
              sign_document()
                    |
                    v
              [ signed ]
```

---

## Creacion (`lifecycle/creation.py`)

### `create_document()`

```python
def create_document(
    document_type_acronym: str,
    reference: str,
    creator_id: str,
    *,
    schema_name: str,
    recipients: Optional[Dict] = None,      # Solo para NOTA
    sender_sector_id: Optional[str] = None,  # Solo para NOTA
    auth_source: Optional[str] = None
) -> Dict[str, Any]:
```

**Flujo:**

1. Validar inputs (acronimo, referencia, UUID del creador)
2. Detectar si es tipo NOTA (validar recipients si aplica)
3. Obtener `document_type_id` desde BD
4. Generar UUID para el documento
5. INSERT en `document_draft` con estado `draft`
6. Asignar creador como firmante numerador (signing_order=1)
7. Si es NOTA con recipients: guardar en misma transaccion
8. Commit y retornar resultado

**Reglas especiales para NOTA:**

| Escenario | Comportamiento |
|-----------|----------------|
| NOTA con recipients | Valida y guarda TO/CC/BCC |
| NOTA sin recipients | Valido (se agregan despues con `save`) |
| No-NOTA con recipients | Ignora con warning en log |

**Contexto de auditoria:**

```python
# Inyecta metadata para triggers de BD
set_audit_context(cursor, user_id=creator_id, auth_source=auth_source)
```

---

## Edicion (`lifecycle/editing.py`)

### `get_document_details_for_editing()`

Obtiene datos completos del documento para la pantalla de edicion.

```python
def get_document_details_for_editing(
    document_id: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Datos incluidos en la respuesta:**

- Datos basicos: referencia, contenido HTML, estado, tipo
- Firmantes con sello, departamento, foto de perfil
- Info de rechazo (si aplica)
- PDF URL (si es documento importado y existe en R2)
- Recipients (si es tipo NOTA)
- Expedientes propuestos

### `save_document_changes()`

Guarda cambios en un documento existente. Soporta actualizacion parcial.

```python
def save_document_changes(
    document_id: str,
    reference: Optional[str] = None,
    content: Optional[str] = None,
    signers: Optional[List[Dict]] = None,
    recipients: Optional[Dict] = None,
    sender_sector_id: Optional[str] = None,
    proposed_case_ids: Optional[List[str]] = None,
    user_id: Optional[str] = None,
    *,
    schema_name: str
) -> Dict[str, Any]:
```

**Operaciones atomicas en transaccion:**

| Campo | Operacion |
|-------|-----------|
| `reference` | UPDATE document_draft.reference |
| `content` | UPDATE document_draft.content (formato `{"html": "..."}`) |
| `signers` | DELETE + INSERT en document_signers (reescritura completa) |
| `recipients` | DELETE + INSERT en note_recipients (solo NOTA) |
| `proposed_case_ids` | DELETE + INSERT en case_proposed_documents (deduplicados) |

!!! info "Deduplicacion"
    Los `proposed_case_ids` se deduplican automaticamente con `set()` para prevenir vinculaciones duplicadas. Se registra en `case_history` solo para expedientes **nuevos**.

### `check_document_can_be_edited()`

Verifica sin lanzar excepciones (retorna `dict` con `can_edit: bool`).

---

## Eliminacion (`lifecycle/deletion.py`)

### `delete_document()`

Soft delete: marca `is_deleted = true` sin borrar fisicamente.

```python
def delete_document(
    document_id: str, user_id: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Validaciones:**

| Validacion | Error |
|------------|-------|
| Documento no existe | `DocumentNotFoundError` |
| Ya eliminado | `ConflictError` |
| No es el creador | `AuthorizationError` |
| Estado no permitido | `DocumentStateError` |

**Estados eliminables:** `draft`, `rejected`

**Proceso:**

1. Validar formato de UUIDs
2. Obtener documento y verificar condiciones
3. Limpiar PDF de R2 (soft-fail, no bloquea si falla)
4. En transaccion: desvincular de expedientes + soft delete
5. Retornar resultado con info de cleanup

```python
# La limpieza de PDF no bloquea el proceso
pdf_cleanup_info = _cleanup_pdf_from_r2(document_id, schema_name=schema_name)
# Retorna: {"attempted": True, "success": True/False, "note": "..."}
```

---

## Rechazo (`lifecycle/rejection.py`)

### `reject_document()`

Rechaza un documento con motivo. Revierte el estado para edicion.

```python
def reject_document(
    document_id: str, user_id: str, reason: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Quien puede rechazar:**

| Rol | Permitido |
|-----|-----------|
| Creador del documento | Si |
| Firmante asignado | Si |
| Otro usuario | No (403) |

**Operaciones en transaccion:**

1. `UPDATE document_draft SET status = 'rejected', sent_to_sign_at = NULL, sent_by = NULL`
2. `INSERT INTO document_rejections`
3. `UPDATE document_signers SET status = 'rejected' WHERE signed_at IS NULL`
4. Limpieza de PDF en R2 (soft-fail)

**Consultas adicionales:**

| Funcion | Proposito |
|---------|-----------|
| `get_document_rejections()` | Historial de rechazos de un documento |
| `get_rejected_documents_for_user()` | Documentos rechazados donde el usuario participa |
| `can_user_reject_document()` | Verificacion sin excepciones |
