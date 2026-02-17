# Frontend - GDI-FRONTEND

GDI-FRONTEND es la aplicacion web para usuarios finales del Sistema de Gestion Documental Inteligente. Permite crear, editar, firmar y gestionar documentos oficiales, expedientes administrativos y notas internas.

## Stack Tecnologico

| Componente | Tecnologia | Version |
|------------|------------|---------|
| Framework | Next.js (Pages Router) | 15.5.9 |
| UI Library | React | 18.3.1 |
| Lenguaje | TypeScript (strict) | 5.x |
| Estilos | Tailwind CSS + tailwindcss-animate | 3.4.3 |
| Componentes UI | shadcn/ui (New York style) | - |
| Primitivas | Radix UI (Dialog, Select, Tabs, Popover, Tooltip, Dropdown) | - |
| Iconos | Lucide React | 0.563.0 |
| Editor Rich Text | React Quill | 2.0.0 |
| Command Palette | cmdk | 1.1.1 |
| Autenticacion | Auth0 via @auth0/nextjs-auth0 | 4.14.0 |
| Deploy | Railway | - |
| Puerto local | 3003 | - |

## Funcionalidades Principales

- **Documentos**: Crear, editar con editor rich text (Quill), previsualizar PDF, enviar a firma
- **Firma Digital**: Proceso de firma multi-firmante con orden secuencial, firma conjunta por lotes
- **Expedientes**: Crear, asignar a sectores, transferir, vincular documentos, timeline de movimientos
- **Notas Internas**: Enviar/recibir notas entre sectores con destinatarios TO/CC/BCC y archivado
- **Dashboard**: Feed de actividad con movimientos de expedientes, estadisticas por sector
- **Busqueda Unificada**: Buscar documentos y expedientes por numero oficial o texto libre
- **Asistente IA**: Chat con agente inteligente por expediente o general (RAG)
- **Multi-Tenant**: Soporte para multiples municipios con colores dinamicos por tenant

## Comunicacion con Backend

El frontend se comunica con GDI-Backend (:8000) exclusivamente a traves de API Routes de Next.js (`/api/*`), que actuan como proxy server-side. El cliente nunca contacta al backend directamente.

```
Browser  -->  Next.js API Route (/api/*)  -->  GDI-Backend (:8000)
                  (server-side proxy)
```

## Subsecciones

| Seccion | Descripcion |
|---------|-------------|
| [Estructura del Proyecto](estructura-proyecto.md) | Arbol de carpetas y organizacion del codigo |
| [Paginas](paginas.md) | Todas las rutas y paginas de la aplicacion |
| [Componentes](componentes.md) | Inventario de componentes por modulo |
| [Hooks](hooks.md) | Custom hooks con parametros y tipos de retorno |
| [Types](types.md) | Interfaces TypeScript agrupadas por dominio |
| [Lib y Utilidades](lib-utils.md) | fetchWithAuth, tenant utils, theme, validacion |
| [Middleware y Auth](middleware-auth.md) | Flujo Auth0 completo y proteccion de rutas |
| [Design System](design-system.md) | Colores, badges, iconografia y patrones de layout |
| [API Routes](api-routes.md) | Proxy routes y endpoints del backend |

## Comandos

```bash
# Desarrollo local (puerto 3003)
npm run dev

# Build de produccion
npm run build

# Iniciar en produccion
npm run start

# Linting
npm run lint
```

## Variables de Entorno

| Variable | Descripcion |
|----------|-------------|
| `AUTH0_SECRET` | Secret para cifrado de sesiones |
| `AUTH0_BASE_URL` | URL base de la aplicacion (ej: `http://localhost:3003`) |
| `AUTH0_ISSUER_BASE_URL` | URL del tenant Auth0 |
| `AUTH0_CLIENT_ID` | Client ID de la aplicacion en Auth0 |
| `AUTH0_CLIENT_SECRET` | Client Secret de Auth0 |
| `AUTH0_AUDIENCE` | API Audience de Auth0 |
| `NEXT_PUBLIC_API_URL` | URL del backend API (ej: `http://localhost:8000`) |
| `NEXT_PUBLIC_ENV` | Entorno: `production` o vacio para desarrollo |
