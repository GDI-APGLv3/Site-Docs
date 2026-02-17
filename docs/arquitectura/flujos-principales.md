# Flujos Principales

## 1. Crear Documento y Firmar

El flujo completo de un documento desde su creacion como borrador hasta su firma oficial y almacenamiento.

```mermaid
sequenceDiagram
    participant U as Usuario
    participant FE as Frontend
    participant BE as Backend
    participant PDF as PDFComposer
    participant NOT as Notary
    participant R2 as Cloudflare R2
    participant PG as PostgreSQL

    Note over U,PG: Fase 1: Creacion del borrador
    U->>FE: Crea documento (Editor Quill)
    FE->>BE: POST /documents
    BE->>PG: INSERT document_draft (status: draft)
    BE-->>FE: document_id

    U->>FE: Edita contenido HTML + agrega firmantes
    FE->>BE: PATCH /documents/{id}
    BE->>PG: UPDATE document_draft + INSERT document_signers

    Note over U,PG: Fase 2: Preview con marca de agua
    U->>FE: Solicita preview
    FE->>BE: POST /documents/preview
    BE->>PDF: POST /preview-pdf (HTML content)
    PDF-->>BE: PDF bytes (con watermark)
    BE-->>FE: PDF preview
    FE-->>U: Visualiza PDF con marca "PREVISUALIZACION"

    Note over U,PG: Fase 3: Envio a firma
    U->>FE: "Enviar a firma"
    FE->>BE: POST /documents/{id}/start-signing-process
    BE->>PDF: POST /generate-pdf (HTML, sin watermark)
    PDF-->>BE: PDF bytes (final)
    BE->>R2: Upload PDF (bucket: tosign)
    BE->>PG: UPDATE status = 'sent_to_sign'
    BE-->>FE: OK

    Note over U,PG: Fase 4: Firma digital
    U->>FE: "Firmar documento"
    FE->>BE: POST /documents/{id}/sign
    BE->>PG: Genera numero oficial (advisory lock 888888)
    BE->>R2: Download PDF de tosign
    BE->>NOT: POST /sign-pdf (PDF + datos firmante + tenant_id)
    NOT-->>BE: PDF firmado (PAdES o visual)
    BE->>R2: Upload PDF firmado (bucket: oficial)
    BE->>R2: Delete PDF de tosign
    BE->>PG: INSERT official_documents + UPDATE status = 'signed'
    BE-->>FE: Documento firmado con numero oficial
    FE-->>U: Muestra PDF firmado
```

**Formato de numero oficial:**

```
{TIPO}-{ANO}-{SECUENCIA:08d}-{CIUDAD}-{DEPT}
Ejemplo: IF-2025-00000034-SMG-ADGEN
```

**Manejo FULLPAGE:** Si Notary responde `400 FULLPAGE` (pagina llena), el Backend agrega una pagina en blanco y reintenta automaticamente.

---

## 2. Crear Expediente con Caratula CAEX

La creacion de un expediente genera automaticamente una caratula oficial (documento CAEX) firmada.

```mermaid
sequenceDiagram
    participant U as Usuario
    participant FE as Frontend
    participant BE as Backend
    participant PDF as PDFComposer
    participant NOT as Notary
    participant R2 as Cloudflare R2
    participant PG as PostgreSQL

    U->>FE: Completa formulario de expediente
    FE->>BE: POST /cases

    Note over BE,PG: BEGIN TRANSACTION
    BE->>PG: INSERT cases (genera numero EE-2025-XXXXXX)
    BE->>PG: INSERT case_movements (tipo: creation)

    Note over BE,R2: Generacion automatica de CAEX
    BE->>PG: INSERT document_draft (tipo: CAEX)
    BE->>PG: Genera numero CAEX-2025-XXXXXXXX (advisory lock)
    BE->>PG: INSERT official_documents
    BE->>PDF: POST /create-case (datos del expediente)
    PDF-->>BE: PDF caratula
    BE->>NOT: POST /sign-pdf (caratula + tenant_id)
    NOT-->>BE: Caratula firmada
    BE->>R2: Upload caratula (bucket: oficial)
    BE->>PG: INSERT case_official_documents (order_number = 1)
    Note over BE,PG: COMMIT

    BE-->>FE: Expediente creado
    FE-->>U: Expediente con caratula firmada
```

La caratula CAEX siempre es el **primer documento** del expediente (`order_number = 1`). Contiene:

- Numero de expediente
- Tipo y motivo
- Area iniciadora
- Creador
- Fecha y hora (UTC)
- Logo municipal

---

## 3. Asignar Expediente con PV Automatica

Al asignar un expediente a otro sector, opcionalmente se genera un documento PV (Pase/Vista) automatico.

```mermaid
sequenceDiagram
    participant U as Usuario
    participant FE as Frontend
    participant BE as Backend
    participant PDF as PDFComposer
    participant NOT as Notary
    participant R2 as Cloudflare R2
    participant PG as PostgreSQL

    U->>FE: Selecciona expediente y sector destino
    FE->>BE: POST /cases/{id}/assign (create_official_doc: true)

    BE->>PG: Valida permisos del usuario

    Note over BE,R2: Generacion automatica de PV (si solicitado)
    BE->>PG: INSERT document_draft (tipo: PV)
    BE->>PG: Genera numero PV-2025-XXXXXXXX (advisory lock)
    BE->>PG: INSERT official_documents
    BE->>PDF: POST /move (datos del movimiento)
    PDF-->>BE: PDF pase de vista
    BE->>NOT: POST /sign-pdf (PV + tenant_id)
    NOT-->>BE: PV firmado
    BE->>R2: Upload PV (bucket: oficial)
    BE->>PG: INSERT case_official_documents (order_number auto)

    Note over BE,PG: Registro del movimiento
    BE->>PG: INSERT case_movements (tipo: assignment)
    BE->>PG: Cierra movimiento anterior (closed_at, closing_reason)

    BE-->>FE: Asignacion completada
    FE-->>U: Expediente asignado con PV adjunta
```

El documento PV contiene:

- Numero de expediente
- Tipo de movimiento (Asignacion)
- Area requiriente (DE)
- Area receptora (A)
- Motivo del movimiento

---

## 4. Chat con Agente IA

El agente IA utiliza LangGraph con un Router que clasifica la intencion del usuario y deriva al nodo especializado.

```mermaid
sequenceDiagram
    participant U as Usuario
    participant FE as Frontend
    participant AG as AgenteLANG
    participant R as Router (Llama 3.3)
    participant N as Nodo Especializado
    participant BE as Backend
    participant PG as PostgreSQL
    participant LLM as Respond (Gemini Flash)

    U->>FE: Escribe mensaje en chat
    FE->>AG: POST /api/v1/chat (message + JWT)
    AG->>AG: Valida JWT, extrae user_id y schema

    Note over AG,R: Clasificacion de intent
    AG->>R: Clasifica mensaje
    R-->>AG: intent: cases | documents | rag | general

    alt Intent: cases
        AG->>N: Cases Node
        N->>BE: search_cases / get_case / get_case_history
        BE->>PG: Query
        PG-->>BE: Datos
        BE-->>N: Resultado
    else Intent: documents
        AG->>N: Documents Node
        N->>BE: search_documents / get_document
        BE->>PG: Query
        PG-->>BE: Datos
        BE-->>N: Resultado
    else Intent: rag
        AG->>N: RAG Node
        N->>PG: Busqueda semantica (pgvector)
        PG-->>N: Documentos similares
    else Intent: general
        AG->>N: General Node (sin tools)
    end

    Note over AG,LLM: Generacion de respuesta
    AG->>LLM: Contexto + resultados de tools
    LLM-->>AG: Respuesta natural
    AG-->>FE: Streaming de respuesta
    FE-->>U: Muestra respuesta del asistente
```

**Modelos utilizados:**

| Modelo | Uso | Costo |
|--------|-----|-------|
| Llama 3.3 70B (FREE) | Router - Clasificacion de intent | Gratis |
| Gemini Flash 2.0 | Nodos especializados + Respuesta final | USD 0.10/1M tokens input |

**Case Chat** (`/api/v1/cases/{id}/chat`): Un chat focalizado que precarga todo el contexto del expediente. No necesita Router porque todas las consultas son sobre ese expediente.

---

## 5. MCP Server (IA Externa)

Flujo de conexion de un cliente MCP externo (Claude Code, ChatGPT, Gemini) al servidor MCP de GDI.

```mermaid
sequenceDiagram
    participant C as Cliente MCP (Claude/ChatGPT)
    participant M as MCP Server (:8005)
    participant A as Auth0
    participant U as Usuario (Navegador)
    participant PG as PostgreSQL

    Note over C,PG: Fase 1: Discovery
    C->>M: POST /mcp (sin auth)
    M-->>C: 401 + WWW-Authenticate
    C->>M: GET /.well-known/oauth-protected-resource
    M-->>C: {authorization_servers: ["https://tu-tenant.us.auth0.com"]}

    Note over C,U: Fase 2: Autenticacion OAuth 2.0
    C->>A: GET /.well-known/openid-configuration
    A-->>C: Metadata OAuth (authorize_endpoint, token_endpoint)
    C->>U: Abre navegador para login
    U->>A: Login con credenciales
    A-->>C: Authorization code
    C->>A: POST /oauth/token (code → JWT)
    A-->>C: JWT (access_token)

    Note over C,PG: Fase 3: Uso de Tools MCP
    C->>M: POST /mcp (tools/list, Bearer JWT)
    M->>M: Valida JWT, extrae user_id
    M-->>C: 14 tools disponibles

    C->>M: POST /mcp (tools/call: search_cases)
    M->>PG: Query con schema del usuario
    PG-->>M: Resultados
    M-->>C: Resultados de busqueda

    C->>M: POST /mcp (tools/call: get_case_history)
    M->>PG: Query historial + ai_summary
    PG-->>M: Historial
    M-->>C: Historial con resumen IA
```

**14 Tools MCP disponibles:**

| Categoria | Tools | Descripcion |
|-----------|-------|-------------|
| Expedientes | `search_cases`, `get_case`, `get_case_history`, `get_case_documents`, `get_case_permissions` | Busqueda y consulta de expedientes |
| Documentos | `search_documents`, `get_document`, `get_document_content`, `get_pending_signatures` | Busqueda y consulta de documentos |
| Sistema | `get_document_types`, `get_sectors`, `get_user_info`, `get_case_templates` | Informacion del sistema |
| Utilidades | `get_agent_guide` | Guia completa para el agente |

---

## 6. Indexacion RAG (AIWorker Background)

El AIWorker de GDI-AgenteLANG procesa documentos en background para habilitar busqueda semantica.

```mermaid
sequenceDiagram
    participant W as AIWorker (cada 60s)
    participant PG as PostgreSQL
    participant LLM as LLM (Gemini Flash)
    participant EMB as Embeddings (OpenAI)
    participant R2 as Cloudflare R2

    Note over W,R2: Tarea 1: Resumir borradores enviados a firma
    W->>PG: SELECT FROM document_draft WHERE status='sent_to_sign' AND resume IS NULL
    PG-->>W: Documentos sin resumen
    loop Cada documento
        W->>LLM: Genera resumen del contenido HTML
        LLM-->>W: Texto resumen
        W->>PG: UPDATE document_draft SET resume = '...'
    end

    Note over W,R2: Tarea 2: Resumir documentos oficiales
    W->>PG: SELECT FROM official_documents WHERE resume IS NULL
    PG-->>W: Docs oficiales sin resumen
    loop Cada documento
        W->>LLM: Genera resumen
        LLM-->>W: Texto resumen
        W->>PG: UPDATE official_documents SET resume = '...'
    end

    Note over W,R2: Tarea 3: Indexacion (chunks + embeddings)
    W->>PG: SELECT docs sin chunks en document_chunks
    PG-->>W: Documentos no indexados
    loop Cada documento
        W->>W: Chunker (HTML a fragmentos)
        W->>EMB: text-embedding-3-small (1536 dims)
        EMB-->>W: Vectores
        W->>PG: INSERT document_chunks (texto + vector)
    end

    Note over W,R2: Tarea 4: Transcripcion de PDFs importados
    W->>PG: SELECT tipo 'Importado' con content IS NULL
    PG-->>W: PDFs sin transcribir
    loop Cada PDF
        W->>R2: Download PDF
        R2-->>W: PDF bytes
        W->>W: PyMuPDF (PDF a imagenes PNG)
        W->>LLM: Gemini Flash 1.5 Vision
        LLM-->>W: Texto transcrito
        W->>PG: UPDATE content (con marcador IA Vision)
    end
```

**Control de costos:** El AIWorker incluye tracking de costos por schema con limites diarios y mensuales configurables en la tabla `ai_usage_limits`.
