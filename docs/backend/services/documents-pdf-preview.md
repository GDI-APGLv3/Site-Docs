# Documents Preview & PDF

Servicios para previsualizacion de documentos y generacion de PDFs.

**Ubicacion:** `services/documents/preview/`

## Arquitectura

```
preview/
├── core.py              # Orquestador principal
├── data_fetcher.py      # Obtiene datos completos del documento
├── document_builder.py  # Construye payload para PDFComposer
└── auto_save.py         # Auto-guardado antes de preview
```

---

## Preview Core (`preview/core.py`)

### `generate_document_preview()`

Orquesta el proceso completo de preview. Soporta tres tipos de documentos.

```python
async def generate_document_preview(
    document_id: str, *, schema_name: str
) -> Dict[str, Any]:
```

**Flujo por tipo de documento:**

=== "HTML (estandar)"

    1. Auto-save si es necesario
    2. Obtener datos del documento via `PreviewDataFetcher`
    3. Llamar a PDFComposer `/preview-pdf/`
    4. Retornar `pdf_content` (bytes) con `is_imported: False`

=== "NOTA"

    1. Auto-save si es necesario
    2. Obtener datos del documento
    3. Formatear recipients para PDF (`"Dept#Sector, Dept2#Sector2"`)
    4. Llamar a PDFComposer `/note-preview/` con `para` y `cc`
    5. Retornar `pdf_content` (bytes) con `is_imported: False`

=== "Importado"

    1. Auto-save (no-op para importados)
    2. Obtener datos del documento
    3. Generar URL firmada desde R2 bucket `tosign`
    4. Retornar `pdf_url` (string) con `is_imported: True`

**Respuesta:**

```python
# Para HTML/NOTA:
{
    "success": True,
    "document_id": "uuid",
    "document_data": {...},       # Metadata del documento
    "pdf_content": b"%PDF...",    # Bytes del PDF
    "is_imported": False
}

# Para Importado:
{
    "success": True,
    "document_id": "uuid",
    "document_data": {...},
    "pdf_url": "https://r2.cloudflare.com/...",  # URL firmada temporal
    "is_imported": True
}
```

---

## Data Fetcher (`preview/data_fetcher.py`)

Clase `PreviewDataFetcher` que obtiene todos los datos necesarios para generar un preview.

```python
class PreviewDataFetcher:
    def __init__(self, *, schema_name: str):
        self.schema_name = schema_name

    def get_complete_document_data(self, document_id: str) -> Dict:
        """Obtiene datos completos incluyendo firmantes y tipo."""

    def _fetch_document_basic_info(self, document_id: str) -> Dict:
        """Obtiene datos basicos desde document_draft."""
```

**Datos obtenidos:**

| Campo | Tabla | Uso |
|-------|-------|-----|
| `reference` | document_draft | Titulo del PDF |
| `content` | document_draft | Cuerpo HTML |
| `type_acronym` | document_types | Header del PDF (ej: "INFORME") |
| `type_name` | document_types | Subtitulo del PDF |
| `source_type` | document_types | Determina flujo (HTML/Importado) |
| `municipality_logo_url` | settings | Logo en header del PDF |

---

## Integracion con PDFComposer

El preview llama a diferentes endpoints de PDFComposer segun el tipo:

| Tipo | Endpoint | Marca de agua |
|------|----------|---------------|
| HTML | `/preview-pdf/` | Si (watermark "PREVISUALIZACION") |
| NOTA preview | `/note-preview/` | Si |
| NOTA final | `/note/` | No |
| Caratula | `/create-case/` | No |
| Pase | `/move/` | No |
| Importado | `/import/` | No |

### Payload para `/preview-pdf/`

```python
pdfcomposer_data = {
    "urlLogo": logo_url,
    "NameAcronyType": "INF",             # Acronimo
    "TypeDocument": "Informe",            # Nombre completo
    "Reference": "Informe sobre ...",     # Referencia
    "Text": '{"html": "<p>contenido</p>"}'  # JSON string con clave html
}
```

!!! warning "Formato de Text"
    PDFComposer espera `Text` como **JSON string** con formato `{"html": "..."}`. Esto se parsea internamente con `json.loads(Text)`.

### Payload para `/note-preview/`

Agrega campos de destinatarios al payload base:

```python
pdfcomposer_data = {
    # ... mismos campos que preview-pdf ...
    "para": "Finanzas#Tesoreria, Legales#Asesoria",  # Destinatarios TO
    "cc": "RRHH#Personal"  # Destinatarios CC (opcional)
}
```

---

## Auto-Save (`preview/auto_save.py`)

Antes de generar un preview, se ejecuta un auto-guardado para asegurar que el PDF refleje los ultimos cambios del editor.

```python
class AutoSaveHandler:
    def __init__(self, *, schema_name: str):
        self.schema_name = schema_name

    async def handle_auto_save_if_needed(self, document_id: str):
        """Guarda automaticamente si el documento tiene cambios pendientes."""
```

---

## Configuracion de PDFComposer

| Variable | Descripcion |
|----------|-------------|
| `PDFCOMPOSER_URL` | URL del microservicio (Railway internal) |
| `PDFCOMPOSER_API_KEY` | API Key para autenticacion |

**Timeout:** 90 segundos (generacion de PDF puede ser lenta).

**Reintentos:** 1 reintento con espera de 1 segundo.

**Validaciones del PDF retornado:**

- Tamano > 0 bytes
- Magic bytes `%PDF`
- Tamano maximo 10MB (30MB para import)
