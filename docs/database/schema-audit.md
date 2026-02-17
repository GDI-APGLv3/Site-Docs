# Schema Audit

Cada municipio tiene un schema de auditoria dedicado con nomenclatura `{schema_name}_audit` (ejemplo: `100_test_audit`). Registra automaticamente cambios en 6 tablas criticas mediante triggers.

## Arquitectura

```
{schema_name}_audit/
+-- audit_log             # Tabla unica de auditoria
+-- fn_log_change()       # Funcion que registra los cambios
+-- 6 triggers            # Uno por tabla auditada
```

## Tabla: audit_log

| Columna | Tipo | Nullable | Default | Descripcion |
|---------|------|----------|---------|-------------|
| `id` | BIGSERIAL | NO | auto | Identificador secuencial |
| `event_time` | TIMESTAMPTZ | NO | `NOW()` | Momento del evento |
| `schema_name` | TEXT | NO | - | Schema donde ocurrio el cambio |
| `table_name` | TEXT | NO | - | Tabla modificada |
| `operation` | TEXT | NO | - | Tipo: INSERT, UPDATE, DELETE |
| `user_name` | TEXT | SI | - | Usuario de PostgreSQL (`current_user`) |
| `user_id` | UUID | SI | - | ID del usuario de la app (via GUC `app.user_id`) |
| `auth_source` | VARCHAR(20) | SI | - | Origen de autenticacion |
| `old_row` | JSONB | SI | - | Fila antes del cambio (UPDATE/DELETE) |
| `new_row` | JSONB | SI | - | Fila despues del cambio (INSERT/UPDATE) |
| `changed_fields` | TEXT[] | SI | - | Lista de campos modificados (solo UPDATE) |

**Indices:**

| Indice | Columnas | Proposito |
|--------|----------|-----------|
| `idx_{schema}_audit_audit_log_event_time` | `event_time` | Busqueda por fecha |
| `idx_{schema}_audit_audit_log_table` | `table_name` | Filtrar por tabla |
| `idx_{schema}_audit_audit_log_user` | `user_id` | Buscar por usuario |

## Valores de auth_source

| Valor | Origen | Descripcion |
|-------|--------|-------------|
| `jwt` | GDI-Frontend | Usuario autenticado via Auth0 JWT |
| `api_key` | REST API | Acceso via API Key (GDI-MCP) |
| `mcp_oauth` | MCP Server | Acceso via OAuth del MCP Server |
| `testing` | Testing mode | Modo de pruebas (sin Auth0) |
| `system` | Backend | Operaciones automaticas del sistema |

## Funcion: fn_log_change

La funcion `fn_log_change()` se ejecuta como trigger AFTER en cada operacion. Captura:

- **INSERT**: Registra `new_row` con la fila insertada
- **UPDATE**: Registra `old_row`, `new_row` y calcula `changed_fields` comparando campo por campo
- **DELETE**: Registra `old_row` con la fila eliminada

### Contexto de Auditoria (GUC)

La funcion lee variables de sesion inyectadas por el Backend:

```sql
-- El Backend inyecta estos valores antes de cada operacion
SET LOCAL app.user_id = 'uuid-del-usuario';
SET LOCAL app.auth_source = 'jwt';
```

```python
# En Python (GDI-Backend), el contexto se inyecta automaticamente:
with get_db_cursor(
    commit=True,
    schema_name=schema_name,
    user_id=creator_id,      # Se inyecta como app.user_id
    auth_source="jwt"        # Se inyecta como app.auth_source
) as cursor:
    cursor.execute("INSERT INTO ...")
```

La funcion usa `current_setting('app.user_id', true)` con el segundo parametro `true` para retornar NULL en lugar de error si la variable no existe.

### Codigo SQL

```sql
CREATE OR REPLACE FUNCTION "{SCHEMA_NAME}_audit"."fn_log_change"()
RETURNS TRIGGER AS $$
DECLARE
    v_old_row JSONB := NULL;
    v_new_row JSONB := NULL;
    v_changed_fields TEXT[] := '{}';
    v_key TEXT;
    v_user_id UUID := NULL;
    v_auth_source VARCHAR(20) := NULL;
BEGIN
    -- Leer contexto de aplicacion (inyectado via GUC)
    BEGIN
        v_user_id := NULLIF(current_setting('app.user_id', true), '')::UUID;
    EXCEPTION WHEN OTHERS THEN
        v_user_id := NULL;
    END;

    BEGIN
        v_auth_source := NULLIF(current_setting('app.auth_source', true), '');
    EXCEPTION WHEN OTHERS THEN
        v_auth_source := NULL;
    END;

    IF TG_OP = 'INSERT' THEN
        v_new_row := to_jsonb(NEW);
        INSERT INTO "{SCHEMA_NAME}_audit".audit_log(
            schema_name, table_name, operation, user_name,
            user_id, auth_source, new_row
        ) VALUES (
            TG_TABLE_SCHEMA, TG_TABLE_NAME, TG_OP, current_user,
            v_user_id, v_auth_source, v_new_row
        );
        RETURN NEW;

    ELSIF TG_OP = 'UPDATE' THEN
        v_old_row := to_jsonb(OLD);
        v_new_row := to_jsonb(NEW);
        -- Detectar campos cambiados
        FOR v_key IN SELECT jsonb_object_keys(v_new_row) LOOP
            IF v_old_row->v_key IS DISTINCT FROM v_new_row->v_key THEN
                v_changed_fields := array_append(v_changed_fields, v_key);
            END IF;
        END LOOP;
        INSERT INTO "{SCHEMA_NAME}_audit".audit_log(
            schema_name, table_name, operation, user_name,
            user_id, auth_source, old_row, new_row, changed_fields
        ) VALUES (
            TG_TABLE_SCHEMA, TG_TABLE_NAME, TG_OP, current_user,
            v_user_id, v_auth_source, v_old_row, v_new_row, v_changed_fields
        );
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        v_old_row := to_jsonb(OLD);
        INSERT INTO "{SCHEMA_NAME}_audit".audit_log(
            schema_name, table_name, operation, user_name,
            user_id, auth_source, old_row
        ) VALUES (
            TG_TABLE_SCHEMA, TG_TABLE_NAME, TG_OP, current_user,
            v_user_id, v_auth_source, v_old_row
        );
        RETURN OLD;
    END IF;

    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

## Tablas Auditadas (6)

| Tabla | Grupo | Que se registra |
|-------|-------|-----------------|
| `departments` | Estructura | Creacion, renombramiento, cambio de jerarquia |
| `sectors` | Estructura | Creacion, activacion/desactivacion |
| `official_documents` | Documentos | Numeracion de documentos oficiales |
| `cases` | Expedientes | Creacion, cambio de estado, archivado |
| `case_movements` | Expedientes | Transferencias, asignaciones, subsanaciones |
| `case_official_documents` | Expedientes | Vinculacion/desvinculacion de docs a expedientes |

### Triggers

Cada trigger es de tipo `AFTER INSERT OR UPDATE OR DELETE ... FOR EACH ROW`:

```sql
-- Ejemplo: trigger en departments
CREATE TRIGGER "trg_audit_departments"
    AFTER INSERT OR UPDATE OR DELETE ON "{SCHEMA_NAME}"."departments"
    FOR EACH ROW EXECUTE FUNCTION "{SCHEMA_NAME}_audit"."fn_log_change"();
```

!!! note "Tablas NO auditadas"
    Tablas de alta frecuencia como `document_draft`, `document_signers`, `user_roles` y `notes_recipients` no tienen triggers de auditoria para evitar impacto en rendimiento. Solo se auditan las tablas donde los cambios tienen impacto legal o administrativo.

## Consultas Utiles

### Ver ultimos cambios

```sql
SELECT event_time, table_name, operation, user_id, changed_fields
FROM "100_test_audit".audit_log
ORDER BY event_time DESC
LIMIT 20;
```

### Filtrar por tabla y operacion

```sql
SELECT event_time, operation, user_id, auth_source,
       new_row->>'case_number' as case_number,
       new_row->>'status' as status
FROM "100_test_audit".audit_log
WHERE table_name = 'cases'
  AND operation = 'UPDATE'
ORDER BY event_time DESC;
```

### Ver cambios de un usuario especifico

```sql
SELECT event_time, table_name, operation, changed_fields
FROM "100_test_audit".audit_log
WHERE user_id = 'uuid-del-usuario'
ORDER BY event_time DESC;
```

### Ver transferencias de expedientes

```sql
SELECT event_time,
       new_row->>'case_id' as case_id,
       new_row->>'type' as movement_type,
       new_row->>'reason' as reason,
       auth_source
FROM "100_test_audit".audit_log
WHERE table_name = 'case_movements'
  AND operation = 'INSERT'
ORDER BY event_time DESC;
```
