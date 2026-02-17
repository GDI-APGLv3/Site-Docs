# Certificados digitales

El modulo `app/certificate_loader.py` maneja la carga y validacion de certificados
PKCS#12 (.p12) organizados por `tenant_id`.

## Estructura de archivos

```
certs/
├── 100_test.p12           # Certificado del tenant 100_test
├── municipio_demo.p12     # Certificado del tenant municipio_demo
├── passwords.json         # Mapeo tenant_id -> password
└── README.md
```

### passwords.json

```json
{
  "100_test": "test123",
  "municipio_demo": "otra_clave"
}
```

## Modelo LoadedCertificate

```python
@dataclass
class LoadedCertificate:
    private_key: PrivateKeyTypes     # Clave privada RSA
    certificate: Certificate         # Certificado X.509
    additional_certs: Optional[list] # Certificados intermedios
    tenant_id: str                   # ID del tenant
    path: Path                       # Ruta al archivo .p12
```

## Funciones principales

### certificate_exists

Verifica si existe un certificado para un tenant:

```python
def certificate_exists(tenant_id: str) -> bool:
    cert_path = get_certificate_path(tenant_id)
    return cert_path.exists()
```

### load_certificate

Carga un certificado PKCS#12 completo:

```python
def load_certificate(tenant_id: str) -> LoadedCertificate:
    cert_path = get_certificate_path(tenant_id)

    # Obtener password de passwords.json
    passwords = load_passwords()
    password = passwords.get(tenant_id)

    # Cargar PKCS#12 con cryptography
    private_key, certificate, additional_certs = pkcs12.load_key_and_certificates(
        p12_data,
        password.encode('utf-8'),
        default_backend()
    )

    return LoadedCertificate(
        private_key=private_key,
        certificate=certificate,
        additional_certs=list(additional_certs) if additional_certs else None,
        tenant_id=tenant_id,
        path=cert_path
    )
```

### validate_certificate

Valida vigencia y permisos del certificado:

```python
def validate_certificate(cert: LoadedCertificate) -> Tuple[bool, str]:
    now = datetime.now(timezone.utc)

    # Verificar vigencia temporal
    if now < cert.certificate.not_valid_before_utc:
        return False, "Certificado aun no es valido"
    if now > cert.certificate.not_valid_after_utc:
        return False, "Certificado expirado"

    # Verificar key_usage (digital_signature)
    key_usage = cert.certificate.extensions.get_extension_for_oid(
        ExtensionOID.KEY_USAGE
    )
    if not key_usage.value.digital_signature:
        return False, "Certificado no permite firma digital"

    return True, "Certificado valido"
```

### get_certificate_info

Obtiene informacion de un certificado sin exponer la clave privada:

```python
def get_certificate_info(tenant_id: str) -> dict:
    # Retorna: exists, subject, issuer, not_valid_before,
    #          not_valid_after, is_valid, serial_number
```

### list_available_certificates

Lista todos los tenant_ids con certificados:

```python
def list_available_certificates() -> list:
    certs_dir = Path(CERTS_DIR)
    return [p.stem for p in certs_dir.glob("*.p12")]
```

## Seguridad: prevencion de path traversal

La funcion `get_certificate_path` incluye verificacion de contencion de path:

```python
def get_certificate_path(tenant_id: str) -> Path:
    certs_dir = Path(CERTS_DIR).resolve()
    cert_path = (certs_dir / f"{tenant_id}.p12").resolve()

    # Verificar que el path resultante no escape del directorio
    if not cert_path.is_relative_to(certs_dir):
        raise CertificateError(
            "Invalid tenant_id: path traversal detected"
        )

    return cert_path
```

Ademas, `validators.py` valida el formato del `tenant_id` con regex:

```python
TENANT_ID_PATTERN = re.compile(r'^[a-zA-Z0-9_-]+$')
```

## Generar certificado de prueba

El script `scripts/generate_test_cert.py` genera certificados auto-firmados:

```bash
python scripts/generate_test_cert.py \
  --tenant 100_test \
  --cn "GESTION DOCUMENTAL INTELIGENTE" \
  --org "Municipalidad del Futuro" \
  --password test123
```

### Opciones del script

| Opcion | Default | Descripcion |
|--------|---------|-------------|
| `--tenant` / `-t` | (requerido) | ID del tenant |
| `--password` / `-p` | `test123` | Password del .p12 |
| `--output` / `-o` | `../certs` | Directorio de salida |
| `--days` / `-d` | `365` | Dias de validez |
| `--cn` | Auto-generado | Common Name |
| `--org` | Auto-generado | Organization |
| `--no-passwords-file` | - | No actualizar passwords.json |

### Propiedades del certificado generado

| Propiedad | Valor |
|-----------|-------|
| Algoritmo | RSA 2048 bits |
| Hash | SHA-256 |
| Key Usage | Digital Signature, Content Commitment |
| Extended Key Usage | Code Signing |
| Validez | 365 dias (configurable) |
| Pais | AR |
| Estado | Buenos Aires |

!!! danger "Solo para pruebas"
    Los certificados generados con este script son **auto-firmados** y
    no son validos para produccion. En produccion, usar certificados
    emitidos por una Autoridad Certificadora (CA) reconocida.

## Excepciones

| Excepcion | Causa |
|-----------|-------|
| `CertificateNotFoundError` | No existe archivo .p12 para el tenant |
| `PasswordNotFoundError` | No hay password en passwords.json |
| `CertificateLoadError` | Error al cargar .p12 (password incorrecto, formato invalido) |
| `CertificateError` | Error generico de certificado |

## Roadmap: Certificados en Cloudflare R2

Actualmente los certificados se almacenan localmente y se copian en la imagen Docker.
La arquitectura futura planea migrarlos a Cloudflare R2:

```
gdi-certificates/
├── {tenant_id}/
│   ├── certificate.p12.enc    # Certificado encriptado AES-256
│   └── metadata.json          # Info del certificado
```

Cambios previstos:

- Nuevo `R2CertificateLoader` en `certificate_loader.py`
- Variable `CERTS_STORAGE=local|r2`
- Encripcion AES-256 en reposo
- Rotacion sin redeploy
