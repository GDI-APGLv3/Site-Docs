# Endpoints de Notas

Endpoints para el sistema de notas internas entre sectores. Prefijo: `/notes`.

Las notas son documentos tipo NOTA con destinatarios (TO, CC, BCC). A diferencia de otros documentos, las notas tienen un flujo de envio y recepcion entre sectores.

## Notas Recibidas

### `GET /notes/received`

Lista notas recibidas por el sector del usuario.

**Query Parameters:**

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| `page` | int | Pagina |
| `page_size` | int | Resultados por pagina |
| `search` | string | Buscar en referencia |

**Response incluye por nota:**

- Datos de la nota (referencia, contenido, fecha)
- Informacion del remitente (sector, usuario)
- Tipo de recepcion: `to`, `cc`, `bcc`
- Estado de lectura: `is_read`

**Archivo:** `endpoints/notes/received.py`

---

## Notas Enviadas

### `GET /notes/sent`

Lista notas enviadas por el usuario autenticado.

**Archivo:** `endpoints/notes/sent.py`

---

## Notas Archivadas

### `GET /notes/archived`

Lista notas archivadas por el usuario.

**Archivo:** `endpoints/notes/archived.py`

---

## Detalle de Nota

### `GET /notes/{note_id}`

Obtiene detalle completo de una nota con todos sus destinatarios y contenido.

**Response incluye:**

- Contenido HTML de la nota
- Lista de destinatarios TO, CC, BCC
- Datos del remitente y sector
- Fecha de envio y estado

**Archivo:** `endpoints/notes/detail.py`

---

## Archivar Nota

### `POST /notes/{note_id}/archive`

Archiva una nota recibida. La nota deja de aparecer en la bandeja de recibidas.

**Archivo:** `endpoints/notes/archive.py`

---

## Flujo de Notas

```
1. POST /documents (crear NOTA con recipients)
    {
        "document_type_acronym": "NOTA",
        "reference": "Convocatoria a reunion",
        "recipients": {
            "to": ["uuid-sector-1"],
            "cc": ["uuid-sector-2"]
        }
    }

2. PATCH /documents/{id}/save (guardar contenido HTML)

3. POST /documents/{id}/start-signing-process (enviar a firma)

4. POST /documents/{id}/super-sign (firmar)
    -> Al oficializarse, la NOTA aparece en /notes/received
       para los sectores destinatarios

5. GET /notes/received (sectores destino ven la nota)

6. POST /notes/{id}/archive (archivar cuando ya fue leida)
```
