# Validaciones

El modulo `app/validators.py` centraliza todas las validaciones de entrada del servicio Notary.
Cada validacion lanza `HTTPException` con codigo de error especifico.

## Validacion de PDF

### validate_pdf_format

Valida formato, tamano y signatura del PDF recibido.

```python
def validate_pdf_format(pdf_file: UploadFile) -> bytes:
```

| Validacion | Condicion | Error |
|------------|-----------|-------|
| Tamano maximo | `pdf_file.size > 10 MB` | `400 FILE_TOO_LARGE` |
| Lectura | No se puede leer el archivo | `400 INVALID_PDF_FILE` |
| Signatura PDF | No comienza con `%PDF` | `400 INVALID_PDF_FORMAT` |

```python
# Tamano maximo: 10 MB
if pdf_file.size > MAX_PDF_SIZE_MB * 1024 * 1024:
    raise HTTPException(status_code=400, detail="PDF file too large")

# Signatura valida
if not pdf_content.startswith(b'%PDF'):
    raise HTTPException(status_code=400, detail="Invalid PDF format")
```

---

## Validacion de parametros de firma

### validate_signature_params

Valida los 4 parametros obligatorios de la firma.

```python
def validate_signature_params(
    name: str,        # 1-100 chars
    seal: str,        # 1-50 chars
    department: str,  # 1-100 chars
    entity: str       # 1-100 chars
) -> dict:
```

| Parametro | Limite | Validaciones |
|-----------|--------|-------------|
| `name` | 1-100 chars | No vacio, no excede 100 |
| `seal` | 1-50 chars | No vacio, no excede 50 |
| `department` | 1-100 chars | No vacio, no excede 100 |
| `entity` | 1-100 chars | No vacio, no excede 100 |

Retorna los parametros con `strip()` aplicado. Si falla, lanza `400 INVALID_PARAMETERS`
con la lista de errores.

---

## Validacion de estampado

### validate_stamp_params

Valida los parametros opcionales de estampado.

```python
def validate_stamp_params(
    document_number: Optional[str],  # max 40 chars
    city: Optional[str]              # max 50 chars, requerido si document_number
) -> Optional[dict]:
```

| Regla | Descripcion |
|-------|-------------|
| `city` es obligatorio | Si se envia `document_number`, `city` es requerido |
| `document_number` max | 40 caracteres |
| `city` max | 50 caracteres |

Error: `400 INVALID_STAMP_PARAMETERS`

### validate_stamp_position

Valida que `stamp_position` sea un valor valido.

```python
def validate_stamp_position(stamp_position: Optional[str]) -> None:
    valid_positions = ["first", "last"]
    if stamp_position not in valid_positions:
        raise HTTPException(
            status_code=400,
            detail=f"stamp_position must be 'first' or 'last'"
        )
```

Error: `400 INVALID_STAMP_POSITION`

---

## Validacion de tenant_id

### validate_tenant_id

Previene path traversal validando el formato del `tenant_id`.

```python
TENANT_ID_PATTERN = re.compile(r'^[a-zA-Z0-9_-]+$')

def validate_tenant_id(tenant_id: str) -> str:
    if not tenant_id or not TENANT_ID_PATTERN.match(tenant_id):
        raise HTTPException(
            status_code=400,
            detail="Invalid tenant_id format"
        )
    return tenant_id
```

**Caracteres permitidos:** `a-z`, `A-Z`, `0-9`, `-`, `_`

!!! danger "Prevencion de path traversal"
    Sin esta validacion, un `tenant_id` como `../../etc/passwd` podria
    hacer que el sistema intente cargar un archivo fuera del directorio
    de certificados. Esta validacion trabaja en conjunto con la verificacion
    de contencion en `certificate_loader.get_certificate_path()`.

Error: `400 INVALID_TENANT_ID`

---

## Sanitizacion de nombres de archivo

### sanitize_filename

Reemplaza caracteres especiales en nombres de archivo.

```python
SPECIAL_CHARS = r'/\:*?"<>|'

def sanitize_filename(filename: str) -> str:
    sanitized = filename
    for char in SPECIAL_CHARS:
        sanitized = sanitized.replace(char, '-')
    return sanitized
```

**Ejemplo:** `IF/2025\doc.pdf` se convierte en `IF-2025-doc.pdf`

---

## Autenticacion (auth.py)

El modulo `app/auth.py` implementa la validacion de API Key:

```python
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def validate_api_key(api_key: str = Security(api_key_header)):
    if api_key != API_KEY:
        raise HTTPException(status_code=401, detail="Invalid or missing API key")
    return api_key
```

Se usa como dependencia de FastAPI en los endpoints protegidos:

```python
@app.post("/sign-pdf")
async def sign_pdf(
    ...,
    api_key: str = Depends(validate_api_key)
):
```

Error: `401 INVALID_API_KEY`

---

## Resumen de codigos de error

| Codigo | Error Code | Modulo | Descripcion |
|--------|------------|--------|-------------|
| 400 | `FILE_TOO_LARGE` | validators | PDF > 10 MB |
| 400 | `INVALID_PDF_FILE` | validators | No se puede leer |
| 400 | `INVALID_PDF_FORMAT` | validators | No es PDF valido |
| 400 | `INVALID_PARAMETERS` | validators | Parametros de firma invalidos |
| 400 | `INVALID_STAMP_PARAMETERS` | validators | Estampado invalido |
| 400 | `INVALID_STAMP_POSITION` | validators | Position no es first/last |
| 400 | `INVALID_TENANT_ID` | validators | Caracteres no permitidos |
| 400 | `CERTIFICATE_NOT_FOUND` | main | Sin cert + FALLBACK=false |
| 400 | `CERTIFICATE_LOAD_ERROR` | main | Error al cargar cert |
| 401 | `INVALID_API_KEY` | auth | API Key invalida |
| 500 | `PADES_ERROR` | main | Error en firma PAdES |
