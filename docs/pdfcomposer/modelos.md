# Modelos Pydantic

Los modelos Pydantic se definen en `app/models/pdf_models.py` y validan todos los datos
de entrada antes de procesarlos. Hay 5 modelos, uno por tipo de documento.

## PDFRequest

Modelo base para preview y documento final.

```python
class PDFRequest(BaseModel):
    urlLogo: Optional[str] = Field(
        None, description="URL publica de la imagen del logo."
    )
    NameAcronyType: str = Field(
        ..., description="Acronimo del nombre del tipo (ej. 'GDI', 'ACME')."
    )
    TypeDocument: str = Field(
        ..., description="Tipo de documento (ej. 'INFORME DE PRUEBA')."
    )
    Reference: str = Field(
        ..., description="Referencia del documento (ej. 'REF-001-2025')."
    )
    Text: Dict[str, Any] = Field(
        ..., description="Contenido principal del documento en formato JSON."
    )
```

!!! info "Campo `Text`"
    El campo `Text` se recibe como string JSON en el Form Data y se parsea
    manualmente con `json.loads()` en el endpoint antes de pasarlo al modelo.
    Debe contener la clave `"html"` con el contenido HTML del documento:
    ```json
    {"html": "<p>Contenido del documento.</p>"}
    ```

**Usado por:** `/preview-pdf/`, `/generate-pdf/`

---

## CaseRequest

Modelo para caratulas de expediente (CAEX).

```python
class CaseRequest(BaseModel):
    urlLogo: Optional[str] = Field(None, description="URL publica del logo.")
    NameAcronyType: str = Field(..., description="Acronimo del tipo.")
    document_type: str = Field(..., description="Tipo de documento para la caratula.")
    reference: str = Field(..., description="Referencia del documento.")
    case_number: str = Field(..., description="Numero de expediente.")
    acrony_case_type: str = Field(..., description="Acronimo del tipo de expediente.")
    case_type: str = Field(..., description="Tipo de expediente.")
    case_motive: str = Field(..., description="Motivo del expediente.")
    initiating_division: str = Field(..., description="Reparticion iniciadora.")
    creator: str = Field(..., description="Nombre del caratulador.")
```

**Usado por:** `/create-case/`

---

## MoveRequest

Modelo para pases de vista entre areas.

```python
class MoveRequest(BaseModel):
    urlLogo: Optional[str] = Field(None, description="URL publica del logo.")
    NameAcronyType: str = Field(..., description="Acronimo del tipo.")
    document_type: str = Field(..., description="Tipo de documento.")
    reference: str = Field(..., description="Referencia del documento.")
    tipo_movimiento: str = Field(..., description="Tipo de movimiento.")
    area_requiriente: str = Field(..., description="Area requiriente (DE).")
    area_receptora: str = Field(..., description="Area receptora (A).")
    motivo: str = Field(..., description="Motivo del movimiento.")
```

**Usado por:** `/move/`

---

## ImportRequest

Modelo para pagina informativa de documentos importados.

```python
class ImportRequest(BaseModel):
    urlLogo: Optional[str] = Field(None, description="URL publica del logo.")
    NameAcronyType: str = Field(..., description="Acronimo del tipo.")
    document_type: str = Field(..., description="Tipo de documento.")
    reference: str = Field(..., description="Referencia del documento.")
    cantidad_paginas: int = Field(
        ..., description="Cantidad de paginas del PDF importado."
    )
```

**Usado por:** `/import/`

---

## NoteRequest

Modelo para notas internas.

```python
class NoteRequest(BaseModel):
    urlLogo: Optional[str] = Field(None, description="URL publica del logo.")
    NameAcronyType: str = Field(..., description="Acronimo del tipo (ej. 'GDI').")
    document_type: str = Field(..., description="Tipo de documento (ej. 'NOTA').")
    reference: str = Field(..., description="Referencia del documento.")
    para: Optional[str] = Field(
        None, description="Destinatario principal (PARA). Si vacio, no aparece."
    )
    cc: Optional[str] = Field(
        None, description="Destinatarios en copia (CC). Si vacio, no aparece."
    )
    Text: Dict[str, Any] = Field(
        ..., description="Contenido HTML del documento."
    )
```

**Usado por:** `/note/`, `/note-preview/`

---

## Mapa de modelos por endpoint

| Endpoint | Modelo | Campos unicos |
|----------|--------|---------------|
| `/preview-pdf/` | `PDFRequest` | `Text` (JSON con `html`) |
| `/generate-pdf/` | `PDFRequest` | + `attachment_file` (File, no en modelo) |
| `/create-case/` | `CaseRequest` | `case_number`, `case_type`, `creator` |
| `/move/` | `MoveRequest` | `area_requiriente`, `area_receptora`, `motivo` |
| `/import/` | `ImportRequest` | `cantidad_paginas` (generado internamente) |
| `/note-preview/` | `NoteRequest` | `para`, `cc` |
| `/note/` | `NoteRequest` | `para`, `cc` |
