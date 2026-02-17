# Hooks

Todos los custom hooks estan en `src/hooks/`. Encapsulan logica de fetching, estado y side effects para mantener los componentes limpios.

## Autenticacion y Tenant

### useAuth

Hook unificado de autenticacion. Soporta modo Auth0 (produccion) y modo testing (desarrollo).

**Archivo**: `src/hooks/useAuth.ts`

```typescript
const {
  user_id,           // string | null - UUID del usuario en el tenant
  auth_id,           // string | null - Auth0 sub claim
  user_name,         // string | null
  user_email,        // string | null
  profile_picture_url, // string | null
  sector,            // string | null - Acronimo del sector
  department,        // string | null - Nombre del departamento
  sector_id,         // string | null - UUID del sector
  default_seal,      // string | null - Nombre del sello
  default_seal_id,   // number | null
  currentTenant,     // string | null - schema_name
  hasTenant,         // boolean
  isAuthenticated,   // boolean
  isLoading,         // boolean
  error,             // Error | null
  isTestingMode,     // boolean
  isStorageReady,    // boolean
  logout,            // () => void
  setTenant,         // (schemaName: string) => void
  setProfile,        // (profile: TenantProfile) => void
  addUserIdToUrl,    // (path: string) => string
} = useAuth();
```

**Flujo multi-tenant**:

1. Usuario hace login con Auth0
2. `useOnboarding` obtiene lista de tenants
3. Usuario selecciona tenant (o auto-select si solo tiene 1)
4. Tenant se guarda en localStorage
5. Todas las requests llevan header `X-Tenant-Schema`

---

### useOnboarding

Obtiene datos de onboarding del usuario: tenants disponibles, tenant por defecto y perfil.

**Archivo**: `src/hooks/useOnboarding.ts`

```typescript
const {
  user,            // OnboardingUser | null
  tenants,         // TenantAccess[]
  defaultTenant,   // TenantAccess | null
  defaultProfile,  // TenantProfile | null
  isLoading,       // boolean
  error,           // string | null
  refetch,         // () => Promise<void>
} = useOnboarding(skip?: boolean);
```

| Parametro | Tipo | Default | Descripcion |
|-----------|------|---------|-------------|
| `skip` | `boolean` | `false` | Si `true`, no ejecuta la llamada al montar |

---

### useTenantProfile

Obtiene el perfil del usuario en el tenant actual.

**Archivo**: `src/hooks/useTenantProfile.ts`

---

### useCsrf

Obtiene y gestiona el token CSRF para requests POST seguros.

**Archivo**: `src/hooks/useCsrf.ts`

---

## Documentos

### useCreateDocument

Crea un nuevo documento en el backend.

**Archivo**: `src/hooks/useCreateDocument.ts`

```typescript
const {
  createDocument,  // (data: CreateDocumentData) => Promise<CreateDocumentResponse | null>
  loading,         // boolean
  error,           // string | null
  clearError,      // () => void
} = useCreateDocument();
```

La funcion `createDocument` recibe:

```typescript
interface CreateDocumentData {
  document_type_acronym: string;  // "IF", "ACTA", "NOTA", etc.
  reference: string;              // Motivo/referencia del documento
}
```

---

### useSaveDocument

Guarda el contenido y metadata de un documento existente.

**Archivo**: `src/hooks/useSaveDocument.ts`

---

### useDocumentDetails

Obtiene los detalles de un documento por su ID.

**Archivo**: `src/hooks/useDocumentDetails.ts`

```typescript
const {
  documentDetails,         // DocumentDetails | null
  loading,                 // boolean
  error,                   // string | null
  refreshDocumentDetails,  // () => void
} = useDocumentDetails(documentId?: string);
```

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| `documentId` | `string \| undefined` | UUID del documento. No fetcha si es `undefined` |

---

### useUnifiedDocumentDetails

Obtiene detalles unificados (editing o signing) de un documento.

**Archivo**: `src/hooks/useUnifiedDocumentDetails.ts`

---

### useDocumentTypes

Obtiene la lista de tipos de documento disponibles.

**Archivo**: `src/hooks/useDocumentTypes.ts`

```typescript
const {
  documentTypes,  // DocumentType[]
  loading,        // boolean
  error,          // string | null
} = useDocumentTypes();
```

Incluye fallback automatico con tipos basicos si la API falla.

---

### useDocumentSearch

Busqueda de documentos con filtros.

**Archivo**: `src/hooks/useDocumentSearch.ts`

---

### useCreateImportedDocument

Crea un documento importado (PDF subido directamente).

**Archivo**: `src/hooks/useCreateImportedDocument.ts`

---

### useReplaceImportedPdf

Reemplaza el PDF de un documento importado.

**Archivo**: `src/hooks/useReplaceImportedPdf.ts`

---

## Firma

### useSignDocument

Firma un documento digitalmente.

**Archivo**: `src/hooks/useSignDocument.ts`

```typescript
const {
  signDocument,        // (documentId: string, userId: string) => Promise<SignDocumentResponse>
  isSigningDocument,   // boolean
  signError,           // string | null
} = useSignDocument();
```

Retorna `{ success: true, data: { official_number, is_numerator, document_status } }` en caso de exito.

---

### useStartSigningProcess

Inicia el proceso de firma de un documento (cambia estado de "editing" a "signing").

**Archivo**: `src/hooks/useStartSigningProcess.ts`

```typescript
const {
  startSigningProcess,  // (documentId: string, userId: string) => Promise<Response | null>
  isStarting,           // boolean
  error,                // string | null
  clearError,           // () => void
} = useStartSigningProcess();
```

---

### useRejectDocument

Rechaza un documento con motivo.

**Archivo**: `src/hooks/useRejectDocument.ts`

```typescript
const {
  rejectDocument,        // (documentId: string, userId: string, reason?: string) => Promise<Response>
  isRejectingDocument,   // boolean
  rejectError,           // string | null
} = useRejectDocument();
```

---

### useBatchSign

Firma multiples documentos en lote.

**Archivo**: `src/hooks/useBatchSign.ts`

---

## PDF

### usePdfDownload

Descarga/previsualiza un PDF de documento.

**Archivo**: `src/hooks/usePdfDownload.ts`

```typescript
const {
  pdfUrl,            // string | null - URL del blob o URL directa
  isLoadingPdf,      // boolean
  pdfError,          // string | null
  fetchPdfDownload,  // (documentId: string, userId: string) => Promise<void>
} = usePdfDownload();
```

Soporta respuestas con PDF directo (blob) o JSON con URL.

---

### usePreviewDocumentInfo

Obtiene informacion de previsualizacion del documento.

**Archivo**: `src/hooks/usePreviewDocumentInfo.ts`

---

### usePreviewInfo

Informacion de preview incluyendo firmantes y estado.

**Archivo**: `src/hooks/usePreviewInfo.ts`

---

## Expedientes

### useExpedienteData

Obtiene datos basicos de un expediente.

**Archivo**: `src/hooks/useExpedienteData.ts`

```typescript
const {
  caseData,            // CaseData | null
  loadingCase,         // boolean
  refetchCaseDetails,  // () => void
} = useExpedienteData(expedienteId, user_id, isRouterReady);
```

| Parametro | Tipo | Descripcion |
|-----------|------|-------------|
| `expedienteId` | `string \| undefined` | UUID del expediente |
| `user_id` | `string \| undefined` | UUID del usuario |
| `isRouterReady` | `boolean` | Si el router de Next.js ya tiene los query params |

---

### useExpedienteDocuments

Obtiene documentos vinculados a un expediente.

**Archivo**: `src/hooks/useExpedienteDocuments.ts`

---

### useExpedienteModals

Gestiona el estado de multiples modales del detalle de expediente.

**Archivo**: `src/hooks/useExpedienteModals.ts`

---

### useCaseHistory

Obtiene el historial completo de un expediente.

**Archivo**: `src/hooks/useCaseHistory.ts`

---

### useCaseMovements

Obtiene los movimientos recientes de un expediente.

**Archivo**: `src/hooks/useCaseMovements.ts`

---

### useCaseSearch

Busca expedientes por numero oficial.

**Archivo**: `src/hooks/useCaseSearch.ts`

---

### useAssignmentForm

Logica del formulario de asignacion de expediente.

**Archivo**: `src/hooks/useAssignmentForm.ts`

---

### useTransferForm

Logica del formulario de transferencia de expediente.

**Archivo**: `src/hooks/useTransferForm.ts`

---

### useSubsanarWorkflow

Workflow de subsanacion de documentos en expediente.

**Archivo**: `src/hooks/useSubsanarWorkflow.ts`

---

### useVincularWorkflow

Workflow para vincular documentos a expedientes.

**Archivo**: `src/hooks/useVincularWorkflow.ts`

---

## Notas

### useNotesReceived

Obtiene notas recibidas con paginacion y filtros.

**Archivo**: `src/hooks/useNotesReceived.ts`

```typescript
const {
  notes,        // NoteReceived[]
  loading,      // boolean
  error,        // string | null
  total,        // number
  totalPages,   // number
  currentPage,  // number
  refetch,      // () => void
} = useNotesReceived({
  page?: number,        // default: 1
  pageSize?: number,    // default: 10
  dateFilter?: string,
  dateFrom?: string,
  dateTo?: string,
  search?: string,
});
```

Soporta cancelacion de requests anteriores via `AbortController`.

---

### useNotesSent

Obtiene notas enviadas. Misma interfaz que `useNotesReceived`.

**Archivo**: `src/hooks/useNotesSent.ts`

---

### useNotesArchived

Obtiene notas archivadas.

**Archivo**: `src/hooks/useNotesArchived.ts`

---

### useNoteDetail

Obtiene el detalle completo de una nota.

**Archivo**: `src/hooks/useNoteDetail.ts`

---

### useArchiveNote

Archiva o desarchiva una nota.

**Archivo**: `src/hooks/useArchiveNote.ts`

---

## Busqueda

### useUnifiedSearch

Busqueda unificada de documentos y expedientes con deteccion automatica de tipo.

**Archivo**: `src/hooks/useUnifiedSearch.ts`

```typescript
const {
  query,          // string
  setQuery,       // (q: string) => void
  results,        // UnifiedSearchResponse | null
  isLoading,      // boolean
  error,          // string | null
  isOpen,         // boolean - dropdown abierto
  searchType,     // SearchType - 'document_official' | 'case_official' | 'free_text'
  partialType,    // 'document' | 'case' | null
  totalResults,   // number
  hasResults,     // boolean
  hasDocuments,   // boolean
  hasExpedientes, // boolean
  clearSearch,    // () => void
  closeDropdown,  // () => void
  openDropdown,   // () => void
} = useUnifiedSearch({
  debounceMs?: number,   // default: 300
  minLength?: number,    // default: 2
  limit?: number,        // default: 5
});
```

---

## Utilidades

### useDebounce

Debounce de valores. Util para busquedas que disparan requests.

**Archivo**: `src/hooks/useDebounce.ts`

```typescript
const debouncedValue = useDebounce<T>(value: T, delay?: number);
```

| Parametro | Tipo | Default | Descripcion |
|-----------|------|---------|-------------|
| `value` | `T` | - | Valor a debounce |
| `delay` | `number` | `300` | Milisegundos de espera |

---

### useSectors

Obtiene la lista de sectores disponibles.

**Archivo**: `src/hooks/useSectors.ts`

```typescript
const {
  sectors,   // Sector[]
  loading,   // boolean
  error,     // string | null
  refresh,   // () => void
} = useSectors();
```

---

### useUserSearch

Busqueda de usuarios por nombre o email.

**Archivo**: `src/hooks/useUserSearch.ts`

---

### useUserInfo / useUserProfile

Informacion y perfil del usuario actual.

**Archivos**: `src/hooks/useUserInfo.ts`, `src/hooks/useUserProfile.ts`

---

### useChatFastAPI

Chat con el asistente IA (GDI-AgenteLANG).

**Archivo**: `src/hooks/useChatFastAPI.ts`

```typescript
const {
  messages,     // ChatMessage[]
  isLoading,    // boolean
  error,        // string | null
  sendMessage,  // (messageText: string) => Promise<void>
} = useChatFastAPI({
  conversationId: string,   // ID de la conversacion
  userId: string,           // UUID del usuario
  caseId?: string,          // Si se define, usa endpoint contextual por expediente
});
```

Genera mensaje de bienvenida automaticamente al montar. Si `caseId` esta definido, obtiene bienvenida contextual del backend.
