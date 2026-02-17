# Cases Service

Servicios de expedientes: creacion, transferencia, asignacion, caratulas CAEX y pases PV.

**Ubicacion:** `services/cases/`

## Modulos

```
cases/
├── core.py                      # Funcion base create_case (sync)
├── creation.py                  # Creacion completa (async)
├── cover_creator.py             # Caratulas CAEX automaticas
├── transfer.py                  # Transferencias y asignaciones
├── transfer_document_creator.py # Pases PV automaticos
├── _document_creator_base.py    # Base compartida para CAEX y PV
├── permissions.py               # Permisos por sector
├── validation.py                # Validaciones de expedientes
├── queries.py                   # SQL centralizado
├── retrieval.py                 # Detalle de expediente
├── history.py                   # Historial de movimientos
├── documents.py                 # Documentos vinculados
└── subsanacion.py               # Proceso de subsanacion
```

---

## Creacion de Expedientes

### Flujo Completo

```
POST /api/v1/cases/
       |
       v
  create_case()        [core.py - sync, dentro de transaccion]
       |
       +-- Genera case_number con advisory lock
       +-- INSERT en cases
       +-- INSERT en case_movements (tipo: creation)
       |
       v
  create_case_cover()  [cover_creator.py - async]
       |
       +-- create_document(type="CAEX")
       +-- Genera numero oficial
       +-- Llama PDFComposer /create-case/
       +-- Firma con Notary
       +-- Sube a R2 oficial
       +-- INSERT en official_documents
       +-- Vincula CAEX al expediente
```

### `create_case()` (core.py)

Funcion base **sincrona** para uso dentro de transacciones existentes.

```python
def create_case(
    connection,                    # Conexion BD activa
    case_template_id: str,
    reference: str,
    created_by_user_id: str,
    filing_department_id: str,
    creator_sector_id: str,
    owner_sector_id: Optional[str] = None,
    *,
    schema_name: str
) -> Dict[str, Any]:
```

**Generacion de numero:**

```
EE-{AÑO}-{SECUENCIA:05d}-{MUNICIPIO}-{DEPARTAMENTO}
Ejemplo: EE-2025-00001-SMG-ADGEN
```

### `create_case_cover()` (cover_creator.py)

Crea la caratula CAEX automatica al crear un expediente.

```python
async def create_case_cover(
    case_id: str,
    case_number: str,
    case_reference: str,
    case_template_acronym: str,
    case_template_name: str,
    filing_department_id: str,
    user_id: str,
    *,
    schema_name: str,
    connection=None
) -> Dict[str, Any]:
```

**Pasos:**

1. Crear documento CAEX en draft (`create_document`)
2. Generar numero oficial con advisory lock
3. Llamar a PDFComposer `/create-case/` para generar PDF de caratula
4. Firmar PDF con Notary (incluye numero oficial y ciudad)
5. Subir PDF firmado a R2 bucket oficial
6. INSERT en `official_documents`
7. Vincular documento al expediente (`case_documents`)

---

## Transferencias

### `transfer_case()`

Transfiere propiedad de un expediente a otro sector.

**Request:**

```python
{
    "target_sector_id": "uuid",
    "reason": "Transferencia por competencia",
    "transfer_ownership": True,
    "assigned_user_id": None,          # Opcional
    "create_official_doc": False       # Si true, genera PV automatico
}
```

**Tipos de movimiento:**

| Tipo | `transfer_ownership` | Efecto |
|------|---------------------|--------|
| Transfer | `true` | Cambia sector propietario |
| Assign | `false` | Asigna tarea sin cambiar propietario |
| Close-assign | N/A | Cierra una asignacion activa |

### Pases PV Automaticos

Si `create_official_doc = true`, se genera un documento PV (Pase de Vista) automatico usando `transfer_document_creator.py`:

```
transfer_case(create_official_doc=true)
       |
       v
  create_transfer_document()
       |
       +-- create_document(type="PV")
       +-- Genera numero oficial
       +-- PDFComposer /move/
       +-- Firma con Notary
       +-- Sube a R2 oficial
       +-- Vincula PV al expediente
```

---

## Permisos (`permissions.py`)

### Reglas de Acceso

Un usuario puede ver un expediente si:

| Condicion | Motivo (`access_reason`) |
|-----------|------------------------|
| Su sector es el propietario actual | `ADMINSECTOR` |
| Su sector tiene una asignacion activa | `ASSIGNED` |
| Su sector fue propietario previamente | `PREVIOUS_OWNER` |
| El usuario es el creador | `CREATOR` |

```python
def get_user_viewable_sector_ids(user_id: str, *, schema_name: str) -> List[str]:
    """Obtiene sector_ids donde el usuario tiene permiso de lectura."""

def get_user_editable_sector_ids(user_id: str, *, schema_name: str) -> List[str]:
    """Obtiene sector_ids donde el usuario puede editar/transferir."""
```

---

## Historial (`history.py`)

### `create_movement()`

Registra un movimiento en `case_movements`.

```python
def create_movement(
    case_id: str,
    movement_type: str,         # creation, transfer, assign, close_assign, document_proposal
    user_id: str,
    creator_sector_id: str,
    admin_sector_id: str,
    reason: str,
    *,
    schema_name: str
) -> Dict[str, Any]:
```

**Tipos de movimiento:**

| Tipo | Descripcion |
|------|-------------|
| `creation` | Expediente creado |
| `transfer` | Transferencia de propiedad |
| `assign` | Asignacion de tarea |
| `close_assign` | Cierre de asignacion |
| `document_proposal` | Propuesta de vinculacion de documento |
