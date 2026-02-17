# Notes Service

Servicios para el sistema de notas internas entre sectores con destinatarios TO, CC, BCC.

**Ubicacion:** `services/notes/`

## Modulos

```
notes/
├── recipients.py      # Obtencion de recipients con seguridad BCC
├── save_recipients.py # Guardar/eliminar recipients
├── validation.py      # Validacion de recipients y tipo NOTA
├── queries.py         # SQL centralizado
├── retrieval.py       # Bandejas recibidas/enviadas/archivadas
├── archiving.py       # Archivar notas
├── header_builder.py  # Header para PDF
└── tracking.py        # Tracking de lectura
```

---

## Recipients (`recipients.py`)

### Seguridad BCC

La regla fundamental: **BCC solo es visible para el sender**.

```python
def get_visible_recipients(
    document_id: str,
    requesting_sector_id: str,
    *, schema_name: str
) -> Dict[str, Any]:
```

| Rol del solicitante | Ve TO | Ve CC | Ve BCC |
|---------------------|-------|-------|--------|
| Sender | Si | Si | Si |
| Destinatario TO/CC | Si | Si | No |
| No relacionado | Error 403 | Error 403 | Error 403 |

**Respuesta:**

```python
{
    "to": [{"sector_id": "uuid", "acronym": "ADGEN", "department_name": "Admin General"}],
    "cc": [...],
    "bcc": [...],           # Solo si es sender
    "is_sender": True,
    "my_recipient_type": "TO"  # o "CC", "BCC", None
}
```

### Formato para PDF

```python
def format_recipients_for_pdf(
    document_id: str, *, schema_name: str
) -> Dict[str, Optional[str]]:
    """
    Formato: "Departamento#Sector, Departamento2#Sector2"
    BCC nunca se incluye en el PDF.
    """
```

**Ejemplo:**

```python
{
    "para": "Finanzas#Tesoreria, Legales#Asesoria",
    "cc": "RRHH#Personal"  # None si no hay CC
}
```

---

## Validacion (`validation.py`)

### Funciones Principales

```python
def is_nota_document_type_by_acronym(
    acronym: str, *, schema_name: str
) -> bool:
    """Verifica si el acronimo corresponde a tipo NOTA."""

def validate_recipients_input(recipients: Dict) -> Dict:
    """
    Valida y normaliza estructura de recipients.
    Incluye deduplicacion automatica.
    Input: {"to": ["uuid1"], "cc": ["uuid2"], "bcc": ["uuid3"]}
    """

def validate_recipients_exist(
    cursor, normalized: Dict, sender_sector_id: str, *, schema_name: str
):
    """Valida que todos los sector_id existan en la BD."""

def validate_nota_recipients_for_signing(
    document_id: str, *, schema_name: str
):
    """Valida que una NOTA tenga al menos un destinatario TO antes de firmar."""
```

### Reglas de Validacion

| Regla | Descripcion |
|-------|-------------|
| Formato | `to`, `cc`, `bcc` deben ser listas de UUIDs |
| Deduplicacion | Sector duplicado en misma categoria se elimina |
| Auto-envio | Sender no puede estar en TO/CC/BCC |
| Minimo para firma | Al menos 1 destinatario TO para iniciar firma |
| Existencia | Todos los sector_id deben existir en `sectors` |

---

## Save Recipients (`save_recipients.py`)

```python
def save_recipients(
    cursor,
    document_id: str,
    sender_sector_id: str,
    normalized_recipients: Dict,
    *, schema_name: str
) -> int:
    """Guarda recipients en BD (dentro de transaccion existente)."""

def delete_recipients(cursor, document_id: str) -> int:
    """Elimina todos los recipients de un documento."""
```

**Tabla `note_recipients`:**

| Columna | Tipo | Descripcion |
|---------|------|-------------|
| `id` | UUID | PK |
| `document_id` | UUID | FK a document_draft |
| `sender_sector_id` | UUID | Sector emisor |
| `sector_id` | UUID | Sector destinatario |
| `recipient_type` | VARCHAR | `TO`, `CC`, `BCC` |

---

## Retrieval (`retrieval.py`)

Servicios para las bandejas de notas.

### Notas Recibidas

```python
def get_received_notes(
    sector_id: str,
    page: int = 1,
    page_size: int = 20,
    search: Optional[str] = None,
    *, schema_name: str
) -> Dict[str, Any]:
```

Retorna notas oficializadas donde el sector es destinatario. Incluye `is_read` para tracking.

### Notas Enviadas

```python
def get_sent_notes(
    user_id: str,
    page: int = 1,
    page_size: int = 20,
    *, schema_name: str
) -> Dict[str, Any]:
```

### Notas Archivadas

```python
def get_archived_notes(
    user_id: str,
    sector_id: str,
    *, schema_name: str
) -> Dict[str, Any]:
```

---

## Flujo Completo de una NOTA

```
1. POST /documents
   create_document(type="NOTA", recipients={to: [...], cc: [...]})

2. PATCH /documents/{id}/save
   save_document_changes(content="...", recipients={...})

3. POST /documents/{id}/start-signing-process
   - Valida recipients existentes
   - PDFComposer /note/ con header de recipients
   - Genera PDF y sube a R2

4. POST /documents/{id}/super-sign
   - Firma con Notary
   - Al oficializarse: aparece en GET /notes/received

5. GET /notes/received
   - Sectores destino ven la nota
   - BCC solo visible para sender

6. POST /notes/{id}/archive
   - Archiva la nota del sector
```
