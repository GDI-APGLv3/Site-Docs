# Documents Signing

Servicios de firma digital: inicio de proceso, firma comun, firma numerador y firma unificada.

**Ubicacion:** `services/documents/signing/`

## Arquitectura de Firma

```
signing/
├── signing.py          # start_signing + sign_document (comun)
├── numerator.py        # sign_document_as_numerator (numerador)
├── unified_signing.py  # super_sign_document (detecta rol automaticamente)
└── details_builder.py  # Construye pantalla de firma
```

## Flujo Completo de Firma

```
  Creador                  Firmantes Comunes           Numerador
     |                           |                        |
  start_signing()                |                        |
     |                           |                        |
  [Genera PDF con PDFComposer]   |                        |
  [Sube a R2 bucket tosign]      |                        |
  [Estado: sent_to_sign]         |                        |
     |                           |                        |
     |                    super_sign()                     |
     |                    (firma comun)                    |
     |                           |                        |
     |                    [Descarga de R2]                 |
     |                    [Firma con Notary]               |
     |                    [Sobrescribe en R2]              |
     |                           |                        |
     |                    (repite por firmante)            |
     |                           |                        |
     |                           |               super_sign()
     |                           |               (numerador)
     |                           |                        |
     |                           |               [Genera numero oficial]
     |                           |               [Firma con Notary + numero]
     |                           |               [Sube a R2 bucket oficial]
     |                           |               [Estado: signed]
```

---

## Inicio de Firma (`signing.py`)

### `start_document_signing_process()`

```python
async def start_document_signing_process(
    document_id: str, user_id: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Solo el creador puede iniciar el proceso.**

**Pasos:**

1. Verificar que el documento existe y esta en estado editable (`draft` o `rejected`)
2. Verificar que `user_id == created_by`
3. Obtener logo del municipio desde `settings`
4. Si es NOTA: validar que tenga recipients configurados
5. Obtener lista de firmantes
6. Generar PDF con PDFComposer (async)
7. Actualizar estado a `sent_to_sign` y registrar `sent_by`
8. Enviar invitaciones a usuarios inactivos (best-effort)
9. Encolar generacion de resumen IA (fire-and-forget)

**Integraciones:**

| Servicio | Funcion |
|----------|---------|
| PDFComposer | Genera PDF final sin marca de agua |
| R2 tosign | Almacena PDF para proceso de firma |
| eMailService | Invita usuarios inactivos (estado=2) |
| Resume trigger | Genera resumen IA del contenido |

---

## Firma Comun (`signing.py`)

### `sign_document()`

Firma de un firmante **no numerador**.

```python
async def sign_document(
    document_id: str, user_id: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Pasos:**

1. Verificar que el usuario es firmante y no ha firmado
2. Descargar PDF desde R2 bucket `tosign`
3. Obtener datos del firmante (nombre, sello, departamento, municipalidad)
4. Firmar con Notary API (`official_number=""`, `city=""`)
5. Sobrescribir PDF en R2 `tosign`
6. Actualizar `document_signers.status = 'signed'`

!!! note "Firmante comun vs numerador"
    El firmante comun firma con `official_number` y `city` **vacios**. Solo el numerador agrega el numero oficial y la ciudad al sello.

---

## Firma Numerador (`numerator.py`)

### `sign_document_as_numerator()`

Proceso completo del numerador: numera, firma y oficializa.

```python
async def sign_document_as_numerator(
    document_id: str, user_id: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Pasos:**

1. **Validaciones previas**: Es numerador, documento en `sent_to_sign`, todos los comunes firmaron
2. **Validar rank y departamento**: El numerador necesita rank suficiente para el tipo de documento
3. **Verificar numero existente**: Si ya existe (caso reintento), reutilizar
4. **Generar numero oficial**: Con advisory lock ultra corto (10-20ms)
5. **Insertar en `official_documents`**: Con contenido, firmantes, sector_ids, resume
6. **Firmar con Notary**: Con `official_number` y `city` reales
7. **Subir a R2 bucket `oficial`**: Filename = `{official_number}.pdf`
8. **Eliminar de R2 `tosign`**: Soft-fail
9. **Actualizar tablas**: `document_signers`, `document_draft.status = 'signed'`
10. **Commit o rollback automatico**

**Validacion de Rank:**

```sql
-- Verifica que el rank del usuario sea suficiente
-- para el tipo de documento (ej: Decreto requiere Intendente)
CASE
    WHEN rr.level IS NULL THEN true      -- Sin restriccion
    WHEN ur.level IS NULL THEN false     -- Usuario sin rank
    WHEN ur.level <= rr.level THEN true  -- Rank suficiente
    ELSE false
END as has_rank_permission
```

---

## Firma Unificada (`unified_signing.py`)

### `super_sign_document()`

Punto de entrada unico que detecta automaticamente si el usuario es firmante comun o numerador.

```python
async def super_sign_document(
    document_id: str, user_id: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Flujo de decision:**

```python
# Query inicial obtiene todo lo necesario
result = cursor.execute(get_signer_role_and_document_status_query(), ...)
# Retorna: is_numerator, signer_status, doc_status, pending_common_signers

# Validaciones comunes
if doc_status != 'sent_to_sign':
    raise DocumentStateError(...)
if signer_status not in ['pending', None]:
    raise ValidationError("Ya firmo")

# Bifurcacion
if not is_numerator:
    result = await sign_document(...)        # Rama A: Comun
else:
    if pending_common_signers > 0:
        raise ValidationError("Faltan firmantes")
    result = await sign_document_as_numerator(...)  # Rama B: Numerador
```

**Respuesta unificada (SuperSignResponse):**

```python
{
    "success": True,
    "message": "Documento firmado exitosamente",
    "document_id": "uuid",
    "signature_id": "uuid",
    "document_status": "signed",  # o "sent_to_sign" si es comun
    "signed_at": "2025-01-15T10:30:00",
    "is_numerator": True,
    "official_number": "IF-2025-0000157-SMG-ADGEN",  # null si comun
    "signed_pdf_url": "https://..."  # null si comun
}
```

---

## Interaccion con Notary

Todas las firmas pasan por `services/shared/notary_api.py`:

```python
signed_pdf_bytes = await call_notary_sign_pdf(
    pdf_bytes=pdf_bytes,
    signer_name="Juan Perez",
    signer_seal="Subsecretario de Gestion",
    signer_department="Administracion General",
    signer_municipality="Municipalidad de Test",
    official_number="IF-2025-0000157-SMG-ADGEN",  # Vacio para comun
    city="San Martin de los Andes",                 # Vacio para comun
    stamp_position="last",  # Solo para importados
    tenant_id=schema_name   # Para firma PAdES
)
```

!!! warning "FULLPAGE"
    Si Notary responde con error FULLPAGE (sin espacio para firma), se agrega automaticamente una pagina con marcador `end-text` y se reintenta.
