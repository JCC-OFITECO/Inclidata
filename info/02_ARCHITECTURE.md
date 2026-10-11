# 02 — Arquitectura
_Fecha: 2026-10-11_

---

## Estructura de alto nivel

IncliData es un monolito multi-página Dash con capas bien definidas:

```
pages/          Callbacks y layouts por dominio (importar → corregir → graficar → imprimir)
utils/          Lógica de negocio y servicios (parsers, algoritmos, motor PDF)
engines/        Motor de renderizado HTML/Playwright (trasplantado de Maketator)
models/         Stubs de compatibilidad con html_engine
biblioteca_*/   Contenido dinámico: scripts de gráficos, tablas y plantillas JSON
data/           JSONs de sensores (datos de campo)
json_inclis/    Puente en disco sensor → motor HTML (escritos en runtime, no versionados)
assets/         CSS global
tests/          Suite pytest
_migracion/     Contratos de integración con Maketator
info/           Documentación estructurada del proyecto (este directorio)
info/legacy/    Código archivado: motor ReportLab, editor, funciones legacy
```

---

## Responsabilidades por módulo

### `app.py`
Punto de entrada único. Inicializa Dash, define el layout de sidebar + contenido, registra los 5 items de navegación y delega el registro de callbacks a cada página con `modulo.register_callbacks(app)`. Gestiona el modo oscuro (switch + CSS variables). Arranca en modo estable o dev según `INCLIDATA_DEBUG`.

### `pages/`

| Archivo | Responsabilidad |
|---|---|
| `info.py` | Página de bienvenida; muestra datos del sensor cargado |
| `importar.py` | Wizard de 5 pasos: selección base → configurar importador → subir raw → procesar → guardar |
| `correcciones.py` | Callbacks de corrección bias (polilínea) y spikes (puntual); escribe `spikes.json` |
| `correcciones_layout.py` | Layout de la página Correcciones (separado de callbacks) |
| `graficar.py` | Callbacks de visualización interactiva + modal de informe PDF. Escribe `json_inclis/{sensor_id}.json` al subir sensor. Llama `report_engine.generate_report_pdf_from_state()` para generar PDF. |
| `graficar_layout.py` | Layout de la página Graficar |
| `importar_umbrales.py` | Importación de umbrales desde Excel |
| `configuraciones.py` | Stub vacío (futuro) |
| `spikes.json` | Configuración de spikes; ruta relativa intencional (usada por `correcciones.py`) |

### `utils/`

| Archivo | Responsabilidad |
|---|---|
| `report_engine.py` | **Fachada del motor PDF.** `generate_report_pdf_from_state(editor_state, context, output_path)` es el punto de entrada desde Graficar. Selecciona motor por `editor_state["configuracion"]["engine"]` (por defecto "html"). |
| `template_service.py` | Carga de plantillas desde `biblioteca_plantillas/html/`. Listado de scripts disponibles. Las funciones de guardado/edición del editor anterior fueron eliminadas con éste. |
| `script_registry.py` | Descubrimiento dinámico de scripts de gráficos con `@register_script`. Scan al arrancar; no requiere registro manual. |
| `asset_manager.py` | Gestión de imágenes de plantillas. `get_asset_path()` / `get_asset_data_uri()` para lectura en render. Las funciones de escritura (`register_asset`, `save_registry`) se conservan por compatibilidad aunque el editor fue eliminado. |
| `funciones_importar.py` | Parsers por fabricante: RST, Sisgeo, Soil_dux |
| `funciones_graficar.py` | Construcción de figuras Plotly: perfil deformada, evolución temporal, polar, 3D |
| `funciones_correcciones.py` | Algoritmos de bias (polilínea por tramos) y detección/corrección de spikes |
| `funciones_comunes.py` | Utilidades compartidas, evaluación de umbrales |
| `dev_logging.py` | Monkey-patch de logging de callbacks; activo con `DEBUG_CALLBACKS=1` |
| `cell_function_registry.py` | Carga dinámica de funciones de celda de tablas en plantillas |
| `rutas.py` | Centralización de rutas del sistema de archivos |

### `engines/`

| Archivo | Responsabilidad |
|---|---|
| `html_engine.py` | Motor principal trasplantado de Maketator. Convierte plantilla JSON → HTML (posiciones CSS en cm) → PDF vía Playwright/Chromium. **No parchear localmente.** |
| `base.py` | Contrato abstracto `BaseReportEngine` (herencia del trasplante; sin uso funcional en IncliData) |

### `models/`

| Archivo | Contenido |
|---|---|
| `server.py` | `class Server` — stub que lanza `RuntimeError`. Satisface import condicional de `html_engine.py:3304`. |
| `database.py` | `get_session()` — stub que lanza `RuntimeError`. Satisface import condicional de `html_engine.py:3305`. |

Ambos son importados por `html_engine.py` solo en la rama que gestiona conexiones SQL remotas (`context["server_id"]`), rama que IncliData nunca activa.

### `biblioteca_graficos/html/`
Scripts Plotly para generación de gráficos en PDF. Convención: `generate(params, figsize) → str (HTML)` + `PARAMETER_METADATA`. El `ScriptRegistry` los descubre automáticamente al arrancar. Namespace `html/` en rutas de plantillas.

### `biblioteca_tablas/funciones/`
Funciones de celda para tablas de tipo "autorrelleno" en plantillas. Convención: `evaluate(params, data, context) → float|str` + `CELL_FUNCTION_METADATA`. `cell_function_registry.py` hace scan dinámico.

Incluye `columna_incli_json.py`: función especial que lee datos de sensor desde `json_inclis/{sensor}.json` con caché en memoria (`_json_cache`).

### `biblioteca_plantillas/html/`
Plantillas JSON creadas en Maketator. Una carpeta por plantilla: `{nombre}/{nombre}.json`. Los assets (imágenes) se gestionan vía `_assets/registry.json`.

### `data/`
JSONs de sensores inclinométricos (datos de campo). Formato canónico en `_migracion/CONTRATO_motor_maketador.md §2.3`. No se modifican por el motor PDF; son fuente de los gráficos.

### `json_inclis/`
Archivos JSON de sensor escritos en runtime por `pages/graficar.py` al subir un sensor. El motor HTML los lee para renderizar gráficos y tablas. No se versionan (`.gitignore`). La carpeta se preserva vía `.gitkeep`.

---

## Flujo principal: importación → corrección → visualización → PDF

```
1. Usuario sube archivo de fabricante (RST/Sisgeo/Soil_dux)
   └── pages/importar.py → utils/funciones_importar.py
       └── Wizard 5 pasos: parse raw → calcular incrementos → validar cronología → guardar JSON en data/

2. Usuario corrige datos (opcional)
   └── pages/correcciones.py → utils/funciones_correcciones.py
       ├── Bias: polilínea de corrección por tramos de profundidad (Bias_1_A/B, Bias_2_A/B)
       └── Spikes: corrección puntual de lecturas a0/a180/b0/b180 por profundidad

3. Usuario visualiza en Graficar
   └── pages/graficar.py
       ├── Al subir sensor: escribe json_inclis/{sensor_id}.json + actualiza store graficar-tubo
       └── Muestra gráficas: perfil deformada, evolución temporal, polar, 3D (Plotly)

4. Usuario genera PDF
   └── pages/graficar.py (modal de informe)
       ├── Selecciona plantilla + parámetros $CURRENT
       └── generate_report_pdf_from_state(editor_state, context, output_path)
           └── engines/html_engine.py → Playwright/Chromium → PDF
               ├── Gráficos: biblioteca_graficos/html/{script}.py (Plotly → HTML)
               └── Tablas: biblioteca_tablas/funciones/{func}.py + json_inclis/{sensor}.json
```

---

## Flujo del motor PDF

```
report_engine.generate_report_pdf_from_state(editor_state, context, output_path)
  │
  ├─ Lee engines/html_engine.py como motor (campo engine:"html" en plantilla)
  │
  └─ html_engine.render(plantilla, context)
       ├── Por cada página → por cada elemento:
       │    tipo=grafico  → _elem_grafico() → script_registry → biblioteca_graficos/html/{script}.generate()
       │    tipo=tabla    → _elem_tabla()   → cell_function_registry → biblioteca_tablas/funciones/{func}.evaluate()
       │    tipo=texto    → _elem_texto()   → _resolve_tokens($CURRENT*)
       │    tipo=imagen   → _elem_imagen() / _elem_imagen_dinamica()
       │    tipo=rect     → _elem_rect()
       │    tipo=linea    → _elem_linea()
       │
       └─ HTML string → Playwright headless → PDF
```

---

## Puntos de extensión

| Qué añadir | Dónde | Patrón |
|---|---|---|
| Nuevo script de gráfico | `biblioteca_graficos/html/{namespace}/{script}.py` | `@register_script(metadata)` + `def generate(params, figsize) -> str` |
| Nueva función de celda de tabla | `biblioteca_tablas/funciones/{funcion}.py` | `CELL_FUNCTION_METADATA` + `def evaluate(params, data, context)` |
| Nueva página Dash | `pages/{nueva_pagina}.py` + `app.py` | Añadir en `NAV_ITEMS` y `routes` dict en `app.py` |
| Nueva plantilla de informe | `biblioteca_plantillas/html/{nombre}/{nombre}.json` | Creada en Maketator; formato en `CONTRATO_motor_maketador.md §2.1` |

---

## Carpetas residuales (sin modificar en esta fase)

| Carpeta / Archivo | Estado | Detalle |
|---|---|---|
| `kk/` | Rastreado por git | Contiene `grafico_inclinometro_v1.py` (script de gráfico antiguo). No está en `.gitignore`. Ver `06_CURRENT_STATUS.md`. |
| `temp_lovable_reference/` | Rastreado por git | Referencia temporal del prototipo Lovable. No está en `.gitignore`. Ver `06_CURRENT_STATUS.md`. |
| `info/legacy/` | Archivado | Motor ReportLab, template_models.py, funciones editor, graficar_vista_previa_legacy.py. No se importa desde código vivo. |
