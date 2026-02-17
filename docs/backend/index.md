# Backend (GDI-Backend)

API REST principal del sistema GDI Latam. Gestiona documentos, expedientes, firmas digitales, usuarios y sectores para gobiernos municipales de Latinoamerica.

## Stack Tecnologico

| Componente | Tecnologia | Version |
|------------|-----------|---------|
| Lenguaje | Python | 3.12 |
| Framework | FastAPI | latest |
| Base de datos | PostgreSQL + pgvector | 17 |
| ORM/Driver | psycopg2 (raw SQL) | - |
| Validacion | Pydantic | v2 |
| Auth | Auth0 JWT (RS256) | - |
| Storage | Cloudflare R2 (S3-compatible) | - |
| Deploy | Railway | - |
| Server | Gunicorn + Uvicorn | 8 workers |

## Puerto

| Entorno | Puerto |
|---------|--------|
| Local | `8000` |
| Railway | Dinamico (variable `PORT`) |

## Arquitectura General

```
Cliente (Frontend/MCP)
        |
        v
   FastAPI App (:8000)
        |
        +-- TenantMiddleware (valida schema)
        +-- Auth (JWT Auth0)
        |
        v
   Endpoints (thin controllers)
        |
        v
   Services (logica de negocio)
        |
        +-- GDI-PDFComposer (generar PDFs)
        +-- GDI-Notary (firmar PDFs)
        +-- Cloudflare R2 (almacenar PDFs)
        +-- PostgreSQL (datos)
```

## Principios de Diseno

- **Endpoints thin**: Solo validacion y delegacion a services
- **Services con logica**: Toda la logica de negocio en la capa de servicios
- **Multi-tenant**: Cada municipalidad tiene su propio schema en PostgreSQL
- **keyword-only `schema_name`**: Patron obligatorio en todas las funciones de BD
- **Auth0 JWT**: Autenticacion via tokens RS256
- **Async-first**: Endpoints async con `async def`

## Integraciones Externas

| Servicio | URL Interna (Railway) | Proposito |
|----------|----------------------|-----------|
| GDI-PDFComposer | `pdfcomposer-svc.railway.internal` | Generar PDFs (preview, final, CAEX, PV) |
| GDI-Notary | `notary-svc.railway.internal` | Firma digital multi-firmante |
| Cloudflare R2 | S3 API | Almacenamiento de PDFs |
| Auth0 | `tu-tenant.us.auth0.com` | Autenticacion OAuth 2.0 |

## Variables de Entorno

| Variable | Descripcion |
|----------|-------------|
| `DATABASE_URL` | Connection string PostgreSQL |
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Conexion individual a BD |
| `AUTH0_DOMAIN` | Dominio Auth0 (`tu-tenant.us.auth0.com`) |
| `AUTH0_AUDIENCE` | Audience Auth0 |
| `AUTH0_CLIENT_ID` | Client ID Auth0 |
| `PDFCOMPOSER_URL` | URL interna GDI-PDFComposer |
| `PDFCOMPOSER_API_KEY` | API Key PDFComposer |
| `NOTARY_URL` | URL interna GDI-Notary |
| `NOTARY_API_KEY` | API Key Notary |
| `R2_ACCOUNT_ID` | Cloudflare account |
| `R2_ACCESS_KEY_ID` | R2 access key |
| `R2_SECRET_ACCESS_KEY` | R2 secret |
| `R2_BUCKET_NAME` | Bucket name |
| `TESTING_MODE` | `true`/`false` - bypass Auth0 en desarrollo |
| `PGBOUNCER_TRANSACTION_MODE` | `true`/`false` - modo transaccional PgBouncer |
| `FRONTEND_URL` | URL del frontend (CORS) |

## Comandos

```bash
# Desarrollo local
uvicorn main:app --reload --port 8000

# Produccion (Railway)
python server.py

# Health check
curl http://localhost:8000/health
```

## Contenido de esta Seccion

| Pagina | Descripcion |
|--------|-------------|
| [Estructura del Proyecto](estructura-proyecto.md) | Arbol de carpetas y organizacion |
| [Models y Schemas](models-schemas.md) | Modelos Pydantic de validacion |
| [Middleware](middleware.md) | TenantMiddleware, CORS |
| [Autenticacion](auth.md) | Auth0, JWT, permisos |
| [Base de Datos](database.md) | Pool, queries, multi-tenant |
| [Endpoints](endpoints/index.md) | Todos los endpoints REST |
| [Services](services/index.md) | Logica de negocio |
| [API Gateway](api-gateway/index.md) | MCP Server + REST API |
