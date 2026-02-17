# Design System

El design system de GDI-FRONTEND esta basado en shadcn/ui (variante New York) con colores dinamicos por tenant. La referencia visual esta disponible en `/design-system` en la aplicacion local.

## Referencia Rapida

| Recurso | Ubicacion |
|---------|-----------|
| Showcase visual | `http://localhost:3003/design-system` |
| Codigo showcase | `src/pages/design-system/index.tsx` |
| Componentes UI | `src/components/ui/` |
| Utilidad cn() | `src/lib/utils.ts` |
| Theme colores | `src/lib/theme.ts` |
| CSS variables | `src/styles/globals.css` |
| Tailwind config | `tailwind.config.js` |
| shadcn config | `components.json` |

---

## Colores

### Color Primario (Dinamico por Tenant)

El color primario cambia segun el municipio (tenant) seleccionado. Se implementa mediante CSS variables que se sobreescriben dinamicamente con JavaScript.

**Escala disponible en Tailwind**:

| Token | Uso tipico |
|-------|-----------|
| `primary-50` | Background muy sutil |
| `primary-100` | Background hover |
| `primary-200` | Borders suaves |
| `primary-300` | Elementos decorativos |
| `primary-400` | Iconos secundarios |
| `primary-500` / `primary` | Color principal (botones, links) |
| `primary-600` | Hover sobre primario |
| `primary-700` | Active/pressed |
| `primary-800` | Textos sobre fondo claro |
| `primary-900` | Textos de alto contraste |
| `primary-foreground` | Texto sobre fondo primario (blanco o negro auto) |

**Default** (sin tenant): `#16158C` (azul indigo, HSL 239/75%/31%)

**Uso en Tailwind**:

```html
<div class="bg-primary-50 border-primary-500 text-primary-900">
  <button class="bg-primary-500 text-primary-foreground hover:bg-primary-600">
    Accion
  </button>
</div>
```

**Implementacion tecnica**: Ver `src/lib/theme.ts` para la funcion `applyThemeColors()` que convierte HEX -> HSL -> escala de 10 niveles.

---

### Colores Semanticos (Fijos)

Estos colores NO cambian por tenant. Se usan para estados universales.

| Token | Valor | Uso |
|-------|-------|-----|
| `success` | `#10B981` | Operacion exitosa |
| `warning` | `#F59E0B` | Advertencias |
| `error` | `#EF4444` | Errores |
| `info` | `#3B82F6` | Informacion |

---

### Grises: neutral-*

!!! danger "Regla obligatoria"
    SIEMPRE usar `neutral-*` para grises. NUNCA usar `gray-*`, `slate-*`, `zinc-*` ni `stone-*`.

```html
<!-- CORRECTO -->
<p class="text-neutral-700">Texto secundario</p>
<div class="bg-neutral-50 border-neutral-200">Contenedor</div>

<!-- INCORRECTO - NUNCA hacer esto -->
<p class="text-gray-700">...</p>
<div class="bg-slate-50">...</div>
```

---

### Colores shadcn/ui (CSS Variables)

Variables HSL usadas por los componentes base de shadcn/ui:

| Variable | Uso |
|----------|-----|
| `--background` | Fondo de la pagina |
| `--foreground` | Texto principal |
| `--card` | Fondo de cards |
| `--popover` | Fondo de popovers |
| `--muted` | Fondos sutiles |
| `--muted-foreground` | Textos secundarios |
| `--border` | Bordes generales |
| `--input` | Bordes de inputs |
| `--ring` | Anillos de focus |
| `--destructive` | Acciones destructivas |

---

## Estados de Documento

Los estados de documentos usan colores fijos (no dinamicos):

| Estado | Color Background | Clase CSS |
|--------|-----------------|-----------|
| En edicion | `#006CDB` (azul) | `bg-[#006CDB] text-white` |
| En proceso de firma | `#6F5AD8` (violeta) | `bg-[#6F5AD8] text-white` |
| Firmar ahora | `#F9765D` (coral) | `bg-[#F9765D] text-white` |
| Firmado | `#369D6D` (verde) | `bg-[#369D6D] text-white` |

---

## Badges de Sectores

Los sectores se muestran con `SectorBadge`, un componente que acepta un color dinamico del API.

### Con color del API (preferido)

```tsx
<SectorBadge
  label="SECOBRA"
  color="#006CDB"
  tooltipText="Secretaria de Obras Publicas"
/>
```

Genera estilos inline:

- Background: `{color}18` (alpha 18 hex = ~10% opacidad)
- Border: `{color}60` (alpha 60 hex = ~38% opacidad)
- Texto: color base

### Sin color (fallback neutro)

```tsx
<SectorBadge label="SECOBRA" />
```

Usa `bg-neutral-100 border-neutral-400 text-neutral-600`.

### Variantes estaticas (legacy)

| Variante | Estilos |
|----------|---------|
| `admin` | `bg-blue-100 border-blue-400 text-blue-800` |
| `actuante` | `bg-sky-100 border-sky-400 text-sky-800` |
| `default` | `bg-neutral-100 border-neutral-400 text-neutral-600` |

!!! note "Preferir color dinamico"
    Siempre usar la prop `color` con el valor del API. Las variantes estaticas son legacy y se deben evitar.

---

## Iconografia

Todos los iconos vienen de **Lucide React** (`lucide-react`).

```tsx
import { FileText, Users, Search, ChevronDown } from 'lucide-react';

<FileText className="h-4 w-4 text-neutral-500" />
```

!!! danger "Nunca usar"
    - Inline SVGs
    - Archivos `icons.tsx` custom
    - Otras librerias de iconos

---

## Tipografia

**Fuente**: Montserrat (Google Fonts), cargada en `globals.css`.

```css
body {
  font-family: 'Montserrat', sans-serif;
}
```

Pesos usados: 400 (regular), 600 (semibold), 700 (bold).

---

## Patrones de Layout

### Pagina con Sidebar

La mayoria de paginas usan un layout con menu lateral (`menuLateral.tsx`) y area de contenido:

```
+---+------------------------------------------+
| M |                                          |
| E |          Contenido Principal             |
| N |                                          |
| U |                                          |
+---+------------------------------------------+
```

### Pagina con Panel Lateral

Paginas como creacion-documento y documentos-firma usan un split con panel lateral:

```
+---+------------------------+------------------+
| M |                        |                  |
| E |    Area Principal      |  Panel Lateral   |
| N |    (Editor/PDF)        |  (Info/Acciones) |
| U |                        |                  |
+---+------------------------+------------------+
```

### Tabla con Filtros

Paginas de listado (documentos, expedientes, notas) siguen:

```
+---+------------------------------------------+
| M |  Header + Acciones                       |
| E |  +------------------------------------+  |
| N |  | Filtros (deslizables)              |  |
| U |  +------------------------------------+  |
|   |  | Tabla de datos paginada            |  |
|   |  |                                    |  |
|   |  +------------------------------------+  |
|   |  | Paginador                          |  |
+---+------------------------------------------+
```

---

## Patron Combobox (Filtros)

Para filtros de busqueda con seleccion, se usa el patron Popover + Command (cmdk):

```tsx
import { Popover, PopoverTrigger, PopoverContent } from "@/components/ui/popover";
import { Command, CommandInput, CommandList, CommandEmpty, CommandGroup, CommandItem } from "@/components/ui/command";

<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>
    <Button variant="outline">
      {selected || "Seleccionar..."}
      <ChevronDown className="ml-2 h-4 w-4" />
    </Button>
  </PopoverTrigger>
  <PopoverContent>
    <Command>
      <CommandInput placeholder="Buscar..." />
      <CommandList>
        <CommandEmpty>Sin resultados</CommandEmpty>
        <CommandGroup>
          {items.map(item => (
            <CommandItem key={item.id} onSelect={() => handleSelect(item)}>
              {item.name}
            </CommandItem>
          ))}
        </CommandGroup>
      </CommandList>
    </Command>
  </PopoverContent>
</Popover>
```

---

## Clases Condicionales con cn()

```typescript
import { cn } from "@/lib/utils";

<Card className={cn(
  "border transition-colors",
  isSelected && "border-primary-500 bg-primary-50",
  isDisabled && "opacity-50 cursor-not-allowed"
)} />
```

`cn()` combina `clsx` (logica condicional) con `tailwind-merge` (resolucion de conflictos de clases).
