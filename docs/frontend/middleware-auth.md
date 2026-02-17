# Middleware y Autenticacion

## Arquitectura de Auth

GDI-FRONTEND usa Auth0 como proveedor de identidad, integrado mediante `@auth0/nextjs-auth0` v4. La autenticacion se maneja en tres capas:

```
1. Middleware (server-side)     -> Protege rutas, redirige si no hay sesion
2. API Routes (server-side)     -> Obtiene token JWT para requests al backend
3. Client hooks (client-side)   -> Expone estado de auth a la UI
```

## Auth0 Client

**Archivo**: `src/lib/auth0.ts`

Instancia singleton del cliente Auth0 server-side:

```typescript
import { Auth0Client } from '@auth0/nextjs-auth0/server';

export const auth0 = new Auth0Client({
  authorizationParameters: {
    audience: process.env.AUTH0_AUDIENCE,
    scope: 'openid profile email offline_access',
  },
});
```

Se configura automaticamente desde variables de entorno: `AUTH0_SECRET`, `AUTH0_ISSUER_BASE_URL`, `AUTH0_CLIENT_ID`, `AUTH0_CLIENT_SECRET`, `AUTH0_BASE_URL`.

---

## Middleware

**Archivo**: `src/middleware.ts`

El middleware intercepta todas las requests (excepto assets estaticos y API routes) y maneja autenticacion.

### Rutas Protegidas

```typescript
const PROTECTED_ROUTES = [
  '/asistente',
  '/documentos',
  '/expedientes',
  '/notas',
  '/dashboard',
  '/mi-equipo',
  '/notificaciones',
  '/creacion-documento',
  '/previsualizar-documento',
  '/select-tenant',
];
```

### Flujo del Middleware

```
Request entrante
    |
    v
Es /auth/* ?  -->  Si: Delegar a Auth0 middleware (login, logout, callback, profile)
    |
    No
    v
Es ruta protegida?  -->  No: Dejar pasar (Next.js la sirve normalmente)
    |
    Si
    v
Tiene sesion Auth0?  -->  Si: Dejar pasar
    |
    No
    v
Es dev + tiene cookie user_id?  -->  Si: Dejar pasar (testing mode)
    |
    No
    v
Redirect a /?session_expired=true
```

### Matcher Config

El middleware se aplica a todas las rutas excepto:

- `/api/*` - API routes (tienen su propia auth)
- `/_next/static/*` - Assets estaticos
- `/_next/image/*` - Optimizacion de imagenes
- `favicon.ico`, `fondo.jpg`, etc. - Assets publicos

```typescript
export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico|favicon.png|fondo.jpg|robots.txt|sitemap.xml|manifest.json).*)',
  ],
};
```

---

## Rutas de Auth0

Auth0 maneja automaticamente estas rutas:

| Ruta | Proposito |
|------|-----------|
| `/auth/login` | Redirige a Auth0 Universal Login |
| `/auth/logout` | Cierra sesion y limpia cookies |
| `/auth/callback` | Procesa el callback OAuth tras login |
| `/auth/profile` | Retorna datos del usuario autenticado |

---

## Flujo de Login Completo

```
1. Usuario visita /dashboard (protegida)
    |
2. Middleware detecta sin sesion -> redirect a /?session_expired=true
    |
3. Usuario hace click en "Iniciar Sesion"
    |
4. Navega a /auth/login
    |
5. Auth0 Universal Login (email + password)
    |
6. Auth0 callback -> /auth/callback
    |
7. Sesion creada en cookies
    |
8. Redirect a / (landing)
    |
9. Landing detecta usuario autenticado
    |
10. useOnboarding() obtiene tenants del usuario
    |
    +--> 1 tenant:  Auto-select + redirect a /dashboard
    |
    +--> 2+ tenants: Redirect a /select-tenant
    |
11. Tenant seleccionado -> guardado en localStorage
    |
12. Redirect a /dashboard (con X-Tenant-Schema en todas las requests)
```

---

## Manejo de Token en Server-Side

**Archivo**: `src/lib/getServerToken.ts`

Las API routes necesitan obtener el token JWT de la sesion Auth0 para enviarlo al backend.

### Jerarquia de Autenticacion

```
Prioridad 1: Auth0 session -> JWT access token
Prioridad 2: Dev mode + X-User-ID header -> user_id como token
Prioridad 3: Sin auth -> Error 401
```

### Funciones Disponibles

=== "getServerToken (sin refresh)"

    ```typescript
    // Usa getSession() - NO refresca token automaticamente
    const token = await getServerToken(req);
    ```

=== "getServerTokenWithRefresh (con refresh)"

    ```typescript
    // Usa getAccessToken() - REFRESCA token si esta por expirar
    const token = await getServerTokenWithRefresh(req, res);
    ```

=== "getServerHeadersAsync (recomendado)"

    ```typescript
    // Obtiene token + email + construye headers completos
    const headers = await getServerHeadersAsync(req);
    // { Authorization, X-Tenant-Schema, X-User-Email, Content-Type, accept }
    ```

### Headers Generados

| Header | Valor | Descripcion |
|--------|-------|-------------|
| `Authorization` | `Bearer {jwt}` | Token JWT de Auth0 |
| `X-Tenant-Schema` | `200_muni` | Schema del tenant (pasado desde el cliente) |
| `X-User-Email` | `user@email.com` | Email como fallback para JWT sin email claim |
| `Content-Type` | `application/json` | Tipo de contenido (opcional) |
| `accept` | `application/json` | Formato de respuesta |

---

## Manejo de Token en Client-Side

### addTenantHeader

Todas las requests del cliente pasan por `addTenantHeader()` para agregar el schema del tenant.

```typescript
const response = await fetch('/api/documents', {
  headers: addTenantHeader({
    'Content-Type': 'application/json',
  }),
});
```

### fetchWithAuth

Wrapper de fetch que detecta 401 y redirige al login automaticamente.

```typescript
const response = await fetchWithAuth('/api/documents');
// Si 401 -> window.location.href = /auth/login?returnTo=...
```

---

## Modo Testing (Solo Desarrollo)

!!! warning "Solo en desarrollo"
    El modo testing solo funciona cuando `NEXT_PUBLIC_ENV !== 'production'`.

En desarrollo, se puede bypassear Auth0 proporcionando un `user_id`:

1. URL con query param: `/dashboard?user_id=UUID`
2. Cookie `user_id` en el navegador
3. Header `X-User-ID` en requests

El testing mode tiene timeout de 30 minutos y se gestiona via `testingMode.ts`.

---

## Proteccion CSRF

**Archivo**: `src/lib/csrf.ts`

Las API routes que aceptan POST usan proteccion CSRF con patron double-submit cookie:

1. El cliente solicita un token CSRF via `GET /api/csrf-token`
2. El servidor genera un token random de 32 bytes y lo setea como cookie `httpOnly`
3. El cliente envia el token en el header `x-csrf-token`
4. El servidor compara header vs cookie usando `crypto.timingSafeEqual`

```typescript
// En la API route
if (!requireCsrf(req, res)) return;  // Retorna 403 si invalido
```

---

## Logout

El proceso de logout limpia todos los estados:

1. Limpia sesion de testing (`clearTestingSession`)
2. Limpia datos multi-tenant (`clearAllTenantData`)
3. Limpia `sessionStorage` completo
4. Limpia `localStorage` completo
5. Elimina todas las cookies del dominio
6. Redirige a Auth0 logout (`/auth/logout?returnTo=...`)
