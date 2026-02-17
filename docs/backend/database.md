# Base de Datos

Capa de acceso a datos definida en `database.py`. Usa **psycopg2** con connection pool y soporte para **PgBouncer** en modo transaccional.

## Pool de Conexiones

```python
# database.py
connection_pool = SimpleConnectionPool(
    minconn=5,
    maxconn=50,
    dsn=DATABASE_URL,
    cursor_factory=RealDictCursor
)
```

El pool se inicializa al arrancar la aplicacion en el evento `startup` de FastAPI.

## Multi-Tenant: schema_name

!!! danger "Regla Critica"
    **Todas** las funciones de BD reciben `schema_name` como parametro **keyword-only** (despues de `*`). Esto es obligatorio para evitar SQL injection y tenant leakage.

```python
# CORRECTO
result = execute_query("SELECT ...", schema_name=schema_name)

# INCORRECTO - causa TypeError en runtime
result = execute_query("SELECT ...", schema_name)
```

### Como funciona

1. El `TenantMiddleware` extrae `schema_name` del header `X-Tenant-Schema`
2. El endpoint lo obtiene via `Depends(get_tenant_schema)`
3. Se pasa a funciones de BD como `schema_name=schema_name`
4. `get_db_connection()` ejecuta `SET search_path TO "{schema}", public`

## Funciones Principales

### get_db_connection

Context manager que obtiene una conexion del pool con schema configurado.

```python
@contextmanager
def get_db_connection(schema_name: str):
    """
    Obtiene conexion con SET search_path al schema.
    Reutiliza conexion en llamadas anidadas (via ContextVar).
    """
```

Caracteristicas:

- Valida `schema_name` contra SQL injection
- Reutiliza conexion activa en llamadas anidadas (ContextVar)
- Usa `SET LOCAL` en modo PgBouncer transaccional
- Reset de `search_path` antes de devolver al pool
- Retry automatico si conexion cerrada

### get_db_cursor

Context manager para obtener cursor con auto-commit opcional y contexto de auditoria.

```python
@contextmanager
def get_db_cursor(
    *,
    commit: bool = False,
    schema_name: str,
    user_id: Optional[str] = None,
    auth_source: Optional[str] = None
):
```

### execute_query

Ejecuta SELECT con retry automatico.

```python
def execute_query(
    query: str,
    params: tuple = None,
    fetch: bool = True,
    fetch_one: bool = False,
    retry_count: int = 2,
    *,
    schema_name: str
) -> Optional[list]:
```

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| `query` | `str` | Query SQL |
| `params` | `tuple` | Parametros para `%s` |
| `fetch` | `bool` | Si `True`, retorna resultados |
| `fetch_one` | `bool` | Si `True`, retorna solo 1 registro |
| `retry_count` | `int` | Intentos en caso de error de conexion |
| `schema_name` | `str` | **keyword-only** - Schema del tenant |

**Retorna:** `dict` (si `fetch_one`) o `list[dict]` (si `fetch`) o `None`.

### execute_update

Ejecuta INSERT/UPDATE/DELETE con commit automatico.

```python
def execute_update(
    query: str,
    params: tuple = None,
    *,
    schema_name: str,
    user_id: Optional[str] = None,
    auth_source: Optional[str] = None
) -> bool:
```

### execute_transaction

Context manager para transacciones atomicas con multiples operaciones.

```python
@contextmanager
def execute_transaction(
    *,
    schema_name: str,
    user_id: Optional[str] = None,
    auth_source: Optional[str] = None
):
```

**Uso:**

```python
with execute_transaction(schema_name="100_test") as (conn, cursor):
    cursor.execute("INSERT INTO ...", params1)
    cursor.execute("UPDATE ...", params2)
    # Auto-commit si no hay excepciones
    # Auto-rollback en caso de error
```

### execute_single_update

Ejecuta una operacion individual con soporte para `RETURNING`.

```python
def execute_single_update(
    query: str,
    params: tuple = None,
    returning: bool = False,
    *,
    schema_name: str,
    user_id: Optional[str] = None,
    auth_source: Optional[str] = None
):
```

**Retorna:** `{"status": "success", "rows_affected": N}` o con `"result"` si `returning=True`.

## Validacion de Schema

```python
def validate_schema_name(schema_name: str) -> str:
```

Valida que el schema sea seguro para SQL:

- No vacio ni None
- Solo letras, numeros y guion bajo (`^[a-zA-Z0-9_]+$`)
- Ejemplos validos: `"100_test"`, `"public"`, `"municipio_abc"`

## PgBouncer Transaction Mode

El backend soporta PgBouncer en modo transaccional para manejar 300-500 conexiones concurrentes:

| Modo | Comando | Comportamiento |
|------|---------|---------------|
| PostgreSQL directo | `SET search_path` | Persiste en la sesion |
| PgBouncer transaction | `SET LOCAL search_path` | Se resetea al fin de transaccion |

Configuracion via variable de entorno:

```
PGBOUNCER_TRANSACTION_MODE=true
```

## Funciones de Validacion

```python
def check_user_exists(user_id: str, *, schema_name: str) -> bool:
def check_document_exists(document_id: str, *, schema_name: str) -> bool:
def get_user_basic_info(user_id: str, *, schema_name: str) -> Optional[dict]:
def get_document_basic_info(document_id: str, *, schema_name: str) -> Optional[dict]:
```

## Numeracion de Expedientes

```python
def get_next_case_sequence(year: int = None, *, schema_name: str) -> int:
```

Usa **Advisory Lock** (`pg_advisory_xact_lock(999999)`) para serializar acceso y evitar race conditions en la numeracion de expedientes.

Formato de numero: `EE-{año}-{secuencia:06d}-{municipio}-{departamento}`

## Contexto de Auditoria

Cuando se pasan `user_id` y `auth_source` a las funciones de BD, se inyectan como GUC (Grand Unified Configuration) de PostgreSQL:

```sql
SET LOCAL app.user_id = 'uuid-del-usuario';
SET LOCAL app.auth_source = 'jwt';
```

Esto permite que los triggers de auditoria en la BD registren quien hizo cada operacion.
