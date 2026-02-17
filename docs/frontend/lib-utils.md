# Lib y Utilidades

Las utilidades compartidas estan en `src/lib/`. Proporcionan funciones de autenticacion, fetching, multi-tenant, tema y validacion usadas en toda la aplicacion.

## Resumen de Archivos

| Archivo | Proposito |
|---------|-----------|
| `auth0.ts` | Instancia del cliente Auth0 |
| `fetchWithAuth.ts` | Fetch con manejo automatico de 401 |
| `fetchWithTimeout.ts` | Fetch con timeout configurable |
| `getServerToken.ts` | Obtencion de token y headers en server-side |
| `tenant.ts` | Storage y headers multi-tenant |
| `theme.ts` | Colores dinamicos por tenant (HEX -> HSL -> CSS vars) |
| `csrf.ts` | Proteccion CSRF con tokens |
| `apiErrorHandler.ts` | Manejo de errores en API routes |
| `validation.ts` | Validacion de parametros y sanitizacion |
| `searchUtils.ts` | Deteccion de tipo de busqueda |
| `documentUtils.ts` | Patrones de numeros oficiales |
| `caseUtils.ts` | Patrones de numeros de expediente |
| `utils.ts` | Utilidad `cn()` de shadcn/ui |
| `logger.ts` | Sistema de logging por modulo |
| `urlValidation.ts` | Validacion de URLs de retorno |
| `testingMode.ts` | Modo testing para desarrollo |

---

## fetchWithAuth

**Archivo**: `src/lib/fetchWithAuth.ts`

Wrapper de `fetch` que intercepta respuestas 401 y redirige automaticamente al login.

```typescript
import { fetchWithAuth } from '@/lib/fetchWithAuth';

// Si retorna 401, redirige a /auth/login con returnTo
const response = await fetchWithAuth('/api/documents', { method: 'GET' });
```

**Implementacion**:

```typescript
export async function fetchWithAuth(
  input: RequestInfo | URL,
  init?: RequestInit
): Promise<Response> {
  const response = await fetch(input, init);

  if (response.status === 401) {
    if (typeof window !== 'undefined') {
      const returnTo = getCurrentReturnUrl(true);
      window.location.href = `/auth/login?returnTo=${encodeURIComponent(returnTo)}`;
    }
    throw new Error('Session expired - redirecting to login');
  }

  return response;
}
```

Tambien exporta `handleAuthError(response)` para manejo manual.

---

## fetchWithTimeout

**Archivo**: `src/lib/fetchWithTimeout.ts`

Fetch con timeout configurable via `AbortController`.

```typescript
import { fetchWithTimeout } from '@/lib/fetchWithTimeout';

const response = await fetchWithTimeout(url, {
  timeout: 15000,  // 15 segundos (default)
  method: 'POST',
  body: JSON.stringify(data),
});
```

Lanza `Error('Request timeout after ${timeout}ms')` si se excede el tiempo.

---

## getServerToken

**Archivo**: `src/lib/getServerToken.ts`

Funciones server-side para obtener tokens y construir headers de autenticacion para requests al backend.

### Funciones Principales

| Funcion | Descripcion |
|---------|-------------|
| `getServerToken(req)` | Obtiene token (Auth0 o testing fallback) |
| `getServerTokenWithRefresh(req, res)` | Obtiene token con auto-refresh de Auth0 |
| `getServerTokenWithEmail(req)` | Obtiene token + email en una llamada |
| `getServerHeaders(token, req, includeContentType?, email?)` | Construye headers con Authorization, X-Tenant-Schema, X-User-Email |
| `getServerHeadersAsync(req, includeContentType?)` | Version async que obtiene token + email + headers automaticamente |

### Flujo de Prioridad

```
1. Auth0 session valida       -> JWT token
2. Dev mode + user_id header  -> user_id como token (testing)
3. Ninguno                    -> Error: No authentication
```

### Ejemplo de Uso en API Route

```typescript
import { getServerHeadersAsync } from '@/lib/getServerToken';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const headers = await getServerHeadersAsync(req);
  // headers = { Authorization: "Bearer ...", X-Tenant-Schema: "100_test", ... }

  const response = await fetch(backendUrl, { headers });
  const data = await response.json();
  res.status(200).json(data);
}
```

---

## tenant.ts

**Archivo**: `src/lib/tenant.ts`

Utilidades client-side para manejo multi-tenant.

### tenantStorage

Persiste el tenant seleccionado en `localStorage` (sobrevive entre sesiones).

```typescript
tenantStorage.get();              // TenantAccess | null
tenantStorage.set(tenant);        // Guarda y aplica colores del tema
tenantStorage.clear();            // Elimina tenant
```

### profileStorage

Persiste el perfil de usuario en `sessionStorage` (se pierde al cerrar tab).

```typescript
profileStorage.get();             // TenantProfile | null
profileStorage.set(profile);
profileStorage.clear();
```

### addTenantHeader

Agrega headers multi-tenant a cualquier request del cliente.

```typescript
import { addTenantHeader } from '@/lib/tenant';

const response = await fetch('/api/documents', {
  headers: addTenantHeader({
    'Content-Type': 'application/json',
  }),
});
// Agrega automaticamente:
// X-Tenant-Schema: "100_test"
// X-User-ID: "uuid..." (solo en testing mode)
```

### clearAllTenantData

Limpia todo: tenant de localStorage, perfil de sessionStorage y resetea tema.

```typescript
clearAllTenantData();  // Llamar en logout
```

### initializeThemeFromStorage

Aplica colores del tenant guardado al cargar la app. Se llama en `_app.tsx`.

---

## theme.ts

**Archivo**: `src/lib/theme.ts`

Generacion de colores dinamicos por tenant. Convierte un color HEX a una escala HSL completa (50-900) y la aplica como CSS variables.

### Flujo

```
HEX (#1E3A8A) -> RGB -> HSL (con rangos validados) -> Escala de 10 niveles -> CSS vars
```

### Funciones

| Funcion | Descripcion |
|---------|-------------|
| `hexToHSL(hex)` | Convierte HEX a HSL con validacion de rangos (S >= 30%, 20% <= L <= 60%) |
| `getContrastColor(hex)` | Determina color de contraste (blanco o negro) usando luminance WCAG |
| `generateColorScale(h, s, l)` | Genera escala 50-900 con ajustes adaptativos por oscuridad |
| `applyThemeColors(primaryHex)` | Aplica toda la escala como `--color-primary-*` en `:root` |
| `resetThemeColors()` | Elimina variables custom, vuelve a defaults de globals.css |

### Ejemplo de Escala Generada

Para un color base `#1E3A8A`:

```css
--color-primary-50: hsl(220, 45%, 75%);   /* Muy claro */
--color-primary-100: hsl(220, 45%, 70%);
--color-primary-200: hsl(220, 52%, 62%);
--color-primary-300: hsl(220, 60%, 52%);
--color-primary-400: hsl(220, 75%, 42%);
--color-primary-500: hsl(220, 75%, 33%);  /* Base */
--color-primary-600: hsl(220, 75%, 25%);
--color-primary-700: hsl(220, 75%, 18%);
--color-primary-800: hsl(220, 75%, 11%);
--color-primary-900: hsl(220, 75%, 5%);   /* Muy oscuro */
--color-primary-foreground: #FFFFFF;
```

---

## csrf.ts

**Archivo**: `src/lib/csrf.ts`

Proteccion CSRF con patron double-submit cookie.

```typescript
generateCsrfToken();                    // Genera token de 32 bytes hex
setCsrfCookie(res, token);              // Setea cookie httpOnly, secure, sameSite=strict
validateCsrfToken(req);                 // Compara header vs cookie (timing-safe)
requireCsrf(req, res);                  // Valida y retorna 403 si invalido
getCsrfTokenFromCookie(req);            // Lee token de la cookie
```

---

## apiErrorHandler.ts

**Archivo**: `src/lib/apiErrorHandler.ts`

Manejo estandarizado de errores en API routes.

```typescript
import { handleApiError } from '@/lib/apiErrorHandler';

export default async function handler(req, res) {
  try {
    // ... logica
  } catch (error) {
    handleApiError(error, res, 'documents');
    // Retorna 401 si es error de auth, 500 en otros casos
  }
}
```

---

## validation.ts

**Archivo**: `src/lib/validation.ts`

Utilidades de validacion para API routes.

```typescript
validateQueryParam(param);                    // string | null
validateQueryParams(query, requiredParams, res);  // Valida y retorna 400 si falta algo
isValidUUID(str);                             // Valida formato UUID v4
sanitizeString(str, maxLength?);              // Limpia < > y trunca
```

---

## searchUtils.ts

**Archivo**: `src/lib/searchUtils.ts`

Deteccion automatica del tipo de busqueda.

```typescript
detectSearchType("INF-2026-00001234-TXST-LEGAL");  // 'document_official'
detectSearchType("EE-2026-000020-TXST-INTE");       // 'case_official'
detectSearchType("habilitacion comercial");          // 'free_text'

detectPartialType("INF-2");   // 'document'
detectPartialType("EE-20");   // 'case'
```

---

## utils.ts (shadcn)

**Archivo**: `src/lib/utils.ts`

La clasica utilidad `cn()` de shadcn/ui para combinar clases Tailwind.

```typescript
import { cn } from "@/lib/utils";

<div className={cn("p-4 rounded", isActive && "bg-primary-50 border-primary-500")} />
```

Usa `clsx` para logica condicional y `tailwind-merge` para resolver conflictos.

---

## API Config

**Archivo**: `src/config/api.config.ts`

Configuracion centralizada de todos los endpoints del backend. Define paths como funciones type-safe.

```typescript
import { API_CONFIG } from '@/config/api.config';

// Construir URL completa
const url = API_CONFIG.url(API_CONFIG.endpoints.documents.currentUserList());
// -> "http://localhost:8000/users/documents"

const url2 = API_CONFIG.url(API_CONFIG.endpoints.cases.getById(caseId));
// -> "http://localhost:8000/api/v1/cases/{caseId}"
```

### Dominios de Endpoints

| Dominio | Endpoints |
|---------|-----------|
| `documents` | `list`, `currentUserList`, `create`, `editorDetails`, `previewInfo`, `previewDownload`, `signatureDetails`, `save`, `startSigningProcess`, `reject`, `delete`, `autocomplete`, `getPdfUrl`, `unifiedDetails`, `searchOfficial` |
| `documentTypes` | `list` |
| `users` | `list`, `search`, `profile` |
| `cases` | `list`, `create`, `getById`, `details`, `search`, `templates`, `linkDocument`, `subsanar`, `prepareAssignment`, `assign`, `movements`, `history`, `closeAssign`, `prepareTransfer`, `transfer`, `permissions`, `acceptProposed`, `rejectProposed` |
| `sectors` | `list`, `getUsers` |
| `notes` | `received`, `sent`, `archived`, `getById`, `archive` |
| `dashboard` | `feed`, `stats` |
