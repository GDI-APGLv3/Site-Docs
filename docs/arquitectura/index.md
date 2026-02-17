# Vision General de la Arquitectura

GDI Latam es un **Sistema de Gestion Documental Inteligente** para gobiernos y municipalidades de America Latina. La plataforma permite crear, firmar, numerar y gestionar documentos oficiales y expedientes administrativos de forma digital.

## Tabla de Servicios

| Servicio | Stack | Puerto | Repositorio | Descripcion |
|----------|-------|--------|-------------|-------------|
| GDI-FRONTEND | Next.js 15, React 18, TypeScript, Tailwind, shadcn/ui | 3003 | GDI-FRONTEND | Aplicacion de usuarios finales |
| GDI-BackOffice-Front | Next.js 15, React 18, TypeScript, Tailwind | 3013 | GDI-BackOffice-Front | Panel de administracion |
| GDI-Backend | Python 3.12, FastAPI, Pydantic, PostgreSQL 17 | 8000 | GDI-Backend | API principal de gestion documental |
| GDI-BackOffice-Back | Python 3.12, FastAPI, psycopg2 | 8010 | GDI-BackOffice-Back | API de administracion (49 endpoints + MCP) |
| GDI-MCP Server | FastAPI, MCP Protocol, OAuth 2.0 | 8005 | GDI-Backend (api_gateway/) | Server MCP para IA externa |
| GDI-AgenteLANG | FastAPI, LangGraph, OpenRouter, pgvector | 8004 | GDI-AgenteLANG | Agente IA con RAG y chat |
| GDI-PDFComposer | FastAPI, Gotenberg 7, Jinja2 | 8002 | GDI-PDFComposer | Motor de generacion de PDFs |
| GDI-Notary | FastAPI, pyHanko, PyMuPDF, ReportLab | 8001 | GDI-Notary | Firma digital PAdES y visual |
| GDI-eMailService | FastAPI, Jinja2, SMTP | 8003 | GDI-eMailService | Envio de emails transaccionales |
| Dashboard GDI | Streamlit + FastAPI | 8501/8000 | Dashboard GDI | Explorador de base de datos |

## Principios Arquitectonicos

### Separacion de responsabilidades

Cada capa del sistema tiene una responsabilidad clara e inviolable:

- **Frontends**: Solo interfaz de usuario. Sin logica de negocio.
- **Backends**: Toda la logica de negocio, validaciones y orquestacion.
- **Microservicios**: Funciones atomicas y stateless (generar PDF, firmar, enviar email).
- **Base de datos**: PostgreSQL como fuente de verdad, con pgvector para busqueda semantica.

### Multi-tenant por schema

Cada municipio opera en su propio schema de PostgreSQL. Esto garantiza aislamiento total de datos entre organizaciones sin necesidad de bases de datos separadas.

```
PostgreSQL
├── public             # Tablas globales (roles, tipos globales, api_keys)
├── 200_muni           # Schema del municipio "Test" (33 tablas)
├── 200_muni_audit     # Auditoria del municipio "Test"
├── 200_salta          # Schema de otro municipio
└── 200_salta_audit    # Su auditoria
```

!!! warning "Regla critica: schema_name keyword-only"
    Todas las funciones que acceden a BD usan `schema_name` como parametro keyword-only.
    Siempre escribir `schema_name=schema_name`, nunca posicional.

### Microservicios stateless

Los microservicios (PDFComposer, Notary, eMailService) no mantienen estado propio. Reciben un request, procesan y responden. Esto permite escalar horizontalmente sin complejidad.

La unica excepcion parcial es **GDI-AgenteLANG**, que mantiene un AIWorker background para procesamiento asincrono de resumenes, embeddings y transcripciones.

### Comunicacion sincrona via REST

Todos los servicios se comunican via HTTP REST. No se usan colas de mensajes, gRPC ni WebSockets. La simplicidad de REST sincrono es suficiente para el volumen actual y reduce la complejidad operativa.

| Patron | Solucion adoptada | Alternativa evitada |
|--------|-------------------|---------------------|
| Comunicacion entre servicios | REST sincrono | gRPC, GraphQL |
| Cola de tareas | PostgreSQL polling (AIWorker) | Redis, RabbitMQ |
| Cache | No se usa (aun) | Redis prematuro |
| Real-time | Polling / SSE | WebSockets |
| Autenticacion | Auth0 JWT (OAuth 2.0) | Autenticacion custom |

### Storage distribuido

- **PDFs**: Cloudflare R2 (S3-compatible) con dos buckets por tenant: `tosign` (pendientes) y `oficial` (firmados).
- **Embeddings**: pgvector en PostgreSQL para busqueda semantica RAG.
- **Imagenes de config**: Cloudflare R2 (logos, isologos) gestionados por BackOffice.

### Deploy en Railway

Todos los servicios se despliegan en Railway como PaaS. Cada servicio tiene su propio deployment con auto-deploy desde GitHub al hacer push a `main`. La comunicacion interna entre servicios usa Railway internal URLs para menor latencia y mayor seguridad.

## Repositorios

El proyecto esta organizado como **multi-repo**: cada carpeta en el directorio raiz es un repositorio Git independiente. No es un monorepo.

```
mi-proyecto/          # Directorio raiz (NO es repo git)
├── GDI-FRONTEND/      # App usuarios (Next.js :3003)
├── GDI-Backend/       # API principal (FastAPI :8000)
├── GDI-BackOffice-Front/  # Admin UI (Next.js :3013)
├── GDI-BackOffice-Back/   # Admin API (FastAPI :8010)
├── GDI-AgenteLANG/    # Agente IA (FastAPI :8004)
├── GDI-PDFComposer/   # Genera PDFs (FastAPI :8002)
├── GDI-Notary/        # Firma digital (FastAPI :8001)
├── GDI-eMailService/  # Emails (FastAPI :8003)
├── GDI-BD/            # Scripts SQL de BD
├── Dashboard GDI/     # Explorador DB (Streamlit)
└── .claude/           # Documentacion interna y agentes
```

!!! info "Git multi-repo"
    Nunca ejecutar `git` desde el directorio raiz. Siempre usar `git -C <repo>` para operar sobre un repositorio especifico.
