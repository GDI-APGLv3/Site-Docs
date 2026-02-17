# Endpoints de Expedientes

Endpoints para gestion de expedientes (cases). Prefijo: `/api/v1/cases`.

## Listar Expedientes

### `GET /api/v1/cases/`

Lista expedientes del usuario autenticado con filtros avanzados y paginacion.

**Query Parameters:**

| Parametro | Tipo | Default | Descripcion |
|-----------|------|---------|-------------|
| `page` | int | 1 | Numero de pagina |
| `page_size` | int | 20 | Elementos por pagina (max 100) |
| `status` | string | null | `active`, `inactive`, `archived` |
| `search` | string | null | Buscar en referencia y numero |
| `date_filter` | string | null | `hoy`, `ayer`, `ultimos_7_dias`, `ultimos_30_dias` |
| `date_from` | string | null | Fecha inicio (YYYY-MM-DD) |
| `date_to` | string | null | Fecha fin (YYYY-MM-DD) |
| `sector_filter` | string | null | UUID del sector asignado |
| `trata_filter` | string | null | UUID del sector administrativo |

**Response (200):**

```json
{
    "success": true,
    "data": {
        "cases": [
            {
                "id": "uuid",
                "case_number": "EE-2025-00001-SMG-ADGEN",
                "reference": "Expediente de habilitacion comercial",
                "last_modified_at": "2025-01-15T10:30:00",
                "case_type": {
                    "name": "Expediente Administrativo",
                    "acronym": "EXP-ADM"
                },
                "access_reason": "ADMINSECTOR",
                "admin_sector": {
                    "acronym": "ADGEN",
                    "department": "Administracion General"
                },
                "assigned_sectors": []
            }
        ],
        "total": 25,
        "page": 1,
        "page_size": 20,
        "total_pages": 2
    },
    "message": "Se encontraron 25 expedientes"
}
```

**Archivo:** `endpoints/cases/list_cases.py`

---

## Crear Expediente

### `POST /api/v1/cases/`

Crea un expediente nuevo con caratula CAEX automatica.

**Request:**

```json
{
    "case_template_id": "uuid-template",
    "reference": "Habilitacion comercial - Panaderia El Sol",
    "owner_sector_id": "uuid-sector"
}
```

**Proceso:**

1. Valida template y usuario
2. Genera numero de expediente (`EE-2025-00001-SMG-ADGEN`)
3. Crea expediente en BD
4. Genera documento CAEX (caratula) automaticamente
5. Firma la caratula automaticamente
6. Vincula CAEX al expediente

**Response (200):**

```json
{
    "success": true,
    "data": {
        "case": {
            "case_id": "uuid",
            "case_number": "EE-2025-00001-SMG-ADGEN"
        },
        "cover": {
            "document_id": "uuid",
            "official_number": "CAEX-2025-00001-SMG-ADGEN"
        }
    },
    "message": "Expediente creado exitosamente: EE-2025-00001-SMG-ADGEN"
}
```

**Archivo:** `endpoints/cases/create_case.py`

---

## Plantillas de Expedientes

### `GET /api/v1/cases/templates`

Obtiene plantillas de expedientes disponibles para el usuario.

**Response (200):**

```json
{
    "success": true,
    "data": {
        "templates": [
            {
                "id": "uuid",
                "name": "Expediente Administrativo",
                "acronym": "EXP-ADM",
                "description": "Expediente de habilitacion comercial",
                "filing_department_name": "Legal y Tecnica"
            }
        ],
        "total": 5
    }
}
```

---

## Transferir Expediente

### `POST /api/v1/cases/{case_id}/transfer`

Transfiere un expediente a otro sector.

**Request:**

```json
{
    "target_sector_id": "uuid-sector-destino",
    "reason": "Transferencia por competencia",
    "transfer_ownership": true,
    "assigned_user_id": null,
    "create_official_doc": false
}
```

| Campo | Descripcion |
|-------|-------------|
| `target_sector_id` | Sector destino |
| `reason` | Motivo (5-500 caracteres) |
| `transfer_ownership` | `true`=transfiere propiedad, `false`=solo asigna tarea |
| `assigned_user_id` | Usuario especifico (opcional) |
| `create_official_doc` | Si `true`, genera PV (Pase de Vista) automatico |

**Archivo:** `endpoints/cases/transfer_case.py`

---

## Asignar Tarea

### `POST /api/v1/cases/{case_id}/assign`

Asigna tarea a otro sector sin transferir propiedad.

**Request:**

```json
{
    "target_sector_id": "uuid-sector",
    "reason": "Solicitud de informe tecnico",
    "assigned_user_id": null,
    "create_official_doc": true
}
```

---

## Cerrar Asignacion

### `POST /api/v1/cases/{case_id}/close-assign`

Cierra una asignacion activa.

**Request:**

```json
{
    "movement_id": "uuid-movimiento",
    "reason": "Tarea completada satisfactoriamente"
}
```

---

## Sectores Disponibles

### `GET /api/v1/cases/{case_id}/available-sectors`

Obtiene sectores disponibles para transferencia o asignacion. Excluye el sector propietario actual.

---

## Usuarios de un Sector

### `GET /api/v1/cases/sectors/{sector_id}/users`

Lista usuarios activos de un sector para asignacion especifica.

---

## Vincular Documento

### `POST /api/v1/cases/{case_id}/link-document`

Vincula un documento oficial firmado a un expediente.

---

## Detalle de Expediente

### `GET /api/v1/cases/{case_id}`

Obtiene detalle completo de un expediente con movimientos, documentos y permisos.

---

## Buscar por Numero

### `GET /api/v1/cases/by-number/{case_number}`

Busca un expediente por su numero exacto (ej: `EE-2025-00001-SMG-ADGEN`).
