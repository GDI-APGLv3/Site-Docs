# API Routes

Todas las API routes estan en `src/pages/api/` y actuan como proxy server-side hacia GDI-Backend (:8000). El cliente nunca contacta al backend directamente.

## Patron General

Cada API route sigue este patron:

```typescript
import { NextApiRequest, NextApiResponse } from "next";
import { API_CONFIG } from "@/config/api.config";
import { getServerToken, getServerHeaders } from "@/lib/getServerToken";
import { handleApiError } from "@/lib/apiErrorHandler";

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  try {
    // 1. Obtener token de autenticacion (Auth0 o testing)
    const token = await getServerToken(req);

    // 2. Construir URL del backend
    const backendUrl = API_CONFIG.url(API_CONFIG.endpoints.documents.list());

    // 3. Proxy al backend con headers de auth
    const response = await fetch(backendUrl, {
      headers: getServerHeaders(token, req),
    });

    // 4. Retornar respuesta al cliente
    const data = await response.json();
    res.status(200).json(data);
  } catch (error) {
    // 5. Manejo estandarizado de errores
    handleApiError(error, res, 'endpoint-name');
  }
}
```

---

## Inventario de API Routes

### Documentos

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/documents` | GET | `GET /users/documents` | Lista documentos del usuario actual |
| `/api/create-document` | POST | `POST /documents` | Crear nuevo documento |
| `/api/documents/[documentId]/save` | POST | `POST /documents/{id}/save` | Guardar contenido del documento |
| `/api/documents/[documentId]/details` | GET | `GET /documents/{id}/details` | Detalles unificados (editing/signing) |
| `/api/documents/[documentId]/start-signing-process` | POST | `POST /documents/{id}/start-signing-process` | Iniciar proceso de firma |
| `/api/documents/[documentId]/super-sign` | POST | `POST /documents/{id}/sign` | Firmar documento |
| `/api/documents/[documentId]/reject` | POST | `POST /documents/{id}/reject` | Rechazar documento con motivo |
| `/api/documents/[documentId]/delete` | DELETE | `DELETE /documents/{id}` | Eliminar documento (soft delete) |
| `/api/documents/[documentId]/preview-info` | GET | `GET /documents/{id}/preview-info` | Info de previsualizacion |
| `/api/documents/[documentId]/preview-download` | POST | `POST /documents/{id}/preview-download` | Descargar PDF de preview |
| `/api/documents/[documentId]/signature-details` | GET | `GET /documents/{id}/signature-details` | Detalles de firma y progreso |
| `/api/documents/[documentId]/replace-pdf` | POST | `POST /documents/{id}/replace-pdf` | Reemplazar PDF importado |
| `/api/documents/check-signer-permissions` | GET | Verificacion de permisos | Verificar si usuario puede firmar |
| `/api/document-details/[documentId]` | GET | `GET /documents/{id}/editor-details` | Detalles para el editor (legacy) |
| `/api/document-types` | GET | `GET /document-types` | Lista tipos de documento |
| `/api/documents-autocomplete` | GET | `GET /api/v1/documents/autocomplete` | Autocompletado de documentos |
| `/api/documents-getpdf` | GET | `GET /documents/{id}/geturl_officialdoc` | URL del PDF oficial |
| `/api/documents-search-official` | GET | `GET /api/v1/documents/search-official/{num}` | Buscar por numero oficial |
| `/api/import-document` | POST | `POST /documents` | Importar documento PDF |
| `/api/subsanar-documento` | POST | `POST /api/v1/cases/{id}/subsanar` | Subsanar documento en expediente |
| `/api/vincular-documento` | POST | `POST /api/v1/cases/{id}/documents/link` | Vincular documento a expediente |

### Expedientes

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/expedientes` | GET | `GET /api/v1/cases/` | Lista expedientes con filtros y paginacion |
| `/api/crear-expediente` | POST | `POST /api/v1/cases/` | Crear nuevo expediente |
| `/api/case-details` | GET | `GET /api/v1/cases/{id}` | Detalle del expediente |
| `/api/detalle-expediente` | GET | `GET /api/v1/cases/{id}/documents` | Documentos de un expediente |
| `/api/case-history` | GET | `GET /api/v1/cases/{id}/case-history` | Historial del expediente |
| `/api/case-movements` | GET | `GET /api/v1/cases/{id}/movements` | Movimientos del expediente |
| `/api/case-permissions` | GET | `GET /api/v1/cases/{id}/permissions` | Permisos del usuario en el expediente |
| `/api/assign-case` | POST | `POST /api/v1/cases/{id}/assign` | Asignar expediente a sector |
| `/api/transfer-case` | POST | `POST /api/v1/cases/{id}/transfer` | Transferir expediente |
| `/api/close-assign` | POST | `POST /api/v1/cases/{id}/close-assign` | Cerrar asignacion |
| `/api/prepare-assignment` | GET | `GET /api/v1/cases/{id}/prepare-assignment` | Datos para formulario de asignacion |
| `/api/prepare-transfer` | GET | `GET /api/v1/cases/{id}/prepare-transfer` | Datos para formulario de transferencia |
| `/api/proposed-document-action` | POST | `POST /api/v1/cases/{id}/proposed-documents/{pid}/accept` | Aceptar/rechazar doc propuesto |
| `/api/buscador-exp` | GET | `GET /api/v1/cases/number/{num}` | Buscar expediente por numero |
| `/api/expedientes/by-number/[caseNumber]` | GET | `GET /api/v1/cases/number/{num}` | Buscar expediente por numero (ruta alternativa) |
| `/api/filtra-trata` | GET | `GET /api/v1/cases/templates` | Obtener plantillas/tratas de expedientes |

### Notas

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/notes/received` | GET | `GET /notes/received` | Notas recibidas con paginacion |
| `/api/notes/sent` | GET | `GET /notes/sent` | Notas enviadas con paginacion |
| `/api/notes/archived` | GET | `GET /notes/archived` | Notas archivadas |
| `/api/notes/[id]` | GET | `GET /notes/{id}` | Detalle de nota |
| `/api/notes/[id]/archive` | POST | `POST /notes/{id}/archive` | Archivar/desarchivar nota |

### Usuarios y Sectores

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/users` | GET | `GET /users` | Lista de usuarios |
| `/api/users/me` | GET | `GET /users/profile` | Perfil del usuario actual |
| `/api/users/search` | GET | `GET /users/search` | Buscar usuarios |
| `/api/user-profile` | GET | `GET /users/profile` | Perfil de usuario (alternativa) |
| `/api/sectores` | GET | `GET /api/v1/sectors/` | Lista de sectores |
| `/api/sector-users` | GET | `GET /api/v1/cases/sectors/{id}/users` | Usuarios de un sector |

### Dashboard

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/dashboard/feed` | GET | `GET /api/v1/dashboard/feed` | Feed de actividad |
| `/api/dashboard/stats` | GET | `GET /api/v1/dashboard/stats` | Estadisticas del sector |

### Chat / Asistente

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/chat` | POST | `POST /chat` | Chat general con IA |
| `/api/case-chat` | POST | `POST /case-chat` | Chat contextual por expediente |
| `/api/case-welcome` | GET | `GET /case-welcome` | Mensaje de bienvenida contextual |

### Busqueda

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/unified-search` | GET | `GET /api/v1/search/unified` | Busqueda unificada (docs + expedientes) |

### Auth y Sistema

| Ruta API | Metodo | Backend Endpoint | Descripcion |
|----------|--------|-----------------|-------------|
| `/api/auth/onboarding` | GET | `GET /onboarding` | Datos de onboarding multi-tenant |
| `/api/csrf-token` | GET | - | Obtener token CSRF (local, no proxy) |
| `/api/health` | GET | - | Health check |
| `/api/index` | GET | - | Endpoint raiz (hello world) |

---

## Rutas Dinamicas

Las API routes con parametros usan el patron de Next.js `[param]`:

```
/api/documents/[documentId]/save.ts      -> /api/documents/abc-123/save
/api/document-details/[documentId].ts    -> /api/document-details/abc-123
/api/notes/[id].ts                       -> /api/notes/abc-123
/api/notes/[id]/archive.ts              -> /api/notes/abc-123/archive
/api/expedientes/by-number/[caseNumber].ts -> /api/expedientes/by-number/EE-2026-000020
```

---

## Manejo de Headers

Cada API route extrae los headers del cliente y los transforma para el backend:

```
Cliente (browser)                    API Route (server)                   Backend
+------------------+                +------------------------+           +------------------+
| Cookie: auth0    | -->  Extrae    | Authorization: Bearer  | -->       | Valida JWT       |
| X-Tenant-Schema  | --> Pasa      | X-Tenant-Schema        | -->       | Selecciona schema|
| X-User-ID (test) | --> Resuelve  | X-User-Email           | -->       | Identifica user  |
+------------------+                +------------------------+           +------------------+
```

---

## Manejo de Errores

Todas las API routes usan `handleApiError()` que:

1. Detecta errores de autenticacion (401)
2. Loguea el error con contexto
3. Retorna respuesta JSON estandarizada

```typescript
// Error de auth -> 401
{ "error": "Unauthorized", "message": "Session expired or invalid. Please login again." }

// Otros errores -> 500
{ "error": "Internal Server Error", "message": "Error description" }
```
