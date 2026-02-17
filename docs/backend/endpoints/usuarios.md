# Endpoints de Usuarios

Endpoints para gestion de usuarios y perfiles.

## Listar Usuarios

### `GET /users`

Lista todos los usuarios activos del sistema. Ordenados alfabeticamente.

**Uso principal:** Dropdowns, selectores de firmantes, directorio.

**Response (200):**

```json
{
    "users": [
        {
            "user_id": "550e8400-e29b-41d4-a716-446655440000",
            "full_name": "Juan Perez",
            "email": "juan.perez@ejemplo.com"
        },
        {
            "user_id": "660f9511-f39c-52e5-b827-557766551111",
            "full_name": "Maria Garcia",
            "email": "maria.garcia@ejemplo.com"
        }
    ],
    "total": 2
}
```

**Archivo:** `endpoints/users/list_users.py`

---

## Perfil de Usuario

### `GET /users/{user_id}/profile`

Obtiene perfil completo de un usuario con sector, departamento y permisos.

**Response incluye:**

- Datos basicos: `full_name`, `email`, `cuit`
- Sector: `sector_id`, `sector_acronym`
- Departamento: `department_id`, `department_name`, `department_acronym`
- Sello por defecto: `default_seal_id`, `default_seal_name`
- Foto de perfil: `profile_picture_url`
- Sectores adicionales con permisos

**Archivo:** `endpoints/users/profile.py`

---

## Buscar Usuarios

### `GET /users/search`

Busca usuarios por nombre o email con paginacion.

**Query Parameters:**

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| `query` | string | Texto de busqueda (nombre o email) |
| `page` | int | Pagina (default 1) |
| `page_size` | int | Resultados por pagina |

**Archivo:** `endpoints/users/search_users.py`

---

## Documentos del Usuario

### `GET /users/{user_id}/documents`

Lista documentos del usuario con filtros avanzados y paginacion.

**Query Parameters:**

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| `page` | int | Pagina |
| `page_size` | int | Resultados por pagina |
| `status` | string | Filtro de estado |
| `document_type` | string | Filtro por tipo (acronym) |
| `search` | string | Buscar en referencia y numero |
| `date_from` | string | Fecha inicio |
| `date_to` | string | Fecha fin |

**Response incluye por documento:**

- `document_id`, `reference`, `status`, `display_status`
- `doc_type`: "draft" o "official"
- `acronym`: Tipo de documento (INF, DICT, etc.)
- `official_number`: Numero oficial (si firmado)
- `rol_usuario`: "creador", "firmante", "numerador", "otro"
- `usuario_ya_firmo`: Si el usuario ya firmo
- `todos_firmantes_comunes_firmaron`: Si todos los comunes firmaron

**Archivo:** `endpoints/users/get_documents.py`
