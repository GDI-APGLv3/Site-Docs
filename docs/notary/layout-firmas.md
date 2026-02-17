# Layout de firmas (2 columnas)

El modulo `app/layout.py` implementa el algoritmo de posicionamiento automatico de firmas
en documentos PDF, usando un layout de 2 columnas.

## Diagrama del layout

```
                    ULTIMA PAGINA
+-------------------------------------+
|         Contenido documento         |
|              "end-text"             |  <-- Punto de referencia
|                                     |
|  [Firma 1]        [Firma 2]        |  X=50 (impar) | X=270 (par)
|  [Firma 3]        [Firma 4]        |
|  [Firma 5]        [Firma 6]        |  Crece hacia abajo
|  ...              ...              |
|                                     |
| ---- MIN_SIGNATURE_Y = 100 -----  |  <-- Limite inferior
+-------------------------------------+
```

## Constantes de configuracion

Definidas en `app/config.py`:

| Constante | Valor | Descripcion |
|-----------|-------|-------------|
| `FIRST_SIGNATURE_X` | 50 | Columna izquierda (firmas impares) |
| `SECOND_SIGNATURE_X` | 270 | Columna derecha (firmas pares) |
| `SIGNATURE_WIDTH` | 200 | Ancho de cada firma (pts) |
| `SIGNATURE_HEIGHT` | 80 | Alto de cada firma (pts) |
| `ROW_SPACING` | 20 | Espaciado entre filas (pts) |
| `SIGNATURE_OFFSET_BELOW` | 100 | Distancia debajo del "end-text" (pts) |
| `MIN_SIGNATURE_Y` | 100 | Posicion Y minima (margen inferior) |
| `PAGE_WIDTH` | 612 | Ancho pagina Letter (pts) |
| `PAGE_HEIGHT` | 792 | Alto pagina Letter (pts) |

## Sistemas de coordenadas

!!! warning "Dos sistemas diferentes"
    PyMuPDF y ReportLab usan sistemas de coordenadas opuestos:

    - **PyMuPDF**: Origen `(0,0)` en esquina **superior izquierda**, Y crece hacia abajo
    - **ReportLab**: Origen `(0,0)` en esquina **inferior izquierda**, Y crece hacia arriba

    La conversion es: `reportlab_y = page_height - pymupdf_y`

## Algoritmo paso a paso

### 1. Buscar "end-text"

```python
def find_end_text_positions(pdf_content: bytes) -> List[Tuple[float, float]]:
    doc = fitz.open(stream=pdf_content, filetype="pdf")
    last_page = doc[-1]
    text_instances = last_page.search_for("end-text")
    positions = [(rect.x0, rect.y0) for rect in text_instances]
    return positions
```

Solo busca en la **ultima pagina** del documento. Si no encuentra "end-text", el proceso falla con `LayoutError`.

### 2. Calcular posicion Y de la primera firma

```python
def get_first_signature_y_position(pdf_content: bytes) -> Optional[float]:
    # Encontrar "end-text" en coordenadas PyMuPDF
    text_pymupdf_y = text_rect.y0

    # Mover 100pts hacia abajo (en PyMuPDF)
    signature_pymupdf_y = text_pymupdf_y + SIGNATURE_OFFSET_BELOW

    # Convertir a coordenadas ReportLab
    reportlab_y = page_height - signature_pymupdf_y

    return reportlab_y
```

### 3. Calcular posicion de la N-esima firma

```python
def calculate_signature_position(
    existing_signatures_count: int = 0,
    pdf_content: bytes = None
) -> Tuple[float, float]:
    # Posicion Y base (primera firma)
    first_signature_y = get_first_signature_y_position(pdf_content)

    # Numero de la nueva firma
    signature_number = existing_signatures_count + 1

    # Columna X: impares a la izquierda, pares a la derecha
    if signature_number % 2 == 1:
        x = FIRST_SIGNATURE_X   # 50
    else:
        x = SECOND_SIGNATURE_X  # 270

    # Fila Y: cada 2 firmas baja una fila
    row = (signature_number - 1) // 2
    y = first_signature_y - (row * (SIGNATURE_HEIGHT + ROW_SPACING))

    # Validar espacio
    if not validate_signature_space(x, y):
        raise LayoutError("FULLPAGE")

    return (x, y)
```

### 4. Ejemplo numerico

Con `end-text` encontrado en PyMuPDF `y=500` y pagina Letter (`height=792`):

| Firma | Numero | Columna | X | Fila | Y (ReportLab) |
|-------|--------|---------|---|------|----------------|
| 1ra | 1 | Izquierda | 50 | 0 | 192 |
| 2da | 2 | Derecha | 270 | 0 | 192 |
| 3ra | 3 | Izquierda | 50 | 1 | 92 |
| 4ta | 4 | Derecha | 270 | 1 | 92 |
| 5ta | 5 | - | - | 2 | Error FULLPAGE |

Calculo primera firma: `page_height - (text_y + offset) = 792 - (500 + 100) = 192`

## Deteccion de firmas existentes

```python
def count_existing_signatures(pdf_content: bytes) -> int:
    # 1. Contar firmas por texto visual
    visual_count = page_text.count(SIGNATURE_DETECTION_TEXT)

    # Patrones de respaldo
    other_patterns = [
        "Firmado digitalmente por:",
        "Digitally signed by:",
        "FIRMA DIGITAL",
        "DIGITAL SIGNATURE"
    ]

    # 2. Contar firmas PAdES embebidas
    pades_count = count_pades_signatures(pdf_content)

    # Usar el mayor de ambos
    return max(visual_count, pades_count)
```

!!! info "Deteccion dual"
    El sistema detecta tanto firmas visuales (por patron de texto) como
    firmas PAdES (por metadatos embebidos). Usa el mayor para evitar
    superposicion en cualquier caso.

## Validacion de espacio

```python
def validate_signature_space(x: float, y: float) -> bool:
    # Margenes horizontales
    if x < SIGNATURE_MARGIN_LEFT:
        return False
    if x + SIGNATURE_WIDTH > PAGE_WIDTH - SIGNATURE_MARGIN_RIGHT:
        return False

    # Limite inferior (critico)
    if y - SIGNATURE_HEIGHT < MIN_SIGNATURE_Y:
        return False

    # Limite superior
    if y > PAGE_HEIGHT - SIGNATURE_MARGIN_TOP:
        return False

    return True
```

Si la validacion falla, se lanza `LayoutError("FULLPAGE")` indicando que no hay mas espacio
para firmas en la pagina.
