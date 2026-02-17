# Endpoints de Documentos

Endpoints para el ciclo de vida completo de documentos: creacion, edicion, preview, firma, rechazo y eliminacion.

## Flujo de Vida de un Documento

```
POST /documents (crear)
    |
    v
PATCH /documents/{id}/save (guardar contenido + firmantes)
    |
    v
POST /documents/{id}/preview-download (preview PDF)
    |
    v
POST /documents/{id}/start-signing-process (enviar a firma)
    |
    v
POST /documents/{id}/super-sign (firma de cada firmante)
    |
    +-- Firmante comun -> firma y sigue en sent_to_sign
    +-- Numerador (ultimo) -> firma, numera, oficializa -> signed
```

## Crear Documento

### `POST /documents`

Crea un documento en estado `draft`.

**Request:**

```json
{
    "document_type_acronym": "INF",
    "reference": "Informe sobre presupuesto anual",
    "recipients": {
        "to": ["uuid-sector-1"],
        "cc": ["uuid-sector-2"],
        "bcc": []
    }
}
```

!!! note "Recipients"
    El campo `recipients` solo es requerido para documentos tipo **NOTA**. Para otros tipos se omite.

**Response (200):**

```json
{
    "document_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "draft",
    "message": "Documento creado exitosamente"
}
```

**Archivo:** `endpoints/documents/create_document.py`

---

## Guardar Cambios

### `PATCH /documents/{document_id}/save`

Guarda cambios parciales o completos en un documento editable. Solo funciona en estado `draft` o `rejected`.

**Request:**

```json
{
    "reference": "Nuevo asunto",
    "content": "<p>Contenido HTML del documento</p>",
    "signers": [
        {
            "user_id": "uuid-firmante",
            "is_numerator": true
        }
    ],
    "proposed_case_ids": ["uuid-expediente-1", "uuid-expediente-2"]
}
```

Todos los campos son opcionales. Solo se actualizan los campos enviados.

**Response (200):**

```json
{
    "success": true,
    "message": "Cambios guardados exitosamente",
    "document_id": "uuid",
    "last_modified_at": "2025-01-15T10:30:00"
}
```

**Archivo:** `endpoints/documents/save_document.py`

---

## Preview

### `POST /documents/{document_id}/preview-info`

Obtiene informacion completa del documento para previsualizacion sin generar PDF.

### `POST /documents/{document_id}/preview-download`

Genera y descarga PDF del documento:

- **Documentos HTML**: PDF con marca de agua "BORRADOR"
- **Documentos importados**: URL firmada del PDF en R2

**Response (documentos HTML):** Binario PDF con `Content-Type: application/pdf`

**Response (documentos importados):**

```json
{
    "success": true,
    "pdf_url": "https://r2.cloudflare.com/...",
    "is_imported": true
}
```

**Archivo:** `endpoints/documents/preview_document.py`

---

## Iniciar Firma

### `POST /documents/{document_id}/start-signing-process`

Cambia el estado del documento de `draft` a `sent_to_sign`.

**Requisitos:**

- Solo el **creador** puede iniciar el proceso
- Debe tener al menos un **firmante** asignado
- Debe tener un **numerador** asignado

**Proceso:**

1. Valida que el usuario sea el creador
2. Verifica firmantes y numerador
3. Genera el PDF final
4. Cambia estado a `sent_to_sign`

**Response (200):**

```json
{
    "success": true,
    "message": "Proceso de firma iniciado exitosamente"
}
```

**Archivo:** `endpoints/documents/start_signing.py`

---

## Firma Unificada

### `POST /documents/{document_id}/super-sign`

Endpoint unificado que detecta automaticamente si el usuario es firmante comun o numerador.

**Firmante comun:**

1. Estampa firma digital via GDI-Notary
2. Actualiza estado del firmante a `signed`
3. Documento sigue en `sent_to_sign`

**Numerador (ultimo firmante):**

1. Valida que todos los firmantes comunes hayan firmado
2. Genera numero oficial (ej: `INF-2025-0001234-SMG-ADGEN`)
3. Estampa firma y numera via GDI-Notary
4. Mueve PDF a bucket oficial en R2
5. Cambia estado a `signed`

**Response (firmante comun):**

```json
{
    "success": true,
    "message": "Documento firmado exitosamente",
    "document_id": "uuid",
    "document_status": "sent_to_sign",
    "is_numerator": false,
    "official_number": null
}
```

**Response (numerador):**

```json
{
    "success": true,
    "message": "Documento firmado y numerado exitosamente",
    "document_id": "uuid",
    "document_status": "signed",
    "is_numerator": true,
    "official_number": "INF-2025-0001234-SMG-ADGEN",
    "signed_pdf_url": "https://..."
}
```

**Archivo:** `endpoints/documents/super_sign.py`

---

## Rechazar Documento

### `POST /documents/{document_id}/reject`

Permite a cualquier firmante rechazar un documento con un motivo.

**Request:**

```json
{
    "reason": "El documento tiene errores en el presupuesto"
}
```

**Response (200):**

```json
{
    "success": true,
    "message": "Documento rechazado exitosamente",
    "rejection_id": "uuid"
}
```

**Archivo:** `endpoints/documents/reject_document.py`

---

## Eliminar Documento

### `DELETE /documents/{document_id}`

Soft delete. Solo el creador puede eliminar documentos en estado `draft` o `rejected`.

**Response (200):**

```json
{
    "success": true,
    "message": "Documento eliminado exitosamente",
    "unlinked_cases": 2
}
```

**Archivo:** `endpoints/documents/delete_document.py`

---

## Detalles Unificados

### `GET /documents/{document_id}/details`

Obtiene detalles de un documento en cualquier estado, delegando al servicio apropiado.

**Response:** `UnifiedDocumentDetailsResponse` con `state_category` ("editing" o "signing") y detalles especificos.

**Archivo:** `endpoints/documents/unified_details.py`

---

## Buscar Documento Oficial

### `GET /api/v1/documents/search-official/{doc_number}`

Busca documento oficial por numero exacto. Sin restricciones de usuario.

**Formato del numero:** `TIPO-AÑO-NUMERO-CIUDAD-SECTOR` (ej: `INF-2025-0001234-SMG-ADGEN`)

**Response:**

```json
{
    "found": true,
    "document": { ... },
    "search_term": "INF-2025-0001234-SMG-ADGEN"
}
```

**Archivo:** `endpoints/documents/search_official.py`
