# Onboarding de un Nuevo Municipio

Guia paso a paso para incorporar un nuevo municipio al sistema GDI Latam.

## Concepto Multi-Tenant

GDI usa una arquitectura **schema-per-tenant** en PostgreSQL. Cada municipio tiene su propio schema de base de datos con 33 tablas, completamente aislado de los demas municipios. Las tablas globales (tipos de documento, roles, etc.) se comparten en el schema `public`.

Ver [Arquitectura Multi-Tenant](../arquitectura/multi-tenant.md) para el detalle tecnico.

## Nomenclatura

| Elemento | Formato | Ejemplo |
|----------|---------|---------|
| Schema BD | `{numero}_{nombre}` | `200_salta` |
| Schema audit | `{numero}_{nombre}_audit` | `200_salta_audit` |
| Bucket oficial | `tenant-{nombre}-oficial` | `tenant-salta-oficial` |
| Bucket tosign | `tenant-{nombre}-tosign` | `tenant-salta-tosign` |
| Acronimo | 3-6 letras | `SALTA`, `BSAS` |

## Paso 1: Crear Schema en Base de Datos

### Opcion A: Script interactivo (recomendado)

```bash
cd GDI-BD
export DATABASE_URL='postgresql://user:pass@host:port/db'
python tools/create_municipio.py
```

El script pregunta interactivamente:

1. Nombre del municipio
2. Acronimo
3. Pais
4. Numero de schema
5. Ciudad
6. Color primario
7. Nombres de buckets R2

Y ejecuta `03-create-municipio.sql` reemplazando los 9 placeholders.

### Opcion B: Ejecucion manual

El archivo `03-create-municipio.sql` tiene 9 placeholders que deben reemplazarse:

| Placeholder | Descripcion | Ejemplo |
|-------------|-------------|---------|
| `{SCHEMA_NAME}` | Nombre completo del schema | `200_salta` |
| `{MUNICIPALITY_NAME}` | Nombre del municipio | `Municipalidad de Salta` |
| `{ACRONYM}` | Acronimo | `SALTA` |
| `{COUNTRY}` | Pais | `Argentina` |
| `{SCHEMA_NUMBER}` | Numero del schema | `200` |
| `{BUCKET_OFICIAL}` | Bucket R2 para PDFs firmados | `tenant-salta-oficial` |
| `{BUCKET_TOSIGN}` | Bucket R2 para PDFs pendientes | `tenant-salta-tosign` |
| `{CITY}` | Ciudad (para sellos de firma) | `Salta` |
| `{PRIMARY_COLOR}` | Color primario (hex sin #) | `16158C` |

```bash
# Reemplazar placeholders y ejecutar
sed -e 's/{SCHEMA_NAME}/200_salta/g' \
    -e 's/{MUNICIPALITY_NAME}/Municipalidad de Salta/g' \
    -e 's/{ACRONYM}/SALTA/g' \
    -e 's/{COUNTRY}/Argentina/g' \
    -e 's/{SCHEMA_NUMBER}/200/g' \
    -e 's/{BUCKET_OFICIAL}/tenant-salta-oficial/g' \
    -e 's/{BUCKET_TOSIGN}/tenant-salta-tosign/g' \
    -e 's/{CITY}/Salta/g' \
    -e 's/{PRIMARY_COLOR}/16158C/g' \
    sql/03-create-municipio.sql | psql $DATABASE_URL
```

### Que crea el script

El script `03-create-municipio.sql` ejecuta todo en secuencia:

1. **Schema principal** (`200_salta`): 33 tablas con indices y trigger de sync
2. **Schema de auditoria** (`200_salta_audit`): tabla `audit_log` + funcion + 6 triggers
3. **Registro en public**: INSERT en `public.municipalities`
4. **Datos iniciales**: departamento ROOT, settings del municipio, estados de usuario

```
03-create-municipio.sql
├── CREATE SCHEMA "200_salta"
├── 33 tablas (Grupos A-I)
├── ~47 indices
├── 1 trigger (sync user_registry)
├── CREATE SCHEMA "200_salta_audit"
├── audit_log + fn_log_change + 6 triggers
├── INSERT INTO public.municipalities
├── INSERT departamento ROOT
└── INSERT settings (logo, colores, buckets, city)
```

Ver [Scripts de Deploy](../database/scripts-deploy.md) para el detalle completo del flujo SQL.

## Paso 2: Crear Buckets en Cloudflare R2

Cada municipio necesita dos buckets en Cloudflare R2:

```
tenant-{nombre}-oficial   # PDFs firmados (documentos oficiales)
tenant-{nombre}-tosign    # PDFs pendientes de firma
```

### Crear via Cloudflare Dashboard

1. Ir a Cloudflare Dashboard > R2 > Create Bucket
2. Crear `tenant-salta-oficial`
3. Crear `tenant-salta-tosign`
4. Asegurar que las API Keys existentes tienen acceso a los nuevos buckets

### Verificar configuracion en BD

Los nombres de los buckets se almacenan en la tabla `settings` del schema:

```sql
SELECT bucket_oficial, bucket_tosign FROM "200_salta".settings;
-- tenant-salta-oficial, tenant-salta-tosign
```

Ver [Cloudflare R2](../deploy/cloudflare-r2.md) para mas detalles.

## Paso 3: Configurar Usuarios en Auth0

### Crear usuarios iniciales

Cada municipio necesita al menos un usuario administrador en Auth0:

1. Ir a Auth0 Dashboard > User Management > Users
2. Crear usuario con email y password
3. Asegurar que el usuario tiene la conexion de base de datos correcta

### Mapeo usuario-municipio

El sistema usa `public.user_registry` para mapear emails a schemas:

```sql
-- Este INSERT se hace automaticamente cuando el usuario hace login por primera vez
-- y pasa por el endpoint de onboarding
INSERT INTO public.user_registry (email, schema_name)
VALUES ('admin@salta.gob.ar', '200_salta');
```

!!! info "Onboarding automatico"
    El endpoint de onboarding del Backend se encarga de registrar al usuario en `user_registry` y crear su registro en la tabla `users` del schema correspondiente. Solo necesitas crear el usuario en Auth0.

## Paso 4: Configurar Estructura Organizacional

### Via BackOffice (recomendado)

El BackOffice (`GDI-BackOffice-Front`) permite configurar la estructura organizacional graficamente:

1. **Departamentos**: Crear la estructura jerarquica (Intendencia > Secretarias > Direcciones)
2. **Sectores**: Crear sectores dentro de cada departamento (PRIV, MESA, ADMIN)
3. **Rangos**: Definir rangos del municipio (Intendente, Secretario, Director)
4. **Sellos**: Crear sellos de firma (asociados a rangos)
5. **Usuarios**: Asignar usuarios a sectores con sus roles y sellos

### Via SQL directo

Para municipios con estructura predefinida, ejecutar SQL:

```sql
-- Crear departamentos
INSERT INTO "200_salta".departments (
    name, acronym, parent_department_id, rank_id, head_user_id
) VALUES
    ('Secretaria de Gobierno', 'GOB', NULL, 2, NULL),
    ('Direccion Legal', 'LEGAL', uuid_del_gob, 3, NULL);

-- Crear sectores
INSERT INTO "200_salta".sectors (
    name, department_id, sector_type
) VALUES
    ('Despacho GOB', uuid_dept_gob, 'PRIV'),
    ('Mesa GOB', uuid_dept_gob, 'MESA');

-- Crear rangos locales
INSERT INTO "200_salta".ranks (name, level, head_signature)
VALUES
    ('Intendente', 1, 'Intendente Municipal'),
    ('Secretario', 2, 'Secretario/a'),
    ('Director', 3, 'Director/a');

-- Crear sellos
INSERT INTO "200_salta".city_seals (
    name, entity_name, entity_description, rank_id
) VALUES
    ('Sello Intendente', 'Municipalidad de Salta', 'Departamento Ejecutivo', 1),
    ('Sello Secretario', 'Municipalidad de Salta', 'Secretaria de Gobierno', 2);

-- Crear usuarios
INSERT INTO "200_salta".users (
    user_id, email, first_name, last_name, sector_id
) VALUES
    (gen_random_uuid(), 'admin@salta.gob.ar', 'Admin', 'Salta', uuid_del_sector);

-- Asignar sellos a usuarios
INSERT INTO "200_salta".user_seals (user_id, seal_id)
VALUES (uuid_del_usuario, uuid_del_sello);
```

### Habilitar tipos de documento

```sql
-- Habilitar tipos basicos para el municipio
INSERT INTO "200_salta".document_types (
    acronym, name, type, global_document_type_id, is_active
)
SELECT acronym, name, type, id, true
FROM public.global_document_types
WHERE is_active = true
AND acronym IN ('IF', 'DICTA', 'RESOL', 'DISP', 'AINSP');

-- Configurar permisos por rango para cada tipo
INSERT INTO "200_salta".document_types_allowed_by_rank (document_type_id, rank_id)
SELECT dt.id, r.id
FROM "200_salta".document_types dt
CROSS JOIN "200_salta".ranks r;
```

## Paso 5: Verificar Acceso

### Health check del Backend

```bash
curl http://localhost:8000/health
# {"status": "healthy", "database": "connected"}
```

### Verificar schema

```sql
-- Verificar que el schema existe y tiene tablas
SELECT COUNT(*) FROM information_schema.tables
WHERE table_schema = '200_salta';
-- Debe retornar 33

-- Verificar municipio registrado
SELECT * FROM public.municipalities WHERE schema_name = '200_salta';

-- Verificar settings
SELECT city, primary_color, bucket_oficial, bucket_tosign
FROM "200_salta".settings;
```

### Test de login

```bash
# Con TESTING_MODE=true
curl http://localhost:8000/users \
  -H "X-User-ID: uuid-del-admin" \
  -H "X-Tenant-Schema: 200_salta"
```

### Test de creacion de documento

```bash
curl -X POST http://localhost:8000/documents \
  -H "Content-Type: application/json" \
  -H "X-User-ID: uuid-del-admin" \
  -H "X-Tenant-Schema: 200_salta" \
  -d '{
    "document_type_id": "uuid-del-tipo",
    "title": "Documento de prueba"
  }'
```

## Checklist

- [ ] Schema creado en BD (`{numero}_{nombre}`)
- [ ] Schema de auditoria creado (`{numero}_{nombre}_audit`)
- [ ] Municipio registrado en `public.municipalities`
- [ ] Buckets R2 creados (oficial + tosign)
- [ ] Settings configurados (city, color, buckets)
- [ ] Al menos un usuario creado en Auth0
- [ ] Departamentos y sectores configurados
- [ ] Rangos definidos
- [ ] Sellos de firma creados
- [ ] Usuarios asignados a sectores con sellos
- [ ] Tipos de documento habilitados con permisos por rango
- [ ] Login funcional verificado
- [ ] Creacion de documento de prueba exitosa

## Flujo Rapido (Resumen)

```
1. SQL: 03-create-municipio.sql (schema + audit + datos iniciales)
2. R2: Crear 2 buckets en Cloudflare
3. Auth0: Crear usuario admin
4. BackOffice: Configurar organigramas, rangos, sellos, usuarios
5. Verificar: Login + crear documento de prueba
```

Ver tambien [Railway](../deploy/railway.md) para configurar el servicio en produccion.
