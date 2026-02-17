# Endpoints de Sectores

Endpoints para consultar la estructura organizacional. Prefijo: `/api/v1/sectors`.

## Listar Sectores

### `GET /api/v1/sectors/`

Lista todos los sectores con sus departamentos asociados.

**Response (200):**

```json
{
    "sectors": [
        {
            "sector_id": "uuid",
            "sector_name": "ADGEN",
            "department_id": "uuid",
            "department_name": "Administracion General",
            "department_acronym": "ADGEN",
            "department_color": "#7222C3",
            "user_count": 5,
            "is_active": true
        }
    ],
    "total": 10
}
```

**Uso principal:**

- Organigrama del municipio
- Selector de sectores para transferencias
- Filtros en listas de expedientes
- Asignacion de usuarios a sectores

**Archivo:** `endpoints/sectors/list_sectors.py`

---

## Estructura Organizacional

La estructura del sistema se organiza en:

```
Municipalidad
├── Departamento (ej: "Secretaria de Obras")
│   ├── Sector A (ej: "SECOBRA")
│   │   ├── Usuario 1
│   │   └── Usuario 2
│   └── Sector B (ej: "PLANIF")
│       └── Usuario 3
└── Departamento (ej: "Mesa de Entradas")
    └── Sector (ej: "MESA")
        ├── Usuario 4
        └── Usuario 5
```

- **Departamento**: Agrupacion administrativa (Secretaria, Direccion, etc.)
- **Sector**: Subdivision operativa dentro del departamento
- **Usuarios**: Pertenecen a un sector primario pero pueden tener permisos en otros

Cada sector tiene:

- `acronym`: Codigo corto para identificacion rapida
- `department_color`: Color para UI
- `user_count`: Cantidad de usuarios activos
