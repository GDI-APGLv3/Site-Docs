# Glosario Tecnico

Terminos tecnicos utilizados en la documentacion de GDI.

## A

**Advisory Lock**
:   Lock a nivel de PostgreSQL (`pg_advisory_xact_lock`) usado para garantizar numeracion secuencial sin colisiones. GDI usa el key `888888`.

**API Gateway**
:   Modulo en GDI-Backend (`api_gateway/`) que expone endpoints REST y MCP Server para integracion con IAs externas (Claude, ChatGPT, Gemini).

**Auth0**
:   Servicio de autenticacion externo. Frontend obtiene JWT tokens, Backend los valida.

## C

**CAEX**
:   Caratula de Expediente. PDF autogenerado que sirve como portada del expediente.

**Cloudflare R2**
:   Object storage compatible con S3 donde se almacenan los PDFs generados y firmados.

## D

**Disposicion**
:   Tipo de documento oficial. Requiere firma del funcionario con rank de Titular.

**Decreto**
:   Tipo de documento oficial. Requiere firma del funcionario con rank de Titular.

## E

**execute_query**
:   Funcion central en `db.py` del Backend para ejecutar queries SQL con schema multi-tenant.

**Expediente**
:   Carpeta digital que agrupa documentos relacionados. Tiene numero unico, caratula y secciones.

## F

**fetchWithAuth**
:   Funcion wrapper en el Frontend que agrega token Auth0 a todas las peticiones HTTP.

## G

**Gotenberg**
:   Servicio Docker que convierte HTML a PDF. Usado por PDFComposer.

## J

**JWT (JSON Web Token)**
:   Token de autenticacion emitido por Auth0. Contiene tenant_id, user_id, email, permissions.

## K

**keyword-only**
:   Convencion Python donde `schema_name` siempre se pasa como argumento nombrado: `schema_name=schema_name`.

## L

**LangGraph**
:   Framework de LangChain para agentes IA con grafos de estado. Usado en GDI-AgenteLANG.

## M

**MCP (Model Context Protocol)**
:   Protocolo de Anthropic para conectar modelos de IA con herramientas. GDI-Backend expone un MCP Server.

**Multi-Tenant**
:   Arquitectura donde cada municipio tiene su propio schema PostgreSQL. El schema se determina por el `tenant_id` del JWT.

## N

**Nota**
:   Comunicacion interna entre funcionarios. Puede tener destinatarios y adjuntos.

**Numeracion**
:   Sistema que asigna numeros secuenciales a documentos y expedientes por tipo, sector y anio. Usa advisory lock.

## O

**OpenRouter**
:   Servicio que provee acceso a multiples modelos LLM (Claude, GPT, etc.) via una API unificada.

## P

**PAdES**
:   PDF Advanced Electronic Signatures. Estandar de firma digital para PDFs usado por GDI-Notary.

**pgvector**
:   Extension de PostgreSQL para busqueda por similaridad vectorial. Usado para RAG en GDI-AgenteLANG.

**Providencia**
:   Documento interno que se genera automaticamente en ciertos flujos (asignacion, transferencia de expedientes).

**PV (Providencia)**
:   Abreviatura de Providencia. Documento auto-generado.

**pyHanko**
:   Libreria Python para firma digital PAdES. Core del servicio Notary.

## R

**Railway**
:   Plataforma de deployment donde corren todos los servicios de GDI en produccion.

**Rank**
:   Nivel jerarquico de un funcionario dentro del organigrama. Determina que tipos de documentos puede firmar.

**Resolucion**
:   Tipo de documento oficial. Requiere firma del funcionario con rank adecuado.

## S

**Schema (PostgreSQL)**
:   Namespace de base de datos. `public` contiene tablas globales (tenants, users). Cada municipio tiene su propio schema (ej: `200_muni`).

**Sector**
:   Unidad organizacional dentro de un municipio (Direccion, Departamento, etc.).

**shadcn/ui**
:   Libreria de componentes React usada en el Frontend. Basada en Radix UI + Tailwind.

## T

**TenantMiddleware**
:   Middleware FastAPI que extrae `tenant_id` del JWT y lo inyecta en el request state.

**Titular**
:   Funcionario con el rank mas alto de un sector. Puede firmar Decretos, Resoluciones y Disposiciones.

## V

**Visual Signature**
:   Representacion grafica de la firma en el PDF. Incluye nombre, cargo, sector y fecha.
