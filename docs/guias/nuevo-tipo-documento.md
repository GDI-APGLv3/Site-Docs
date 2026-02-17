# Agregar un Nuevo Tipo de Documento

Guia paso a paso para agregar un nuevo tipo de documento al sistema GDI. Un tipo de documento define la categoria de un documento (Informe, Dictamen, Acta, etc.) con su acronimo, formato y reglas de visibilidad.

## Contexto

El sistema de tipos de documento tiene dos niveles:

1. **Global** (`public.global_document_types`): Catalogo maestro con 61 tipos definidos. Compartido entre todos los municipios.
2. **Per-tenant** (`{schema}.document_types`): Tipos habilitados para cada municipio, referenciando al catalogo global.

Cada municipio puede habilitar un subconjunto de los tipos globales y definir que rangos/sectores pueden usarlos.

Ver [Schema Public](../database/schema-public.md) y [Schema Municipio](../database/schema-municipio.md) para el detalle de las tablas.

## Paso 1: Agregar al Catalogo Global (SQL)

### En `02-seed-global.sql`

Agregar el nuevo tipo al INSERT de `global_document_types`:

```sql
-- En GDI-BD/sql/02-seed-global.sql
-- Agregar al final del INSERT INTO public.global_document_types
INSERT INTO public.global_document_types (id, acronym, name, type, is_active)
VALUES
    -- ... tipos existentes (1-61) ...
    (62, 'MITIPO', 'Mi Tipo de Documento', 'HTML', true);
```

| Campo | Descripcion | Valores |
|-------|-------------|---------|
| `id` | Secuencial unico | Siguiente disponible (62+) |
| `acronym` | Acronimo corto (usado en numeracion) | Ej: `INF`, `DICT`, `RESOL` |
| `name` | Nombre completo | Ej: `Informe`, `Dictamen` |
| `type` | Formato del documento | `HTML` (editor), `Importado` (PDF externo), `NOTA` (notas) |
| `is_active` | Visible en el catalogo | `true` / `false` |

!!! warning "Tipos internos"
    Los tipos PV (Pase de Vista) y CAEX (Caratula de Expediente) tienen `is_active = false` porque son generados automaticamente por el sistema al transferir o crear expedientes. No deben ser visibles para los usuarios.

### Ejecutar en BD existente

```sql
-- Agregar a BD de produccion/dev
INSERT INTO public.global_document_types (id, acronym, name, type, is_active)
VALUES (62, 'MITIPO', 'Mi Tipo de Documento', 'HTML', true);
```

## Paso 2: Habilitar en Municipio (SQL)

Cada municipio tiene su propia tabla `document_types` que referencia al catalogo global. Al habilitar un tipo, se define que rangos pueden crear documentos de ese tipo.

### Agregar a `document_types` del municipio

```sql
-- Habilitar el tipo en el municipio 100_test
INSERT INTO "100_test".document_types (
    acronym, name, type, global_document_type_id, is_active
)
VALUES (
    'MITIPO',
    'Mi Tipo de Documento',
    'HTML',
    62,    -- ID del tipo global
    true
);
```

### Configurar permisos por rango

La tabla `document_types_allowed_by_rank` define que rangos pueden crear cada tipo:

```sql
-- Obtener el ID local del tipo recien creado
-- SELECT id FROM "100_test".document_types WHERE acronym = 'MITIPO';

-- Permitir que rangos 1, 2 y 3 creen este tipo
INSERT INTO "100_test".document_types_allowed_by_rank (document_type_id, rank_id)
SELECT dt.id, r.id
FROM "100_test".document_types dt
CROSS JOIN "100_test".ranks r
WHERE dt.acronym = 'MITIPO';
```

### Configurar visibilidad por sector (opcional)

La tabla `enabled_document_types_by_sector` permite restringir tipos por sector:

```sql
-- Habilitar solo para sectores especificos
INSERT INTO "100_test".enabled_document_types_by_sector (document_type_id, sector_id)
SELECT dt.id, s.sector_id
FROM "100_test".document_types dt
CROSS JOIN "100_test".sectors s
WHERE dt.acronym = 'MITIPO'
AND s.sector_type = 'PRIV';  -- Solo sectores privados
```

!!! info "Sin restriccion por sector"
    Si no se agrega ninguna fila a `enabled_document_types_by_sector`, el tipo esta disponible para todos los sectores.

### Actualizar seeds para futuros deploys

En `GDI-BD/sql/04-seed-demo.sql`, agregar el tipo al INSERT de document_types del schema 100_test para que nuevas instalaciones lo incluyan:

```sql
-- En la seccion de document_types de 04-seed-demo.sql
INSERT INTO "100_test".document_types (acronym, name, type, global_document_type_id, is_active)
VALUES
    -- ... tipos existentes ...
    ('MITIPO', 'Mi Tipo de Documento', 'HTML', 62, true);
```

Y tambien en el template de nuevos municipios `03-create-municipio.sql` si debe estar disponible por defecto.

## Paso 3: Backend - Schemas y Validaciones

El Backend ya carga los tipos de documento de la BD dinamicamente. No necesitas cambiar codigo para tipos basicos (HTML o Importado).

### Verificar carga del tipo

El endpoint `GET /document-types` retorna todos los tipos activos del municipio:

```bash
curl http://localhost:8000/document-types \
  -H "X-User-ID: user-uuid" \
  -H "X-Tenant-Schema: 100_test"
```

### Para tipos con logica especial

Si el nuevo tipo necesita logica diferente (como NOTA que tiene destinatarios), debes modificar el service de creacion:

```python
# services/documents/lifecycle/creation.py
async def create_document(data, *, schema_name: str):
    # ... logica existente ...

    # Si el tipo tiene logica especial
    if document_type == "MITIPO":
        # Validaciones adicionales
        # Campos extras
        pass
```

Ver [Documents Lifecycle](../backend/services/documents-lifecycle.md) para entender el flujo de creacion.

## Paso 4: PDFComposer - Template Jinja2 (si necesario)

Si el nuevo tipo necesita un formato PDF diferente al estandar, debes crear un template en GDI-PDFComposer.

### Crear template HTML

```html
<!-- GDI-PDFComposer/app/templates/mitipo_template.html -->
<!DOCTYPE html>
<html>
<head>
    <style>
        /* Estilos del documento */
        body { font-family: Arial, sans-serif; margin: 40px; }
        .header { text-align: center; margin-bottom: 30px; }
        .content { line-height: 1.6; }
        .footer { margin-top: 40px; font-size: 10px; color: #666; }
    </style>
</head>
<body>
    <div class="header">
        {% if logo_url %}
        <img src="{{ logo_url }}" height="60"/>
        {% endif %}
        <h2>{{ document_reference }}</h2>
    </div>

    <div class="content">
        {{ document_content | safe }}
    </div>

    <div class="footer">
        <p>{{ entity_name }} - {{ city }}</p>
    </div>
</body>
</html>
```

### Registrar template en el servicio

```python
# GDI-PDFComposer/app/services/pdf_service.py
TEMPLATE_MAP = {
    "default": "plantilla.html",
    "CAEX": "caex_template.html",
    "PV": "pv_template.html",
    "MITIPO": "mitipo_template.html",  # Nuevo template
}
```

Ver [PDFComposer Templates](../pdfcomposer/templates.md) para referencia de templates existentes.

!!! tip "Template por defecto"
    La mayoria de los tipos de documento usan el template `default` (`plantilla.html`). Solo crea un template custom si el formato PDF difiere significativamente del estandar.

## Paso 5: Frontend - UI de Creacion

### Agregar a la UI de creacion

El Frontend carga los tipos de documento del Backend dinamicamente via `GET /document-types`. Para tipos basicos, el selector ya los muestra sin cambios.

Si necesitas un icono o color especifico:

```typescript
// GDI-FRONTEND/src/lib/document-type-config.ts
export const DOCUMENT_TYPE_CONFIG: Record<string, DocumentTypeConfig> = {
  // ... tipos existentes ...
  MITIPO: {
    icon: "FileText",       // Icono de Lucide React
    color: "#4A90D9",       // Color para badges y bordes
    description: "Mi Tipo de Documento",
  },
};
```

### Para tipos con campos adicionales

Si el tipo requiere campos extra en el formulario de creacion (como NOTA que tiene destinatarios):

```typescript
// GDI-FRONTEND/src/components/documents/create/MiTipoFields.tsx
export function MiTipoFields({ onChange }: MiTipoFieldsProps) {
  return (
    <div>
      {/* Campos especificos del tipo */}
    </div>
  );
}
```

Y condicionar su renderizado en el formulario principal:

```typescript
// En el componente de creacion
{documentType === "MITIPO" && <MiTipoFields onChange={handleChange} />}
```

## Paso 6: BackOffice - Configuracion

El BackOffice permite a administradores gestionar los tipos de documento habilitados. Si el tipo ya esta en `global_document_types`, aparecera automaticamente en la interfaz de configuracion.

Para agregar configuraciones especificas:

1. **Habilitar/deshabilitar tipo**: La tabla `document_types` del municipio con `is_active`
2. **Permisos por rango**: La tabla `document_types_allowed_by_rank`
3. **Restriccion por sector**: La tabla `enabled_document_types_by_sector`

## Checklist

- [ ] Tipo agregado a `public.global_document_types` (SQL)
- [ ] Tipo habilitado en `{schema}.document_types` del municipio
- [ ] Permisos por rango configurados en `document_types_allowed_by_rank`
- [ ] Seeds actualizados (`02-seed-global.sql`, `04-seed-demo.sql`)
- [ ] Template `03-create-municipio.sql` actualizado (si el tipo va por defecto)
- [ ] Backend: tipo aparece en `GET /document-types`
- [ ] PDFComposer: template Jinja2 creado (solo si formato especial)
- [ ] Frontend: icono/color configurado (opcional)
- [ ] Frontend: campos adicionales en formulario (si aplica)
- [ ] Probado creacion, preview y firma del nuevo tipo

## Ejemplo: Tipo NOTA

El tipo NOTA es un buen ejemplo de tipo con logica especial:

- **SQL**: Tiene `type = 'NOTA'` (enum `document_type_source`)
- **Backend**: Service dedicado en `services/notes/` con destinatarios, tracking de lectura, archivado
- **BD**: Tablas adicionales `notes_recipients` y `notes_openings`
- **Frontend**: UI especifica con selector de destinatarios y bandejas (recibidos, enviados, archivados)

Ver [Notes Service](../backend/services/notes-service.md) y [Endpoints de Notas](../backend/endpoints/notas.md).
