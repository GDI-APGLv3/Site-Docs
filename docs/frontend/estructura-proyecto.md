# Estructura del Proyecto

## Arbol de Carpetas

```
GDI-FRONTEND/
├── src/
│   ├── pages/                    # Next.js Pages Router (SOLO paginas)
│   │   ├── _app.tsx              # App wrapper global
│   │   ├── index.tsx             # Landing / login redirect
│   │   ├── inicio.tsx            # Pagina de inicio post-login
│   │   ├── error.tsx             # Pagina de error global
│   │   ├── api/                  # API Routes (proxy al backend)
│   │   ├── asistente/            # Chat con IA
│   │   ├── creacion-documento/   # Editor de documentos
│   │   ├── dashboard/            # Panel principal con feed
│   │   ├── design-system/        # Showcase del design system
│   │   ├── documentos/           # Lista de documentos
│   │   ├── documentos-firma/     # Documentos en proceso de firma
│   │   ├── expedientes/          # Lista y detalle de expedientes
│   │   ├── firma-conjunta/       # Firma por lotes
│   │   ├── loading/              # Pantalla de carga
│   │   ├── login/                # Pagina de login
│   │   ├── mi-equipo/            # Gestion de equipo
│   │   ├── no-access/            # Sin permisos
│   │   ├── notas/                # Notas internas
│   │   ├── notificaciones/       # Centro de notificaciones
│   │   ├── previsualizar-documento/ # Preview PDF
│   │   └── select-tenant/        # Selector de municipio
│   │
│   ├── components/               # Componentes React por modulo
│   │   ├── ui/                   # shadcn/ui (13 componentes base)
│   │   ├── chat/                 # ChatContainer, ChatInput, ChatMessage
│   │   ├── creacion-documento/   # Header y paneles del editor
│   │   ├── dashboard/            # FeedCard, FeedList, StatsBar
│   │   ├── documentos/           # Tabla, paginador, filtros
│   │   ├── documentos-firma/     # Header y paneles del firmador
│   │   ├── expedientes/          # Tabla, filtros, tabs, formularios
│   │   ├── firma-conjunta/       # Tabla y progreso de firma batch
│   │   ├── layouts/              # Layouts compartidos
│   │   ├── notas/                # Tablas, filtros, detalle
│   │   ├── previsualizar-documento/ # Componentes de preview
│   │   ├── search/               # Busqueda unificada
│   │   └── (componentes sueltos) # Editor, PDFViewer, menuLateral, etc.
│   │
│   ├── hooks/                    # Custom hooks (40+ hooks)
│   ├── lib/                      # Utilidades y configuracion
│   │   ├── document-strategies/  # Strategy pattern para documentos
│   │   ├── expedientes/          # Utilidades de expedientes
│   │   └── panel-registry/       # Registro de paneles dinamicos
│   ├── config/                   # Configuracion centralizada (API)
│   ├── layouts/                  # Layout wrappers
│   ├── styles/                   # globals.css (Tailwind + CSS vars)
│   ├── types/                    # Interfaces TypeScript (11 archivos)
│   └── middleware.ts             # Auth middleware
│
├── public/                       # Assets estaticos
├── next.config.ts                # Configuracion Next.js
├── tailwind.config.js            # Configuracion Tailwind
├── components.json               # Configuracion shadcn/ui
├── tsconfig.json                 # TypeScript config
└── package.json                  # Dependencias
```

## Descripcion de Carpetas

### `src/pages/`

Contiene exclusivamente paginas de Next.js Pages Router. Cada archivo o carpeta se convierte en una ruta publica. Las paginas importan componentes de `src/components/` y hooks de `src/hooks/`.

!!! warning "Regla critica"
    NUNCA colocar componentes reutilizables en `src/pages/`. Next.js interpreta todo dentro de `pages/` como rutas publicas.

### `src/components/`

Componentes React organizados por modulo funcional. Los componentes base de shadcn/ui estan en `ui/`. Cada modulo (documentos, expedientes, notas) tiene su propia subcarpeta.

### `src/components/ui/`

Componentes de shadcn/ui (variante New York) basados en Radix UI:

| Componente | Primitiva Radix | Uso |
|------------|----------------|-----|
| `button.tsx` | Slot | Botones con variantes |
| `input.tsx` | - | Campos de entrada |
| `badge.tsx` | - | Etiquetas de estado |
| `card.tsx` | - | Contenedores de informacion |
| `dialog.tsx` | Dialog | Modales |
| `select.tsx` | Select | Selectores dropdown |
| `tabs.tsx` | Tabs | Navegacion por pestanas |
| `table.tsx` | - | Tablas de datos |
| `popover.tsx` | Popover | Popovers flotantes |
| `command.tsx` | cmdk | Combobox con busqueda |
| `dropdown-menu.tsx` | DropdownMenu | Menus contextuales |
| `tooltip.tsx` | Tooltip | Tooltips informativos |
| `sector-badge.tsx` | - | Badge con color dinamico por sector |

### `src/hooks/`

Custom hooks que encapsulan logica de fetching, estado y efectos. Hay 40+ hooks organizados por dominio: autenticacion, documentos, expedientes, notas, busqueda, etc.

### `src/lib/`

Utilidades compartidas: `fetchWithAuth` para manejo de sesion expirada, `tenant.ts` para multi-tenant, `theme.ts` para colores dinamicos, `csrf.ts` para proteccion CSRF, entre otros.

### `src/config/`

Configuracion centralizada de la API (`api.config.ts`). Define todos los endpoints del backend como funciones type-safe.

### `src/types/`

Interfaces TypeScript agrupadas por dominio. Son la fuente de verdad para los tipos compartidos entre componentes y hooks.

### `src/styles/`

Archivo `globals.css` con la configuracion de Tailwind, CSS variables para colores dinamicos por tenant, y variables de shadcn/ui.
