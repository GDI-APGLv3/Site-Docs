# Diagrama de Servicios

## Arquitectura Completa

```mermaid
graph TB
    subgraph Usuarios["Usuarios"]
        U1["Usuario Municipal<br/>(Navegador)"]
        U2["Administrador<br/>(Navegador)"]
        U3["IA Externa<br/>(Claude, ChatGPT, Gemini)"]
    end

    subgraph Frontends["Frontends"]
        FE["GDI-FRONTEND<br/>Next.js 15 :3003"]
        BO["GDI-BackOffice-Front<br/>Next.js 15 :3013"]
    end

    subgraph Backends["Backends"]
        BE["GDI-Backend<br/>FastAPI :8000"]
        BOBE["GDI-BackOffice-Back<br/>FastAPI :8010"]
        MCP["GDI-MCP Server<br/>FastAPI :8005<br/>(integrado en Backend)"]
    end

    subgraph Microservicios["Microservicios"]
        PDF["GDI-PDFComposer<br/>FastAPI :8002"]
        NOT["GDI-Notary<br/>FastAPI :8001"]
        EMAIL["GDI-eMailService<br/>FastAPI :8003"]
        AGENT["GDI-AgenteLANG<br/>FastAPI :8004"]
        GOT["Gotenberg<br/>Chromium :3000"]
    end

    subgraph Storage["Almacenamiento"]
        PG["PostgreSQL 17<br/>+ pgvector"]
        R2["Cloudflare R2<br/>(S3-compatible)"]
    end

    subgraph Auth["Autenticacion"]
        A0["Auth0<br/>(OAuth 2.0 / JWT)"]
    end

    subgraph AI["Proveedores IA"]
        OR["OpenRouter<br/>(Gemini Flash, Llama 3.3)"]
    end

    U1 --> FE
    U2 --> BO
    U3 --> MCP

    FE -->|"REST + Auth0 JWT"| BE
    BO -->|"REST + Auth0 JWT"| BOBE

    BE -->|"REST + API Key"| PDF
    BE -->|"REST + API Key"| NOT
    BE -->|"REST + API Key"| EMAIL
    BE -->|"S3 API"| R2
    BE --> PG

    BOBE --> PG
    BOBE -->|"S3 API"| R2

    MCP -->|"OAuth 2.0"| A0

    AGENT -->|"REST + JWT"| BE
    AGENT --> PG
    AGENT -->|"API"| OR

    PDF -->|"HTTP"| GOT

    FE -->|"OAuth 2.0"| A0
    BO -->|"OAuth 2.0"| A0
```

## Diagrama por Capas

```mermaid
graph LR
    subgraph Presentacion["Capa de Presentacion"]
        F1["GDI-FRONTEND :3003"]
        F2["GDI-BackOffice-Front :3013"]
        F3["Dashboard Frontend :8501"]
    end

    subgraph Aplicacion["Capa de Aplicacion"]
        B1["GDI-Backend :8000"]
        B2["GDI-BackOffice-Back :8010"]
        B3["GDI-MCP Server :8005"]
        B4["Dashboard Backend :8000"]
    end

    subgraph Servicios["Capa de Microservicios"]
        S1["GDI-PDFComposer :8002"]
        S2["GDI-Notary :8001"]
        S3["GDI-eMailService :8003"]
        S4["GDI-AgenteLANG :8004"]
        S5["Gotenberg :3000"]
    end

    subgraph Datos["Capa de Datos"]
        D1["PostgreSQL + pgvector"]
        D2["Cloudflare R2"]
    end

    Presentacion --> Aplicacion
    Aplicacion --> Servicios
    Aplicacion --> Datos
    Servicios --> Datos
```

## Puertos y Tecnologias

| Servicio | Puerto | Tecnologia | Workers | Protocolo |
|----------|--------|------------|---------|-----------|
| GDI-FRONTEND | 3003 | Next.js 15 (Pages Router) | Node.js | HTTP |
| GDI-BackOffice-Front | 3013 | Next.js 15 (Pages Router) | Node.js | HTTP |
| GDI-Backend | 8000 | FastAPI + Gunicorn | 8 Uvicorn workers | HTTP |
| GDI-BackOffice-Back | 8010 | FastAPI + Uvicorn | 1 worker | HTTP |
| GDI-MCP Server | 8005 | FastAPI (integrado en Backend) | Compartido | MCP + HTTP |
| GDI-AgenteLANG | 8004 | FastAPI + Uvicorn | 1 + AIWorker background | HTTP |
| GDI-PDFComposer | 8002 | FastAPI + Gunicorn | 4 Uvicorn workers | HTTP |
| GDI-Notary | 8001 | FastAPI + Gunicorn | 3 Uvicorn workers | HTTP |
| GDI-eMailService | 8003 | FastAPI + Uvicorn | 1 worker | HTTP |
| Gotenberg | 3000 | Go + Chromium headless | - | HTTP |
| Dashboard Backend | 8000 | FastAPI + Uvicorn | 1 worker | HTTP |
| Dashboard Frontend | 8501 | Streamlit | 1 worker | HTTP |
| PostgreSQL | 5432 | PostgreSQL 17 + pgvector | Railway managed | TCP |

## Dependencias entre Servicios

```mermaid
graph TD
    FE["GDI-FRONTEND"] --> BE["GDI-Backend"]
    BO["GDI-BackOffice-Front"] --> BOBE["GDI-BackOffice-Back"]

    BE --> PDF["GDI-PDFComposer"]
    BE --> NOT["GDI-Notary"]
    BE --> EMAIL["GDI-eMailService"]
    BE --> R2["Cloudflare R2"]
    BE --> PG["PostgreSQL"]

    BOBE --> PG
    BOBE --> R2

    PDF --> GOT["Gotenberg"]

    AGENT["GDI-AgenteLANG"] --> BE
    AGENT --> PG
    AGENT --> OR["OpenRouter"]

    MCP["GDI-MCP Server"] --> PG
    MCP --> A0["Auth0"]

    DASH["Dashboard GDI"] --> PG

    FE --> A0
    BO --> A0
```

!!! note "MCP Server integrado"
    El MCP Server (puerto 8005) esta integrado dentro del repositorio GDI-Backend en la carpeta `api_gateway/`. No es un servicio separado en Railway, sino un modulo del Backend que escucha en un puerto adicional.
