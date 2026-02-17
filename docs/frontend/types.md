# Types

Todas las interfaces TypeScript estan en `src/types/`, agrupadas por dominio. Son la fuente de verdad para los contratos de datos entre componentes, hooks y API routes.

## Archivos de Tipos

| Archivo | Dominio | Descripcion |
|---------|---------|-------------|
| `cases.ts` | Expedientes | CaseItem, Expediente, CaseDetail, CaseMovement |
| `notes.ts` | Notas | NoteReceived, NoteSent, NoteDetail, NoteArchived |
| `nota.ts` | Nota (documento) | Sector, NotaRecipients, helpers de recipients |
| `dashboard.ts` | Dashboard | FeedMovement, DashboardStats, MovementType |
| `chat.ts` | Chat IA | ChatMessage, ChatResponse, SuggestedAction |
| `documentTypes.ts` | Tipos Doc | DocumentType, SectorRestriction |
| `preview.ts` | Preview | PreviewDocumentInfo, Signer, SignatureDetails |
| `search.ts` | Busqueda | UnifiedDocumentResult, UnifiedExpedienteResult |
| `tenant.ts` | Multi-Tenant | TenantAccess, TenantProfile, OnboardingResponse |
| `unifiedDocument.ts` | Doc Unificado | EditingDocumentDetails, SigningDocumentDetails |
| `quill.d.ts` | Editor | Declaraciones de tipos para React Quill |

---

## Expedientes (cases.ts)

### CaseItem

Representacion de un expediente como viene de la API.

```typescript
interface CaseItem {
  id: string;
  case_number: string;         // "EE-2026-000020-TXST-INTE"
  reference: string;           // Motivo del expediente
  last_modified_at: string;    // ISO datetime
  case_type: CaseType;         // { name, acronym }
  access_reason: AccessReason; // "ADMINSECTOR" | "ASSIGNEDSECTOR" | "VIEW"
  admin_sector: Sector | null;
  assigned_sectors: Sector[];
}
```

### Expediente

Tipo transformado para la UI (datos ya mapeados).

```typescript
interface Expediente {
  case_id: string;
  ultimaModificacion: string;
  trata: string;
  motivo: string;
  numero: string;
  administradora: string;
  administradora_color?: string;
  actuantes: string;
  actuantes_colors?: string[];
  contador: string;
}
```

### CaseDetail

Detalle completo con movimientos y documentos.

```typescript
interface CaseDetail extends CaseApiResponse {
  ai_summary?: string;
  movements?: CaseMovement[];
  documents?: CaseDocument[];
}
```

### CaseMovement

Un movimiento dentro de un expediente.

```typescript
interface CaseMovement {
  id: string;
  message: string;
  created_at: string;
  created_by_user?: { id: string; full_name: string };
  sector?: SectorWithMetadata;
}
```

### Tipos Auxiliares

```typescript
type AccessReason = "ADMINSECTOR" | "ASSIGNEDSECTOR" | "VIEW";

interface CaseType {
  name: string;     // "Expediente Administrativo"
  acronym: string;  // "EXP-ADM"
}

interface Sector {
  acronym: string;
  department: string;
  sector_id?: string;
  sector_color?: string;
}

interface CaseTemplate {
  id: string;
  name: string;
  acronym: string;
  description?: string;
  filing_department_name?: string;
  filing_department_color?: string;
}
```

---

## Notas (notes.ts)

### NoteReceived

Nota recibida con estado de lectura.

```typescript
interface NoteReceived {
  document_id: string;
  official_number: string;
  reference: string;
  signed_at: string;
  ai_summary: string | null;
  document_type: string;
  recipient_type: RecipientType;  // "TO" | "CC" | "BCC"
  sender: NoteSender;
  read_status: NoteReadStatus;
}
```

### NoteSent

Nota enviada con destinatarios y aperturas.

```typescript
interface NoteSent {
  document_id: string;
  official_number: string;
  reference: string;
  signed_at: string;
  ai_summary: string | null;
  document_type: string;
  recipients: NoteRecipient[];
  openings_count: number;
}
```

### NoteDetail

Detalle completo de una nota, incluyendo contenido HTML, firmantes y aperturas.

```typescript
interface NoteDetail {
  document_id: string;
  official_number: string;
  reference: string;
  content: { html: string };
  signed_at: string | null;
  signed_pdf_url: string | null;
  ai_summary: string | null;
  signers: NoteSigner[];
  document_type: NoteDocumentType;
  department_name: string;
  recipients: NoteRecipientsDetail;
  my_access: NoteMyAccess;
  openings: NoteOpening[] | null;
  proposed_cases?: NoteProposedCase[] | null;
}
```

### NoteArchived

```typescript
interface NoteArchived extends NoteReceived {
  is_archived: boolean;
  archived_at: string | null;
}
```

### Tipos Auxiliares

```typescript
type RecipientType = 'TO' | 'CC' | 'BCC';
type NotesTab = 'received' | 'sent' | 'archived';

interface NoteSender {
  sector_id: string;
  acronym: string;
  department_name: string;
  signer_name?: string;
}

interface NoteReadStatus {
  opened: boolean;
  opened_at: string | null;
}
```

---

## Nota - Destinatarios (nota.ts)

Tipos especificos para documentos tipo NOTA con destinatarios por sector.

```typescript
interface NotaRecipients {
  to: string[];   // sector_ids - requerido al menos 1
  cc: string[];   // sector_ids - opcional
  bcc: string[];  // sector_ids - opcional
}
```

Incluye helpers:

- `hasRequiredRecipients(recipients)` - Verifica que hay al menos un TO
- `flattenRecipients(recipients)` - Convierte a array plano
- `groupRecipientsByType(recipients)` - Agrupa por tipo
- `EMPTY_RECIPIENTS` - Estado inicial vacio

---

## Dashboard (dashboard.ts)

### FeedMovement

Un movimiento individual del feed de actividad.

```typescript
interface FeedMovement {
  movement_id: string;
  movement_type: MovementType;
  message: string;
  reason: string;
  created_at: string;
  is_active: boolean;
  case: FeedCase;
  user: FeedUser;
  supporting_document: FeedDocument | null;
}

type MovementType =
  | "creation"
  | "transfer"
  | "assignment"
  | "document_link"
  | "status_change"
  | "subsanacion";
```

### DashboardStats

```typescript
interface DashboardStats {
  cases_in_sector: number;
}
```

---

## Chat (chat.ts)

```typescript
interface ChatMessage {
  id: string;
  tipo: "enviado" | "recibido";
  texto: string;
  fecha: string;
  usuario?: string;
  sources?: string[];  // Fuentes RAG
}

interface ChatResponse {
  message: string;
  conversation_id: string;
  sources?: string[];
  suggested_actions?: SuggestedAction[];
}
```

---

## Tipos de Documento (documentTypes.ts)

```typescript
interface DocumentType {
  id: string;
  name: string;       // "Informe"
  acronym: string;    // "IF"
  type?: 'HTML' | 'Importado';
  description?: string;
  restricted_sectors?: SectorRestriction[] | null;
}
```

---

## Multi-Tenant (tenant.ts)

### TenantAccess

```typescript
interface TenantAccess {
  schema_name: string;        // "100_test"
  display_name: string;       // "Municipio Test"
  is_default: boolean;
  logo_url?: string | null;
  isologo_url?: string | null;
  primary_color?: string | null;  // hex sin # (ej: "1E3A8A")
}
```

### TenantProfile

```typescript
interface TenantProfile {
  user_id: string;
  email: string;
  full_name?: string;
  sector_id?: string | null;
  sector_acronym?: string | null;
  department_id?: string | null;
  department_name?: string | null;
  department_acronym?: string | null;
  default_seal_id?: number | null;
  default_seal_name?: string | null;
  estado: number;
}
```

---

## Documento Unificado (unifiedDocument.ts)

Discriminated union que representa un documento en estado de edicion o firma.

```typescript
type UnifiedDocumentResponse = EditingDocumentDetails | SigningDocumentDetails;
```

### EditingDocumentDetails

```typescript
interface EditingDocumentDetails extends BaseDocument {
  state_category: "editing";
  selected_users: Array<{ user_id, user_name, email, seal_name, signing_order, ... }>;
  creator_info: { user_id, user_name, seal_name, department_acronym, ... };
  rejection_info?: { reason, rejected_at, rejected_by, rejected_by_name } | null;
  recipients?: { to, cc, bcc } | null;  // Solo para NOTAs
  proposed_cases?: Array<{ case_id, case_number, reference }> | null;
}
```

### SigningDocumentDetails

```typescript
interface SigningDocumentDetails extends BaseDocument {
  state_category: "signing";
  current_signer: { user_id, signing_order, already_signed, ... };
  signature_progress: { completed, total, signatures: UnifiedSigner[] };
  can_sign: boolean;
  pdf_url: string;
  official_number?: string | null;
}
```

### Type Guards

```typescript
function isEditingDetails(details: UnifiedDocumentResponse): details is EditingDocumentDetails;
function isSigningDetails(details: UnifiedDocumentResponse): details is SigningDocumentDetails;
function getDocumentRoute(stateCategory: StateCategory): string;
```

---

## Busqueda (search.ts)

```typescript
interface UnifiedSearchResponse {
  success: boolean;
  search_type: SearchType;
  query: string;
  documents: { items: UnifiedDocumentResult[]; total: number; has_more: boolean };
  expedientes: { items: UnifiedExpedienteResult[]; total: number; has_more: boolean };
}
```

Incluye constantes de mapeo para labels y colores de status:

```typescript
const DOCUMENT_STATUS_LABELS: Record<string, string>;  // "pending" -> "Borrador"
const DOCUMENT_STATUS_COLORS: Record<string, string>;  // "signed" -> "bg-green-100 text-green-700"
```
