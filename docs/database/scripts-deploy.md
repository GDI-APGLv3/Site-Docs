# Scripts de Deploy

El repositorio **GDI-BD** contiene 4 archivos SQL y 4 herramientas Python para administrar la base de datos.

## Estructura del Repositorio

```
GDI-BD/
+-- sql/                          # Scripts SQL (4 archivos)
|   +-- 01-install.sql            # Extensiones + ENUMs + 9 tablas public
|   +-- 02-seed-global.sql        # Datos globales: roles, doc types, case templates
|   +-- 03-create-municipio.sql   # Template para crear municipio (con placeholders)
|   +-- 04-seed-demo.sql          # Crear 200_muni + datos demo (autonomo)
|
+-- tools/                        # Scripts Python
|   +-- create_municipio.py       # Crear municipio nuevo (interactivo)
|   +-- install.py                # Instalacion limpia de 200_muni
|   +-- run_single_script.py      # Ejecutar script SQL individual
|   +-- verify_db.py              # Verificar integridad de BD
|
+-- .env.example                  # Template de credenciales
+-- .gitignore
+-- README.md
```

## Flujos de Deploy

=== "Produccion (municipio nuevo)"

    Se ejecutan 3 scripts en orden. Los dos primeros solo se ejecutan una vez al crear la BD.

    ```
    01-install.sql           -- Extensiones + ENUMs + 9 tablas public
         |
         v
    02-seed-global.sql       -- 3 roles + 61 doc types + 30 case templates + 6 display states + 3 registry families
         |
         v
    03-create-municipio.sql  -- Schema del municipio + audit + datos iniciales
    ```

    El script `03` requiere reemplazo de 9 placeholders antes de ejecutar.

=== "Dev/Test (200_muni)"

    Se ejecutan 3 scripts. El tercero (`04-seed-demo.sql`) es autonomo y contiene toda la estructura y datos hardcoded.

    ```
    01-install.sql           -- Extensiones + ENUMs + 9 tablas public
         |
         v
    02-seed-global.sql       -- Datos globales
         |
         v
    04-seed-demo.sql         -- 200_muni completo (schemas + datos demo)
    ```

---

## Scripts SQL

### 01-install.sql

Crea la infraestructura base en el schema `public`. Solo se ejecuta una vez por BD.

**Contenido:**

- 3 extensiones: `vector`, `unaccent`, `pg_trgm`
- 7 tipos enumerados (ver [ENUMs y Tipos](enums-y-tipos.md))
- 9 tablas globales (ver [Schema Public](schema-public.md))

```sql
-- Extensiones
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- ENUMs
CREATE TYPE "public"."country_enum" AS ENUM ('AR', 'BR', 'UY', ...);
CREATE TYPE "public"."document_status" AS ENUM ('draft', 'sent_to_sign', ...);
-- ... (7 ENUMs en total)

-- Tablas
CREATE TABLE "public"."roles" (...);
CREATE TABLE "public"."global_document_types" (...);
-- ... (9 tablas en total)
```

### 02-seed-global.sql

Inserta datos iniciales en las tablas globales.

| Tabla | Cantidad | Descripcion |
|-------|----------|-------------|
| `roles` | 3 | Usuario General, Funcionario, Administrador |
| `global_document_types` | 61 | 58 publicos + PV + CAEX (internos) |
| `global_case_templates` | 30 | EEVAR, LICPUB, HABI, COMP, DEM, RRHH, etc. |
| `document_display_states` | 6 | DRAFT, PENDING_SIGN, SIGNED, REJECTED, CANCELLED, NUMBERED |
| `global_registry_families` | 3 | ARQ, LUM, ORD |

### 03-create-municipio.sql

Template para crear un municipio nuevo. Contiene **9 placeholders** que deben reemplazarse antes de ejecutar.

**Placeholders:**

| Placeholder | Ejemplo | Descripcion |
|-------------|---------|-------------|
| `{SCHEMA_NAME}` | `201_otra` | Nombre del schema |
| `{MUNICIPALITY_NAME}` | `Municipalidad de Buenos Aires` | Nombre completo |
| `{ACRONYM}` | `BSAS` | Acronimo (2-4 letras) |
| `{COUNTRY}` | `AR` | Codigo de pais |
| `{SCHEMA_NUMBER}` | `101` | Numero auto-incremental |
| `{BUCKET_OFICIAL}` | `gdi-bsas-oficial` | Bucket R2 para docs oficiales |
| `{BUCKET_TOSIGN}` | `gdi-bsas-tosign` | Bucket R2 para docs a firmar |
| `{CITY}` | `Buenos Aires` | Ciudad (aparece en sellos) |
| `{PRIMARY_COLOR}` | `16158C` | Color hex sin # |

**Secciones del script:**

1. **Schema municipio** - 33 tablas (Grupos A-I) + ~47 indices + trigger sync
2. **Schema audit** - `audit_log` + `fn_log_change` + 6 triggers
3. **Datos iniciales** - settings + estado_users
4. **Registro del tenant** - INSERT en `public.municipalities` + departamento ROOT + 2 case_templates base (EEVAR, ECAPA)

!!! tip "Usar el script Python"
    En lugar de reemplazar placeholders manualmente, usar `python tools/create_municipio.py` que lo hace de forma interactiva.

### 04-seed-demo.sql

Script autonomo que crea el tenant `200_muni` completo. No necesita `03-create-municipio.sql` porque contiene toda la estructura embebida.

**Secciones:**

| Seccion | Contenido |
|---------|-----------|
| 1 | Schema `200_muni`: 33 tablas + indices + trigger sync |
| 2 | Schema `200_muni_audit`: audit_log + fn_log_change + 6 triggers |
| 3 | Datos iniciales: settings + estado_users + municipio en public |
| 4 | Datos demo: departamentos, sectores, ranks, usuarios, doc types, case templates, registry families |
| 5 | Drafts de bienvenida: 5 documentos borrador tipo IF |

**Datos demo incluidos:**

| Dato | Cantidad | Descripcion |
|------|----------|-------------|
| Departamentos | 10 | INTE, LEGAL, INNO, SAL, HAC, TESO, CONT, SEG, OOPU, OOPA |
| Sectores | 14 | 10 PRIV + 4 MESA/ADMIN |
| Ranks | 3 | Intendente (1), Secretario (2), Director (3) |
| City Seals | 4 | Innovador (generico), Intendente, Secretario, Director |
| Usuarios | 15 | Ficticios @munitest.com |
| User Seals | 15 | Uno por usuario |
| Document Types | 20 | IF, NOTA, PROV, ACT, RESOL, DICTA, PERMI, etc. |
| Case Templates | 6 | TEST, HABI, HIND, COMP, DEM, RRHH |
| Registry Families | 3 | ARQ, LUM, ORD (con permisos) |
| Drafts | 5 | Bienvenida a GDI (IF) |

---

## Herramientas Python

Todos los scripts Python usan la variable de entorno `DATABASE_URL` para conectarse:

```bash
# Configurar conexion
export DATABASE_URL='postgresql://USER:PASSWORD@HOST:PORT/DATABASE'
```

O copiar `.env.example` a `.env` y editar:

```bash
# .env
DATABASE_URL=postgresql://USER:PASSWORD@HOST:PORT/DATABASE
```

### install.py

Instalacion limpia del tenant `200_muni`. Elimina y recrea todo.

```bash
cd GDI-BD/tools
python install.py
```

**Flujo:**

1. Pide confirmacion (elimina datos existentes)
2. DROP schemas `200_muni` y `200_muni_audit`
3. Ejecuta `01-install.sql`
4. Ejecuta `02-seed-global.sql`
5. Ejecuta `04-seed-demo.sql`
6. Ejecuta `verify_db.py` para verificar

### create_municipio.py

Crea un municipio nuevo de forma interactiva.

```bash
cd GDI-BD/tools
python create_municipio.py
```

**Flujo:**

1. Pide datos: nombre, acronimo, pais, ciudad, color, buckets
2. Auto-calcula `schema_number` (MAX + 1 de municipalities)
3. Auto-genera `schema_name` como `{number}_{acronym_lower}`
4. Muestra resumen y pide confirmacion
5. Reemplaza los 9 placeholders en `03-create-municipio.sql`
6. Ejecuta el SQL
7. Verifica la creacion (tablas, settings, departamento ROOT)
8. Muestra pasos manuales pendientes (buckets R2, Auth0, etc.)

### run_single_script.py

Ejecuta un archivo SQL individual. Util para migraciones o scripts ad-hoc.

```bash
cd GDI-BD/tools
python run_single_script.py 02-seed-global.sql
```

### verify_db.py

Verifica la integridad de la BD. Lista schemas, tablas y conteos.

```bash
cd GDI-BD/tools
python verify_db.py
```

**Verifica:**

- Schema `public`: tablas y datos (roles, document types, case templates, municipalities, display states)
- Schema `200_muni`: tablas y datos (settings, estado_users, city_seals, document_types, case_templates)
- Schema `200_muni_audit`: tabla audit_log y conteo

---

## Migraciones

Las migraciones historicas se encuentran en `sql/migrations/`. Cada migracion tiene un numero secuencial y una descripcion.

!!! note "Migraciones integradas"
    Las migraciones antiguas (001-013) fueron integradas en los templates base y eliminadas. Solo se conservan las migraciones recientes (014+) para actualizar BDs existentes.

Ejemplos de migraciones activas:

| Archivo | Descripcion |
|---------|-------------|
| `014_add_notes_tables.sql` | Tablas de notas (Grupo H) |
| `015_add_auth_source_to_audit.sql` | Trazabilidad en auditoria |
| `016_add_api_key_users.sql` | Usuarios autorizados por API Key |
| `022_add_assignment_close_movement_type.sql` | Nuevo tipo de movimiento |
| `023_add_primary_color_departments_sectors.sql` | Color en departamentos/sectores |

Para ejecutar una migracion:

```bash
cd GDI-BD/tools
python run_single_script.py migrations/023_add_primary_color_departments_sectors.sql
```
