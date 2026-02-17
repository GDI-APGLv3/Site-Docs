# Storage Service

Cliente de Cloudflare R2 para almacenamiento de PDFs con soporte multi-tenant.

**Ubicacion:** `services/storage/cloudflare.py`

## Arquitectura

```
Cloudflare R2 (S3-compatible)
├── Bucket tosign      # PDFs en proceso de firma
│   └── {uuid_sin_guiones}.pdf
└── Bucket oficial     # PDFs firmados oficialmente
    └── {official_number}.pdf
```

Cada tenant tiene sus **propios buckets** configurados en `{schema}.settings`.

---

## CloudflareR2Client

Clase principal que interactua con R2 via boto3 (S3-compatible).

```python
class CloudflareR2Client:
    def __init__(self, bucket_oficial: str = None, bucket_tosign: str = None):
        """
        Inicializa cliente R2 con credenciales desde ENV vars.

        Variables requeridas:
        - CF_R2_ENDPOINT: URL del endpoint Cloudflare R2
        - CF_R2_ACCESS_KEY_ID: Access key
        - CF_R2_SECRET_ACCESS_KEY: Secret key
        """
```

### Metodos

| Metodo | Bucket | Descripcion |
|--------|--------|-------------|
| `get_oficial_url(official_number)` | oficial | URL firmada para documento oficial |
| `get_tosign_url(filename)` | tosign | URL firmada para documento en firma |
| `exists_tosign(filename)` | tosign | Verifica existencia sin descargar |
| `upload_tosign(pdf_bytes, filename)` | tosign | Sube PDF para proceso de firma |
| `upload_oficial(pdf_bytes, filename)` | oficial | Sube PDF firmado oficialmente |
| `delete_tosign(filename)` | tosign | Elimina PDF (idempotente) |

### URLs Firmadas

Todas las URLs generadas son **temporales** con expiracion configurable:

```python
url = self._client.generate_presigned_url(
    'get_object',
    Params={'Bucket': self.bucket_oficial, 'Key': filename},
    ExpiresIn=self.url_expiration  # Default: 600 segundos
)
```

### Convenciones de Filenames

| Bucket | Formato | Ejemplo |
|--------|---------|---------|
| tosign | `{uuid_sin_guiones}.pdf` | `214c5d1695ea4865876de8e826ef3ece.pdf` |
| oficial | `{official_number}.pdf` | `IF-2025-000000157-MT-DGOBR.pdf` |

!!! note "Normalizacion"
    Los metodos agregan `.pdf` automaticamente si no esta presente.

### Upload

```python
def upload_tosign(self, pdf_bytes: bytes, filename: str) -> dict:
    """
    Retorna:
    {
        "status": "success",
        "location": "tosign",
        "filename": "214c5d1695ea4865876de8e826ef3ece.pdf",
        "uploaded_at": "2025-10-22T10:30:00Z",
        "size_bytes": 245678
    }
    """
```

### Delete (Idempotente)

```python
def delete_tosign(self, filename: str) -> dict:
    """
    SOFT-FAIL: Si el archivo no existe, retorna success.
    Util para cleanup de documentos rechazados/eliminados.
    """
```

---

## Multi-Tenant

### `get_tenant_r2_client()`

Obtiene un cliente R2 configurado con los buckets del tenant. Usa cache thread-safe.

```python
def get_tenant_r2_client(*, schema_name: str) -> CloudflareR2Client:
    """
    Lee bucket names desde {schema}.settings y crea cliente.
    Cache thread-safe con TTL de 1 hora.
    """
```

**Flujo:**

1. Verificar cache (thread-safe con `Lock`)
2. Si cache valido (< 1 hora): retornar cliente existente
3. Si no: query a `{schema}.settings` para obtener `bucket_oficial` y `bucket_tosign`
4. Crear nuevo `CloudflareR2Client` con buckets del tenant
5. Guardar en cache

### `get_tenant_settings()`

```python
def get_tenant_settings(schema_name: str) -> Dict[str, str]:
    """Query: SELECT bucket_oficial, bucket_tosign FROM settings LIMIT 1"""
```

### Invalidar Cache

```python
def invalidate_tenant_r2_cache(*, schema_name: str):
    """Usar si se actualizan los settings de buckets."""
```

---

## Variables de Entorno

| Variable | Descripcion | Default |
|----------|-------------|---------|
| `CF_R2_ENDPOINT` | URL del endpoint R2 | Requerido |
| `CF_R2_ACCESS_KEY_ID` | Access key | Requerido |
| `CF_R2_SECRET_ACCESS_KEY` | Secret key | Requerido |
| `CF_R2_BUCKET_OFICIAL` | Bucket oficial (fallback) | `tenant-test-oficial` |
| `CF_R2_BUCKET_TOSIGN` | Bucket tosign (fallback) | `tenant-test-tosign` |
| `CF_R2_SIGN_EXPIRATION` | Expiracion URLs (segundos) | `600` |

---

## Uso en Servicios

```python
# Obtener cliente para el tenant
from services.storage.cloudflare import get_tenant_r2_client

r2_client = get_tenant_r2_client(schema_name=schema_name)

# Subir PDF
r2_client.upload_tosign(pdf_bytes, f"{document_id_no_hyphens}.pdf")

# Obtener URL firmada
url = r2_client.get_oficial_url("IF-2025-0000157-SMG-ADGEN")

# Verificar existencia
exists = r2_client.exists_tosign(f"{document_id_no_hyphens}.pdf")

# Eliminar (idempotente)
r2_client.delete_tosign(f"{document_id_no_hyphens}.pdf")
```
