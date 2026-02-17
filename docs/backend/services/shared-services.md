# Shared Services

Servicios compartidos: clientes de microservicios externos, utilidades de configuracion y datos de usuario.

**Ubicacion:** `services/shared/`

## Modulos

```
shared/
├── pdfcomposer_api.py  # Cliente GDI-PDFComposer (6 endpoints)
├── notary_api.py       # Cliente GDI-Notary (firma digital)
├── settings_utils.py   # Configuracion del tenant con cache
├── signer_data.py      # Datos completos del firmante
├── user_data.py        # Datos del usuario
├── user_queries.py     # Queries de usuarios
├── sector_utils.py     # Utilidades de sectores
├── external_api.py     # Funciones de integracion legacy
├── pdf_utils.py        # Manipulacion de PDFs (SignPage)
├── retry.py            # Logica de reintentos
└── resume_trigger.py   # Trigger de resumen IA
```

---

## PDFComposer API (`pdfcomposer_api.py`)

Cliente HTTP async para GDI-PDFComposer. Genera PDFs desde templates HTML.

### Endpoints Disponibles

| Funcion | Endpoint PDFComposer | Uso |
|---------|---------------------|-----|
| `call_pdfcomposer_preview_pdf()` | `/preview-pdf/` | Preview con marca de agua |
| `call_pdfcomposer_note_preview()` | `/note-preview/` | Preview NOTA con marca de agua |
| `call_pdfcomposer_note_final()` | `/note/` | NOTA final sin marca de agua |
| `call_pdfcomposer_create_case()` | `/create-case/` | Caratula CAEX |
| `call_pdfcomposer_create_transfer()` | `/move/` | Pase PV |
| `call_pdfcomposer_import()` | `/import/` | Procesar PDF importado |

### Patron Comun

Todas las funciones siguen el mismo patron:

```python
async def call_pdfcomposer_preview_pdf(document_data: Dict) -> bytes:
    # 1. Validar campos requeridos
    missing_fields = [f for f in required_fields if not document_data.get(f)]
    if missing_fields:
        raise ValidationError(f"Campos faltantes: {', '.join(missing_fields)}")

    # 2. Obtener credenciales desde ENV
    pdfcomposer_url = os.getenv('PDFCOMPOSER_URL')
    pdfcomposer_api_key = os.getenv('PDFCOMPOSER_API_KEY')

    # 3. Preparar payload (multipart/form-data)
    pdfcomposer_data = {
        "urlLogo": logo_url,
        "NameAcronyType": document_data["document_type_acronym"],
        "TypeDocument": document_data["document_type_name"],
        "Reference": document_data["reference"],
        "Text": json.dumps({"html": html_content})
    }

    # 4. Llamar con retry (2 intentos, 1s entre ellos)
    async with httpx.AsyncClient(timeout=90.0) as client:
        for attempt in range(2):
            response = await client.post(url, data=pdfcomposer_data, headers=headers)
            pdf_bytes = await response.aread()
            # Validar: size > 0, magic bytes %PDF, max 10MB
            return pdf_bytes
```

### Campos por Endpoint

=== "Caratula `/create-case/`"

    ```python
    {
        "urlLogo": "https://...",
        "NameAcronyType": "CAEX",
        "document_type": "Caratula de Expediente",
        "reference": "Habilitacion comercial",
        "case_number": "EE-2025-00001-SMG-ADGEN",
        "acrony_case_type": "EXP-ADM",
        "case_type": "Expediente Administrativo",
        "case_motive": "Habilitacion de panaderia",
        "initiating_division": "Mesa de Entradas",
        "creator": "Juan Perez"
    }
    ```

=== "Pase `/move/`"

    ```python
    {
        "urlLogo": "https://...",
        "NameAcronyType": "PV",
        "document_type": "Pase de Vista",
        "reference": "Transferencia de expediente",
        "tipo_movimiento": "Transferencia",
        "area_requiriente": "Mesa de Entradas",
        "area_receptora": "Obras Publicas",
        "motivo": "Cambio de competencia"
    }
    ```

=== "Import `/import/`"

    ```python
    files = {'pdf_file': (filename, pdf_bytes, 'application/pdf')}
    data = {
        'urlLogo': "https://...",
        'NameAcronyType': "ANEXO",
        'document_type': "Anexo",
        'reference': "Plano de obra"
    }
    ```

---

## Notary API (`notary_api.py`)

Cliente HTTP async para GDI-Notary. Firma digital de PDFs.

### `call_notary_sign_pdf()`

```python
async def call_notary_sign_pdf(
    pdf_bytes: bytes,
    signer_name: str,
    signer_seal: str,
    signer_department: str,
    signer_municipality: str,
    official_number: str,       # Vacio para firmante comun
    city: str = "LATAM",        # Vacio para firmante comun
    stamp_position: str = "",   # "last" para importados
    *,
    tenant_id: str = None       # Para firma PAdES
) -> bytes:
```

**Manejo automatico de FULLPAGE:**

```
Intento 1: Enviar PDF original
    |
    +-- 200 OK --> Retornar PDF firmado
    |
    +-- 400 FULLPAGE --> Agregar SignPage.pdf --> Intento 2
                                                     |
                                                     +-- 200 OK --> PDF firmado
                                                     +-- Error --> Raise
```

`SignPage.pdf` contiene un marcador `end-text` que indica a Notary donde colocar la firma.

**Configuracion:**

| Variable | Descripcion |
|----------|-------------|
| `NOTARY_URL` | URL del microservicio Notary |
| `NOTARY_API_KEY` | API Key para autenticacion |

**Timeout:** 120 segundos (firmas pueden ser lentas).

---

## Settings Utils (`settings_utils.py`)

Acceso centralizado a configuraciones del tenant con cache de 5 minutos.

### `get_tenant_settings()`

```python
def get_tenant_settings(schema_name: str) -> dict:
    """
    Retorna: {logo_url, isologo_url, city, municipality_name}
    Cache TTL: 300 segundos (5 minutos)
    """
```

### Funciones Helper

```python
def get_logo_url(*, schema_name: str) -> str:
    """Logo con fallback a DEFAULT_LOGO_URL."""

def get_city_from_settings(cursor=None, *, schema_name: str = None) -> str:
    """
    Ciudad con fallback a DEFAULT_CITY.
    Con cursor: usa transaccion existente (sin cache)
    Sin cursor: usa cache de 5 minutos
    """

def invalidate_settings_cache(*, schema_name: Optional[str] = None):
    """Invalidar cache tras UPDATE a settings."""
```

---

## Signer Data (`signer_data.py`)

Obtiene datos completos del firmante necesarios para Notary.

```python
def get_signer_data(user_id: str, *, schema_name: str) -> Dict[str, str]:
    """
    Retorna:
    {
        "full_name": "Juan Perez",
        "seal": "Subsecretario de Gestion",
        "department_name": "Administracion General",
        "municipality_name": "Municipalidad de Test"
    }
    """
```

---

## PDF Utils (`pdf_utils.py`)

### `add_blank_page_to_pdf()`

Agrega una pagina con marcador `end-text` a un PDF cuando Notary responde FULLPAGE.

```python
def add_blank_page_to_pdf(pdf_bytes: bytes) -> bytes:
    """
    Usa SignPage.pdf como template (contiene marcador "end-text").
    Retorna PDF con pagina adicional para firma.
    """
```

---

## Resume Trigger (`resume_trigger.py`)

Encola generacion de resumen IA despues de enviar a firma.

```python
def enqueue_resume_fire_and_forget(document_id: str, schema_name: str):
    """
    Fire-and-forget: no bloquea el proceso de firma.
    El resumen se genera asincrono y se guarda en document_draft.resume.
    """
```
