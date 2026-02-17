# Endpoints de Dashboard

Endpoints para estadisticas y feed de actividad. Prefijo: `/api/v1/dashboard`.

## Feed de Actividad

### `GET /api/v1/dashboard/feed`

Feed cronologico de actividad reciente relevante al usuario.

**Query Parameters:**

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| `page` | int | Pagina |
| `page_size` | int | Resultados por pagina |

**Tipos de eventos en el feed:**

- Documentos creados, editados, enviados a firma
- Documentos firmados y oficializados
- Documentos rechazados
- Expedientes creados
- Expedientes transferidos
- Asignaciones de tarea

**Archivo:** `endpoints/dashboard/feed.py`

---

## Estadisticas

### `GET /api/v1/dashboard/stats`

Obtiene estadisticas del usuario y su sector.

**Response incluye:**

- Total de documentos por estado (draft, sent_to_sign, signed, rejected)
- Documentos pendientes de firma del usuario
- Expedientes activos del sector
- Actividad reciente (ultimos 7 y 30 dias)

**Archivo:** `endpoints/dashboard/stats.py`

---

## Schemas

Los schemas de dashboard estan definidos en `schemas/dashboard_schemas.py`:

```python
class DashboardStatsResponse(BaseModel):
    documents_by_status: dict
    pending_signatures: int
    active_cases: int
    recent_activity: dict
```
