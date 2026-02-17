# Componentes

Todos los componentes estan en `src/components/`, organizados por modulo funcional. Se usan componentes funcionales con TypeScript strict y Tailwind CSS.

## Componentes UI Base (shadcn/ui)

Ubicacion: `src/components/ui/`

Estos son componentes de shadcn/ui (variante New York) construidos sobre Radix UI primitives:

| Componente | Archivo | Base | Props principales |
|------------|---------|------|-------------------|
| Button | `button.tsx` | CVA + Slot | `variant`, `size`, `asChild` |
| Input | `input.tsx` | - | `type`, `placeholder`, `className` |
| Badge | `badge.tsx` | CVA | `variant` |
| Card | `card.tsx` | - | `CardHeader`, `CardTitle`, `CardContent`, `CardFooter` |
| Dialog | `dialog.tsx` | Radix Dialog | `open`, `onOpenChange` |
| Select | `select.tsx` | Radix Select | `value`, `onValueChange` |
| Tabs | `tabs.tsx` | Radix Tabs | `value`, `onValueChange`, `defaultValue` |
| Table | `table.tsx` | - | `TableHeader`, `TableBody`, `TableRow`, `TableHead`, `TableCell` |
| Popover | `popover.tsx` | Radix Popover | `open`, `onOpenChange` |
| Command | `command.tsx` | cmdk | `CommandInput`, `CommandList`, `CommandItem` |
| DropdownMenu | `dropdown-menu.tsx` | Radix DropdownMenu | `DropdownMenuTrigger`, `DropdownMenuContent`, `DropdownMenuItem` |
| Tooltip | `tooltip.tsx` | Radix Tooltip | `TooltipTrigger`, `TooltipContent` |
| SectorBadge | `sector-badge.tsx` | CVA | `label`, `color`, `variant`, `tooltipText` |
| Accordion | `accordion.tsx` | Radix Accordion | `type`, `collapsible` |
| Alert | `alert.tsx` | CVA | `variant` |

### SectorBadge

Componente especializado para mostrar badges de sectores con colores dinamicos:

```tsx
// Con color del API (preferido)
<SectorBadge label="SECOBRA" color="#006CDB" tooltipText="Secretaria de Obras" />

// Sin color (fallback gris)
<SectorBadge label="SECOBRA" />

// Con variante estatica (legacy, evitar)
<SectorBadge label="ADMIN" variant="admin" />
```

La prop `color` genera automaticamente background (`color + 18` alpha), border (`color + 60` alpha) y texto del color base.

---

## Componentes por Modulo

### Documentos

Ubicacion: `src/components/documentos/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `documentosTable` | Tabla de documentos con columnas: fecha, sector, editor, tipo, numero, referencia, estado | `/documentos` |
| `documentosPaginator` | Paginacion con navegacion de paginas | `/documentos` |
| `FilterSlider` | Panel deslizable de filtros (tipo, estado, fecha, sector) | `/documentos` |

### Creacion de Documento

Ubicacion: `src/components/creacion-documento/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `creacionDocumentoHeader` | Header con titulo, tipo de documento y acciones (guardar, enviar a firma) | `/creacion-documento` |
| `paneles/` | Paneles dinamicos: firmantes, destinatarios NOTA, expedientes vinculados | `/creacion-documento` |

### Documentos Firma

Ubicacion: `src/components/documentos-firma/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `firmadorHeader` | Header del firmador con nombre del documento y progreso | `/documentos-firma` |
| `paneles/` | Paneles: progreso de firmas, info del documento, acciones | `/documentos-firma` |

### Firma Conjunta (Batch)

Ubicacion: `src/components/firma-conjunta/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `BatchSignTable` | Tabla de documentos pendientes con checkbox de seleccion | `/firma-conjunta` |
| `BatchSignProgress` | Barra de progreso durante la firma batch | `/firma-conjunta` |
| `BatchSignResults` | Resultados con exitos y errores de la firma batch | `/firma-conjunta` |

### Expedientes

Ubicacion: `src/components/expedientes/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `expedientesTable` | Tabla: fecha, trata, numero, motivo, admin/actuantes con SectorBadge | `/expedientes` |
| `expedientesFilters` | Contenedor de filtros combinados | `/expedientes` |
| `expedientesFilter` | Filtro individual generico | `/expedientes` |
| `expedientesPaginator` | Paginacion de expedientes | `/expedientes` |
| `expedientesHeader` | Header de la pagina de expedientes | `/expedientes` |
| `sectorFilter` | Combobox de filtro por sector | `/expedientes` |
| `fechaFilter` | Filtro por rango de fechas | `/expedientes` |
| `ExpedienteHeader` | Header del detalle con numero, trata, sectores | `/expedientes/[id]` |
| `ExpedienteTabs` | Tabs: Informacion, Acciones, Asistente | `/expedientes/[id]` |
| `DocumentsList` | Lista de documentos vinculados al expediente | `/expedientes/[id]` |
| `ActionSelectionMenu` | Menu de acciones: asignar, transferir, vincular, subsanar | `/expedientes/[id]` |
| `AssignmentForm` | Formulario para asignar expediente a un sector | `/expedientes/[id]` |
| `TransferForm` | Formulario para transferir expediente a otro sector | `/expedientes/[id]` |
| `CaseHistoryTimeline` | Timeline de historial del expediente | `/expedientes/[id]` |
| `MovementsPanel` | Panel de movimientos recientes | `/expedientes/[id]` |
| `crearExpedientes` | Formulario de creacion de expediente | `/expedientes/crear-expedientes` |
| `VerificationPanel` | Panel de verificacion de datos | `/expedientes/[id]` |
| `PermissionErrorModal` | Modal de error de permisos | `/expedientes/[id]` |
| `DuplicateAssignmentModal` | Modal para asignacion duplicada | `/expedientes/[id]` |

### Notas

Ubicacion: `src/components/notas/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `NotasTableReceived` | Tabla de notas recibidas con indicador de lectura | `/notas` (tab recibidas) |
| `NotasTableSent` | Tabla de notas enviadas con contador de aperturas | `/notas` (tab enviadas) |
| `NotasFilter` | Filtros de notas (fecha, busqueda) | `/notas` |
| `NotasPaginator` | Paginacion de notas | `/notas` |
| `NoteDetailHeader` | Header del detalle: remitente, destinatarios, fecha | `/notas` (detalle) |
| `NoteDetailSidebar` | Sidebar con metadatos: aperturas, estado de lectura | `/notas` (detalle) |

### Chat / Asistente

Ubicacion: `src/components/chat/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `ChatContainer` | Contenedor principal del chat con scroll y mensajes | `/asistente`, `/expedientes/[id]` |
| `ChatInput` | Input de mensaje con boton de enviar | `/asistente`, `/expedientes/[id]` |
| `ChatMessage` | Burbuja de mensaje (enviado/recibido) con markdown | `/asistente`, `/expedientes/[id]` |

### Dashboard

Ubicacion: `src/components/dashboard/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `FeedCard` | Card de un movimiento del feed con tipo, caso y documento | `/dashboard` |
| `FeedList` | Lista paginada de FeedCards | `/dashboard` |
| `FeedFilters` | Filtros del feed por tipo de movimiento | `/dashboard` |
| `FeedTabs` | Tabs para alternar vistas del feed | `/dashboard` |
| `StatsBar` | Barra de estadisticas del sector | `/dashboard` |

### Busqueda

Ubicacion: `src/components/search/`

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `UnifiedSearchInput` | Input de busqueda con deteccion de tipo y dropdown | Header global |
| `SearchResultsTable` | Tabla de resultados dividida por documentos y expedientes | Header global |

### Componentes Globales

Ubicacion: `src/components/` (raiz)

| Componente | Descripcion | Donde se usa |
|------------|-------------|--------------|
| `Editor.tsx` | Editor rich text (React Quill) | `/creacion-documento` |
| `PDFViewer.tsx` | Visualizador de PDFs | `/previsualizar-documento`, `/documentos-firma` |
| `menuLateral.tsx` | Menu lateral de navegacion principal | Layout global |
| `modalRejectDocument.tsx` | Modal para rechazar documento con motivo | `/documentos-firma` |
| `modalSuccesFirma.tsx` | Modal de confirmacion de firma exitosa | `/documentos-firma` |
| `newDocumentoPopUp.tsx` | Popup para seleccionar tipo de documento nuevo | `/documentos` |
| `DocumentResume.tsx` | Resumen de documento (AI summary) | Varios |
| `DocumentSearchInput.tsx` | Input de busqueda de documentos | Varios |
| `PersonalCardPopUp.tsx` | Popup con datos del usuario | Header |
| `PdfDropzone.tsx` | Zona de drag & drop para PDFs importados | `/creacion-documento` |
| `VincularDocumentoModal.tsx` | Modal para vincular documento a expediente | `/expedientes/[id]` |
| `Spinner.tsx` | Indicador de carga | Global |
| `AsistenteWelcome.tsx` | Mensaje de bienvenida del asistente | `/asistente` |
| `alertDocumentRejected.tsx` | Alerta de documento rechazado | `/creacion-documento` |
