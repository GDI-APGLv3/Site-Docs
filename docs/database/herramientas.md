# Herramientas

## Conexion a la Base de Datos

### psql (linea de comandos)

```bash
# Formato generico
psql "postgresql://USER:PASSWORD@HOST:PORT/DATABASE"

# Conectar a un schema especifico
psql "postgresql://USER:PASSWORD@HOST:PORT/DATABASE" -c "SET search_path TO '100_test', public;"
```

### pgAdmin 4

1. Descargar desde [pgadmin.org](https://www.pgadmin.org/)
2. Registrar servidor nuevo:
    - **Host**: el host de Railway (ej: `caboose.proxy.rlwy.net`)
    - **Port**: el puerto de Railway (ej: `39969`)
    - **Database**: `railway`
    - **Username**: `postgres`
    - **Password**: password de Railway
3. En el panel izquierdo, expandir: Servers > railway > Databases > railway > Schemas
4. Los schemas de municipio aparecen como `100_test`, `101_bsas`, etc.

### DBeaver

1. Nueva conexion PostgreSQL
2. Configurar host, puerto, database, usuario, password
3. En la pestana "PostgreSQL", marcar "Show all databases"

---

## Scripts Python

Todos los scripts estan en `GDI-BD/tools/` y requieren la variable de entorno `DATABASE_URL`.

### Configuracion

```bash
# Opcion 1: Variable de entorno
export DATABASE_URL='postgresql://USER:PASSWORD@HOST:PORT/DATABASE'

# Opcion 2: Archivo .env (en GDI-BD/)
cp .env.example .env
# Editar .env con los valores reales
```

### install.py - Instalacion limpia

Elimina y recrea todo el entorno de desarrollo (`100_test`).

```bash
cd GDI-BD/tools
python install.py
```

```
======================================================================
  GDI LATAM - INSTALACION LIMPIA (100_test)
======================================================================

  DATABASE_URL: postgresql://postgres:xxxxx@caboose...
  Flujo: 01-install.sql -> 02-seed-global.sql -> 04-seed-demo.sql

  ATENCION: Esto eliminara los schemas 100_test y 100_test_audit
  y recreara toda la estructura desde cero.

  Continuar? (s/N): s
```

!!! danger "Operacion destructiva"
    `install.py` ejecuta `DROP SCHEMA IF EXISTS ... CASCADE` sobre `100_test` y `100_test_audit`. Todos los datos se pierden.

### create_municipio.py - Crear municipio

Script interactivo para crear un nuevo municipio en produccion.

```bash
cd GDI-BD/tools
python create_municipio.py
```

```
======================================================================
  GDI LATAM - CREAR MUNICIPIO NUEVO
======================================================================

  --- Datos del municipio ---

  Nombre del municipio (ej: Municipalidad de La Plata): Municipalidad de La Plata
  Acronimo (4 chars max, ej: LPLA): LPLA
  Codigo pais (2 chars) [AR]: AR
  Ciudad (ej: La Plata, Buenos Aires): La Plata
  Color primario hex sin # (ej: 16158C) [16158C]: 1A5276

  --- Buckets Cloudflare R2 ---

  Bucket oficial [gdi-lpla-oficial]:
  Bucket tosign [gdi-lpla-tosign]:

======================================================================
  RESUMEN - Nuevo Municipio
======================================================================
  Nombre:          Municipalidad de La Plata
  Acronimo:        LPLA
  Pais:            AR
  Ciudad:          La Plata
  Color primario:  #1A5276
  Schema number:   101
  Schema name:     101_lpla
  Audit schema:    101_lpla_audit
  Bucket oficial:  gdi-lpla-oficial
  Bucket tosign:   gdi-lpla-tosign
======================================================================

  Confirmar creacion? (s/N): s
```

Despues de crear el municipio, el script muestra los pasos manuales pendientes:

1. Crear buckets en Cloudflare R2
2. Configurar permisos CORS
3. Crear usuarios en Auth0
4. Subir logo via BackOffice

### run_single_script.py - Ejecutar script individual

```bash
cd GDI-BD/tools
python run_single_script.py 02-seed-global.sql
python run_single_script.py migrations/023_add_primary_color_departments_sectors.sql
```

### verify_db.py - Verificar integridad

```bash
cd GDI-BD/tools
python verify_db.py
```

```
================================================================================
VERIFICACION DE SCHEMAS Y TABLAS
================================================================================

[Schema PUBLIC]
  Tablas: 9
    - api_key_users
    - api_keys
    - document_display_states
    - global_case_templates
    - global_document_types
    - global_registry_families
    - municipalities
    - roles
    - user_registry

[Schema 100_test]
  Tablas: 33
    - case_movements
    - case_official_documents
    - cases
    - ...

[Datos en SCHEMA PUBLIC]
  Roles: 3
  Global Document Types: 61
  Global Case Templates: 30
  Municipalities: 1
  Document Display States: 6

================================================================================
[OK] Base de datos verificada correctamente
================================================================================
```

---

## Queries Utiles

### Listar schemas de municipios

```sql
SELECT schema_name, name, acronym, country, is_active
FROM public.municipalities
ORDER BY schema_number;
```

### Contar tablas por schema

```sql
SELECT table_schema, COUNT(*) as table_count
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('pg_catalog', 'information_schema')
GROUP BY table_schema
ORDER BY table_schema;
```

### Ver usuarios de un municipio

```sql
SELECT u.id, u.email, u.full_name, s.acronym as sector,
       d.name as departamento, d.acronym as dept_acronym
FROM "100_test".users u
LEFT JOIN "100_test".sectors s ON u.sector_id = s.id
LEFT JOIN "100_test".departments d ON s.department_id = d.id
ORDER BY d.name, u.full_name;
```

### Ver tipos de documento activos

```sql
SELECT id, acronym, name, type, trust, is_active
FROM "100_test".document_types
WHERE is_active = true
ORDER BY id;
```

### Documentos oficiales recientes

```sql
SELECT od.official_number, od.reference,
       dt.acronym as tipo, od.signed_at,
       u.full_name as numerador
FROM "100_test".official_documents od
JOIN "100_test".document_types dt ON od.document_type_id = dt.id
JOIN "100_test".users u ON od.numerator_id = u.id
ORDER BY od.signed_at DESC
LIMIT 20;
```

### Expedientes activos con ultimo movimiento

```sql
SELECT c.case_number, c.reference, c.status,
       d.name as departamento_actual,
       cm.type as ultimo_movimiento,
       cm.created_at as fecha_movimiento
FROM "100_test".cases c
JOIN "100_test".departments d ON c.owner_department_id = d.id
LEFT JOIN LATERAL (
    SELECT type, created_at
    FROM "100_test".case_movements
    WHERE case_id = c.id
    ORDER BY created_at DESC
    LIMIT 1
) cm ON true
WHERE c.status = 'active'
ORDER BY cm.created_at DESC;
```

### Ver registros de auditoria recientes

```sql
SELECT event_time, table_name, operation,
       user_id, auth_source, changed_fields
FROM "100_test_audit".audit_log
ORDER BY event_time DESC
LIMIT 50;
```

### Verificar indices de un schema

```sql
SELECT indexname, tablename, indexdef
FROM pg_indexes
WHERE schemaname = '100_test'
ORDER BY tablename, indexname;
```

### Tamano de tablas

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) as table_size,
    pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) as index_size
FROM pg_tables
WHERE schemaname = '100_test'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;
```

### Buscar con unaccent (sin acentos)

```sql
-- Busca "tramite" y encuentra "tramite" (y viceversa)
SELECT * FROM "100_test".document_draft
WHERE unaccent(reference) ILIKE '%' || unaccent('tramite') || '%';
```

---

## Dependencias Python

Los scripts Python de `GDI-BD/tools/` requieren:

```bash
pip install psycopg2-binary python-dotenv
```

- **psycopg2-binary**: Driver PostgreSQL para Python
- **python-dotenv**: Para cargar `.env` automaticamente (usado por `run_single_script.py`)
