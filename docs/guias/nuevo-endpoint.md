# Agregar un Nuevo Endpoint

Guia paso a paso para agregar un endpoint nuevo en GDI-Backend.

## Arquitectura de Endpoints

El Backend sigue un patron de **endpoints thin**: los endpoints solo validan input y delegan la logica a services.

```
endpoints/{categoria}/router.py   # Rutas HTTP (thin controllers)
services/{dominio}/               # Logica de negocio
models/                            # Schemas Pydantic
```

Ver [Estructura del Proyecto](../backend/estructura-proyecto.md) y [Services](../backend/services/index.md) para referencia completa.

## Paso 1: Crear Schemas Pydantic

Definir los modelos de request y response en `models/`.

```python
# models/mi_feature.py
from pydantic import BaseModel, Field
from typing import Optional
from datetime import datetime


class MiFeatureRequest(BaseModel):
    """Request para crear mi feature."""
    titulo: str = Field(..., min_length=1, max_length=200, description="Titulo del recurso")
    descripcion: Optional[str] = Field(None, max_length=2000)
    sector_id: str = Field(..., description="UUID del sector")


class MiFeatureResponse(BaseModel):
    """Response de mi feature."""
    id: str
    titulo: str
    descripcion: Optional[str]
    created_at: datetime
    created_by: str
```

!!! tip "Validacion"
    Pydantic v2 valida automaticamente los datos de entrada. Usar `Field(...)` para campos requeridos y `Field(None)` para opcionales. Ver [Models y Schemas](../backend/models-schemas.md).

## Paso 2: Crear el Service

Toda la logica de negocio va en la capa de services. Nunca poner logica en los endpoints.

```python
# services/mi_feature/mi_feature_service.py
from shared.database import execute_query, execute_update
from shared.exceptions import NotFoundError, ForbiddenError


async def crear_mi_feature(
    titulo: str,
    descripcion: str | None,
    sector_id: str,
    user_id: str,
    *,
    schema_name: str,
) -> dict:
    """Crea un nuevo recurso de mi feature.

    Args:
        titulo: Titulo del recurso
        descripcion: Descripcion opcional
        sector_id: UUID del sector
        user_id: UUID del usuario creador
        schema_name: Schema del tenant (keyword-only)

    Returns:
        dict con datos del recurso creado
    """
    # Validar que el sector existe
    sector = await execute_query(
        "SELECT sector_id FROM sectors WHERE sector_id = %s AND is_active = true",
        (sector_id,),
        schema_name=schema_name,
    )
    if not sector:
        raise NotFoundError(f"Sector {sector_id} no encontrado")

    # Insertar recurso
    result = await execute_update(
        """
        INSERT INTO mi_tabla (titulo, descripcion, sector_id, created_by)
        VALUES (%s, %s, %s, %s)
        RETURNING id, titulo, descripcion, created_at, created_by
        """,
        (titulo, descripcion, sector_id, user_id),
        schema_name=schema_name,
    )

    return result


async def obtener_mi_feature(
    feature_id: str,
    *,
    schema_name: str,
) -> dict:
    """Obtiene un recurso por ID."""
    result = await execute_query(
        "SELECT * FROM mi_tabla WHERE id = %s AND is_deleted = false",
        (feature_id,),
        schema_name=schema_name,
    )
    if not result:
        raise NotFoundError(f"Recurso {feature_id} no encontrado")

    return result[0]
```

!!! danger "Regla keyword-only"
    El parametro `schema_name` **siempre** va despues de `*` en la firma de la funcion. Nunca pasarlo como argumento posicional. Ver [Multi-Tenant](../arquitectura/multi-tenant.md).

## Paso 3: Crear el Router (Endpoint)

Los endpoints solo validan, extraen datos del request y delegan al service.

```python
# endpoints/mi_feature/router.py
from fastapi import APIRouter, Request, Path, Depends, HTTPException
from models.mi_feature import MiFeatureRequest, MiFeatureResponse
from services.mi_feature.mi_feature_service import crear_mi_feature, obtener_mi_feature
from shared.exceptions import NotFoundError, ForbiddenError

router = APIRouter(
    prefix="/api/v1/mi-feature",
    tags=["Mi Feature"],
)


@router.post("/", response_model=MiFeatureResponse, status_code=201)
async def create_mi_feature_endpoint(
    request: Request,
    body: MiFeatureRequest,
):
    """Crea un nuevo recurso."""
    schema_name = request.state.schema_name
    user_id = request.state.user_id

    try:
        result = await crear_mi_feature(
            titulo=body.titulo,
            descripcion=body.descripcion,
            sector_id=body.sector_id,
            user_id=user_id,
            schema_name=schema_name,
        )
        return MiFeatureResponse(**result)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except ForbiddenError as e:
        raise HTTPException(status_code=403, detail=str(e))


@router.get("/{feature_id}", response_model=MiFeatureResponse)
async def get_mi_feature_endpoint(
    request: Request,
    feature_id: str = Path(..., description="UUID del recurso"),
):
    """Obtiene un recurso por ID."""
    schema_name = request.state.schema_name

    try:
        result = await obtener_mi_feature(
            feature_id,
            schema_name=schema_name,
        )
        return MiFeatureResponse(**result)
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

### Patron del Endpoint

Todos los endpoints siguen el mismo flujo:

1. Extraer `schema_name` de `request.state.schema_name` (inyectado por TenantMiddleware)
2. Extraer `user_id` de `request.state.user_id` (inyectado por Auth middleware)
3. Llamar al service con `schema_name=schema_name`
4. Capturar excepciones y convertirlas a HTTP responses

Ver [Middleware](../backend/middleware.md) y [Autenticacion](../backend/auth.md).

## Paso 4: Registrar el Router en main.py

El Backend carga endpoints dinamicamente por categoria. Hay dos opciones:

### Opcion A: Agregar a categoria existente

Si tu feature encaja en una categoria existente (documents, cases, users, sectors, etc.), el router se carga automaticamente al colocarlo en la carpeta correcta.

### Opcion B: Nueva categoria

Si necesitas una nueva categoria, editar `main.py`:

```python
# main.py - agregar la nueva categoria
endpoint_categories = [
    'auth', 'documents', 'users', 'system',
    'cases', 'sectors', 'dashboard', 'notes',
    'mi_feature',  # Nueva categoria
]
```

Y agregar la carga del router en la seccion de `include_endpoints`:

```python
elif category == 'mi_feature':
    router_module = importlib.import_module(f"{category_path}.router")
    if hasattr(router_module, 'router'):
        app.include_router(router_module.router)
```

## Paso 5: Agregar Autenticacion y Permisos

### Auth0 JWT (produccion)

El `TenantMiddleware` ya valida el JWT y extrae el usuario. Solo necesitas verificar permisos especificos en el service si aplica:

```python
# En el service, verificar permiso especifico
async def crear_mi_feature(..., user_id: str, *, schema_name: str):
    # Verificar que el usuario tiene permiso en el sector
    user = await get_authenticated_user(user_id, schema_name=schema_name)
    if user["sector_id"] != sector_id:
        raise ForbiddenError("No tienes permiso en este sector")
```

### Testing Mode

Con `TESTING_MODE=true`, el middleware acepta `X-User-ID` como header en lugar de JWT. No necesitas cambiar nada en el endpoint.

## Paso 6: Testear

### Con Swagger UI

```bash
# Iniciar Backend
uvicorn main:app --reload --port 8000

# Abrir en navegador
# http://localhost:8000/docs
```

### Con curl

```bash
# Crear recurso (TESTING_MODE=true)
curl -X POST http://localhost:8000/api/v1/mi-feature/ \
  -H "Content-Type: application/json" \
  -H "X-User-ID: 457c52a4-9305-4e8a-9642-0b9380a4768a" \
  -H "X-Tenant-Schema: 100_test" \
  -d '{
    "titulo": "Mi primer recurso",
    "descripcion": "Prueba de endpoint",
    "sector_id": "uuid-del-sector"
  }'

# Obtener recurso
curl http://localhost:8000/api/v1/mi-feature/uuid-del-recurso \
  -H "X-User-ID: 457c52a4-9305-4e8a-9642-0b9380a4768a" \
  -H "X-Tenant-Schema: 100_test"
```

### Health check

```bash
curl http://localhost:8000/health
```

## Checklist

- [ ] Schemas Pydantic creados en `models/`
- [ ] Service creado en `services/` con logica de negocio
- [ ] `schema_name` es keyword-only en todas las funciones de BD
- [ ] Router creado en `endpoints/{categoria}/`
- [ ] Router registrado en `main.py` (si nueva categoria)
- [ ] Endpoint extrae `schema_name` de `request.state`
- [ ] Excepciones convertidas a HTTP responses
- [ ] Testeado con curl o Swagger UI
- [ ] Funciona en `TESTING_MODE=true`

## Ejemplo Completo: Endpoint Existente

Para referencia, ver como esta implementado el modulo de notas:

- **Endpoints**: `endpoints/notes/router.py`
- **Service**: `services/notes/`
- **Registrado en**: `main.py` linea 88 (`'notes'` en `endpoint_categories`)

Ver [Endpoints de Notas](../backend/endpoints/notas.md) y [Notes Service](../backend/services/notes-service.md).
