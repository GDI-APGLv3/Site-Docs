# Endpoints

Todos los endpoints (excepto `/health`) requieren el header `X-API-Key` para autenticacion.
Los datos se envian como `multipart/form-data` y la respuesta es siempre un archivo PDF (`application/pdf`).

---

## GET /health

Health check basico. No requiere autenticacion.

**Response:**
```json
{"status": "healthy"}
```

---

## GET /health/details

Health check detallado con estado de Gotenberg. Requiere API Key.

**Response:**
```json
{
  "status": "healthy",
  "service": "pdfcomposer",
  "version": "2.3.0",
  "gotenberg": "ok"
}
```

!!! info "Estados posibles"
    - `healthy`: Gotenberg responde correctamente
    - `degraded`: Gotenberg no responde o esta caido

---

## POST /preview-pdf/

Genera un PDF con marca de agua diagonal "PREVISUALIZACION".

**Parametros (Form Data):**

| Campo | Tipo | Requerido | Descripcion |
|-------|------|-----------|-------------|
| `urlLogo` | string | No | URL publica de la imagen del logo |
| `NameAcronyType` | string | Si | Acronimo del tipo (ej. `GDI`) |
| `TypeDocument` | string | Si | Tipo de documento (ej. `INFORME`) |
| `Reference` | string | Si | Referencia del documento |
| `Text` | string (JSON) | Si | Contenido HTML: `{"html": "<p>...</p>"}` |

**Template:** `plantilla.html`

**Response:** PDF con `Content-Disposition: attachment; filename={uuid}.pdf`

**Ejemplo:**

```bash
curl -X POST "http://localhost:8002/preview-pdf/" \
     -H "X-API-Key: miapikey" \
     -F "NameAcronyType=GDI" \
     -F "TypeDocument=INFORME" \
     -F "Reference=Ref: Solicitud de presupuesto 2025" \
     -F 'Text={"html": "<p>Contenido del documento.</p>"}' \
     --output preview.pdf
```

---

## POST /generate-pdf/

Genera el documento PDF final (sin marca de agua). Campos de fecha y numero en blanco (`color: white`) para completar a mano. Opcionalmente anexa un PDF adjunto al final.

**Parametros (Form Data):**

| Campo | Tipo | Requerido | Descripcion |
|-------|------|-----------|-------------|
| `urlLogo` | string | No | URL publica de la imagen del logo |
| `NameAcronyType` | string | Si | Acronimo del tipo |
| `TypeDocument` | string | Si | Tipo de documento |
| `Reference` | string | Si | Referencia del documento |
| `Text` | string (JSON) | Si | Contenido HTML: `{"html": "..."}` |
| `attachment_file` | File (PDF) | No | PDF adjunto a anexar al final (max 25 MB) |

**Template:** `generate-pdf.html`

**Errores adicionales:**

| Codigo | Detalle |
|--------|---------|
| 400 | El archivo adjunto no es un PDF valido |
| 413 | Archivo adjunto excede 25 MB |

!!! warning "Validacion del adjunto"
    Si se envia `attachment_file`, se valida que:

    1. No exceda 25 MB
    2. Comience con la signatura `%PDF` (magic bytes)
    3. Se pueda procesar con PyMuPDF

---

## POST /create-case/

Genera una caratula de expediente (CAEX). La fecha de caratulacion se genera automaticamente en UTC.

**Parametros (Form Data):**

| Campo | Tipo | Requerido | Descripcion |
|-------|------|-----------|-------------|
| `urlLogo` | string | No | URL publica del logo |
| `NameAcronyType` | string | Si | Acronimo del tipo |
| `document_type` | string | Si | Tipo de documento (ej. `CARATULA DE EXPEDIENTE`) |
| `reference` | string | Si | Referencia (ej. `CAEX-2025-0000123-MUNI`) |
| `case_number` | string | Si | Numero de expediente (ej. `EX-2025-0000123-MUNI`) |
| `acrony_case_type` | string | Si | Acronimo del tipo de expediente (ej. `HAB`) |
| `case_type` | string | Si | Tipo de expediente (ej. `Habilitacion Comercial`) |
| `case_motive` | string | Si | Motivo del expediente |
| `initiating_division` | string | Si | Reparticion iniciadora |
| `creator` | string | Si | Nombre del caratulador |

**Template:** `caratula.html`

---

## POST /move/

Genera un pase de vista (PV) de expediente entre areas.

**Parametros (Form Data):**

| Campo | Tipo | Requerido | Descripcion |
|-------|------|-----------|-------------|
| `urlLogo` | string | No | URL publica del logo |
| `NameAcronyType` | string | Si | Acronimo del tipo |
| `document_type` | string | Si | Tipo de documento (ej. `PASE DE VISTA`) |
| `reference` | string | Si | Referencia (ej. `PV-2025-0000456-MUNI`) |
| `tipo_movimiento` | string | Si | Tipo de movimiento |
| `area_requiriente` | string | Si | Area que envia (DE) |
| `area_receptora` | string | Si | Area que recibe (A) |
| `motivo` | string | Si | Motivo del movimiento |

**Template:** `movimiento.html`

---

## POST /import/

Importa un PDF externo, cuenta sus paginas, genera una pagina informativa y devuelve el PDF compilado (original + pagina info al final).

**Parametros (Form Data):**

| Campo | Tipo | Requerido | Descripcion |
|-------|------|-----------|-------------|
| `pdf_file` | File | Si | Archivo PDF a importar |
| `urlLogo` | string | No | URL publica del logo |
| `NameAcronyType` | string | Si | Acronimo del tipo |
| `document_type` | string | Si | Tipo de documento (ej. `DOCUMENTO IMPORTADO`) |
| `reference` | string | Si | Referencia del documento |

**Errores:**

| Codigo | Detalle |
|--------|---------|
| 400 | El archivo no es un PDF valido |
| 413 | PDF excede 25 MB |

---

## POST /note-preview/

Genera una nota con marca de agua de previsualizacion.

**Parametros (Form Data):**

| Campo | Tipo | Requerido | Descripcion |
|-------|------|-----------|-------------|
| `urlLogo` | string | No | URL publica del logo |
| `NameAcronyType` | string | Si | Acronimo del tipo |
| `document_type` | string | Si | Tipo de documento (ej. `NOTA`) |
| `reference` | string | Si | Referencia del documento |
| `para` | string | No | Destinatario principal (PARA) |
| `cc` | string | No | Destinatarios en copia (CC) |
| `Text` | string (JSON) | Si | Contenido HTML: `{"html": "..."}` |

**Template:** `nota_preview.html`

---

## POST /note/

Genera una nota final sin marca de agua.

Mismos parametros que `/note-preview/`.

**Template:** `nota.html`

---

## Codigos de error comunes

| Codigo | Causa | Solucion |
|--------|-------|----------|
| 403 | API Key falta o incorrecta | Verificar header `X-API-Key` |
| 422 | Datos invalidos (Pydantic) | Verificar campos requeridos y formato de `Text` |
| 500 | Error interno o Gotenberg fallo | Revisar logs, verificar `/health/details` |
| 504 | Timeout de Gotenberg | Aumentar `DEFAULT_TIMEOUT` o reiniciar Gotenberg |
