# Paginas

Todas las paginas usan Next.js Pages Router. Cada archivo en `src/pages/` se mapea directamente a una ruta URL.

## Mapa de Rutas

| Ruta | Archivo | Autenticacion | Descripcion |
|------|---------|:-------------:|-------------|
| `/` | `index.tsx` | No | Landing page, redirect si autenticado |
| `/login` | `login/` | No | Login con Auth0 |
| `/select-tenant` | `select-tenant/` | Si | Selector de municipio (multi-tenant) |
| `/dashboard` | `dashboard/` | Si | Panel principal con feed de actividad |
| `/documentos` | `documentos/` | Si | Lista de documentos del usuario |
| `/creacion-documento` | `creacion-documento/` | Si | Editor rich text de documentos |
| `/previsualizar-documento` | `previsualizar-documento/` | Si | Preview PDF del documento |
| `/documentos-firma` | `documentos-firma/` | Si | Vista de firma de documento |
| `/firma-conjunta` | `firma-conjunta/` | Si | Firma por lotes (batch) |
| `/expedientes` | `expedientes/` | Si | Lista de expedientes |
| `/expedientes/[id]` | `expedientes/[id].tsx` | Si | Detalle de expediente (tabs) |
| `/expedientes/crear-expedientes` | `expedientes/crear-expedientes/` | Si | Crear nuevo expediente |
| `/notas` | `notas/` | Si | Notas internas (recibidas/enviadas/archivadas) |
| `/asistente` | `asistente/` | Si | Chat con IA |
| `/mi-equipo` | `mi-equipo/` | Si | Gestion de equipo/sector |
| `/notificaciones` | `notificaciones/` | Si | Centro de notificaciones |
| `/design-system` | `design-system/` | No | Showcase del design system |
| `/no-access` | `no-access/` | No | Pagina de acceso denegado |
| `/loading` | `loading/` | No | Pantalla de carga |

## Detalle por Pagina

### `/` - Landing Page

**Archivo**: `src/pages/index.tsx`

- Muestra la pagina de bienvenida con boton de login
- Si el usuario ya esta autenticado, ejecuta el flujo de onboarding:
    - Obtiene lista de tenants del usuario via `useOnboarding`
    - Si tiene 1 solo tenant: auto-selecciona y redirige a `/dashboard`
    - Si tiene 2+ tenants: redirige a `/select-tenant`
- Muestra mensajes de sesion expirada (`?session_expired=true`) o logout exitoso (`?logged_out=true`)

**Hooks**: `useAuth`, `useOnboarding`

---

### `/dashboard` - Panel Principal

**Archivo**: `src/pages/dashboard/index.tsx`

- Feed de actividad con movimientos de expedientes del sector
- Barra de estadisticas (cantidad de expedientes en el sector)
- Filtros por tipo de movimiento y fecha
- Paginacion del feed

**Componentes**: `FeedList`, `FeedCard`, `FeedFilters`, `FeedTabs`, `StatsBar`

---

### `/documentos` - Lista de Documentos

**Archivo**: `src/pages/documentos/index.tsx`

- Tabla paginada de documentos del usuario actual
- Filtros por tipo de documento, estado, fecha y sector
- Click en documento navega a `/creacion-documento` (edicion) o `/documentos-firma` (firma) segun estado
- Boton para crear nuevo documento

**Componentes**: `documentosTable`, `documentosPaginator`, `FilterSlider`
**Hooks**: `useDocumentTypes`

---

### `/creacion-documento` - Editor de Documentos

**Archivo**: `src/pages/creacion-documento/index.tsx`

- Editor rich text con React Quill para crear/editar documentos
- Panel lateral con informacion del documento (tipo, referencia, firmantes)
- Seleccion de firmantes con orden secuencial
- Para documentos tipo NOTA: seleccion de destinatarios (TO/CC/BCC)
- Opciones: guardar borrador, enviar a firma, previsualizar

**Parametros URL**: `?document_id=UUID` (para editar existente), `?document_type=ACRONYM` (para crear nuevo)

**Componentes**: `Editor`, `creacionDocumentoHeader`, paneles dinamicos
**Hooks**: `useCreateDocument`, `useSaveDocument`, `useStartSigningProcess`, `useDocumentDetails`

---

### `/previsualizar-documento` - Preview PDF

**Archivo**: `src/pages/previsualizar-documento/index.tsx`

- Visualizacion del PDF generado del documento
- Informacion de firmantes y estado
- Boton para volver al editor o enviar a firma

**Parametros URL**: `?document_id=UUID`

**Componentes**: `PDFViewer`
**Hooks**: `usePreviewDocumentInfo`, `usePdfDownload`

---

### `/documentos-firma` - Firma de Documento

**Archivo**: `src/pages/documentos-firma/index.tsx`

- Vista del PDF del documento a firmar
- Progreso de firmas (completadas/total)
- Botones de firmar o rechazar
- Modal de confirmacion de firma exitosa
- Modal de rechazo con motivo

**Parametros URL**: `?document_id=UUID`

**Componentes**: `firmadorHeader`, `PDFViewer`, `modalSuccesFirma`, `modalRejectDocument`
**Hooks**: `useSignDocument`, `useRejectDocument`, `useUnifiedDocumentDetails`

---

### `/firma-conjunta` - Firma por Lotes

**Archivo**: `src/pages/firma-conjunta/index.tsx`

- Tabla de documentos pendientes de firma del usuario
- Seleccion multiple para firma en lote
- Progreso de firma batch con resultados

**Componentes**: `BatchSignTable`, `BatchSignProgress`, `BatchSignResults`
**Hooks**: `useBatchSign`

---

### `/expedientes` - Lista de Expedientes

**Archivo**: `src/pages/expedientes/index.tsx`

- Tabla paginada de expedientes accesibles por el usuario
- Filtros por trata (tipo), sector, fecha y busqueda libre
- Badges de color por sector (administrador/actuantes)
- Click navega a `/expedientes/[id]`

**Componentes**: `expedientesTable`, `expedientesFilters`, `expedientesPaginator`, `sectorFilter`, `fechaFilter`

---

### `/expedientes/[id]` - Detalle de Expediente

**Archivo**: `src/pages/expedientes/[id].tsx`

- Header con numero, trata, motivo y sectores
- Tres tabs principales:
    - **Informacion**: Documentos vinculados, documentos propuestos
    - **Acciones**: Asignar sector, transferir, vincular documento, subsanar
    - **Asistente**: Chat IA contextual al expediente
- Timeline de movimientos del expediente

**Parametros URL**: `id` (UUID del expediente, parametro de ruta dinamica)

**Componentes**: `ExpedienteHeader`, `ExpedienteTabs`, `DocumentsList`, `ActionSelectionMenu`, `AssignmentForm`, `TransferForm`, `CaseHistoryTimeline`, `MovementsPanel`, `ChatContainer`
**Hooks**: `useExpedienteData`, `useExpedienteDocuments`, `useCaseHistory`, `useCaseMovements`, `useChatFastAPI`

---

### `/expedientes/crear-expedientes` - Crear Expediente

**Archivo**: `src/pages/expedientes/crear-expedientes/index.tsx`

- Formulario para crear un nuevo expediente
- Seleccion de plantilla/trata
- Campo de referencia (motivo)
- Seleccion de sector administrador

**Componentes**: `crearExpedientes`

---

### `/notas` - Notas Internas

**Archivo**: `src/pages/notas/index.tsx`

- Tres tabs: Recibidas, Enviadas, Archivadas
- Tabla paginada con filtros por fecha y busqueda
- Indicador de lectura para notas recibidas
- Click abre detalle de la nota con PDF y metadata

**Componentes**: `NotasTableReceived`, `NotasTableSent`, `NotasFilter`, `NotasPaginator`, `NoteDetailHeader`, `NoteDetailSidebar`
**Hooks**: `useNotesReceived`, `useNotesSent`, `useNotesArchived`, `useNoteDetail`, `useArchiveNote`

---

### `/asistente` - Chat con IA

**Archivo**: `src/pages/asistente/index.tsx`

- Chat general con el asistente IA (GDI-AgenteLANG)
- Markdown rendering para respuestas
- Fuentes de documentos usadas (RAG)

**Componentes**: `ChatContainer`, `ChatInput`, `ChatMessage`, `AsistenteWelcome`
**Hooks**: `useChatFastAPI`

---

### `/design-system` - Showcase

**Archivo**: `src/pages/design-system/index.tsx`

- Pagina sin autenticacion para referencia visual del design system
- Muestra todos los componentes shadcn/ui con variantes
- Colores, badges, botones, inputs, tablas, modales, etc.
- Incluye SectorBadge con colores dinamicos

!!! info "Uso interno"
    Esta pagina es solo para desarrollo. No aparece en la navegacion de la aplicacion.
