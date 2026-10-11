# 01 — Reglas Técnicas
_Fecha: 2026-10-11_

Reglas etiquetadas: **[REGLA]** comportamiento obligatorio · **[CONVENCION]** estilo/patrón acordado · **[RESTRICCION-MODULO]** restricción de modificación en módulo específico.

---

## Código Python

[CONVENCION] **Nomenclatura:** `snake_case` para funciones y variables; `PascalCase` para clases. Fuente: `CLAUDE.md`.

[CONVENCION] **Idioma:** identificadores, comentarios y nombres de campo JSON en español canónico (`color_borde`, `grosor`, `alineacion_h`). No mezclar inglés en el mismo objeto JSON. Los motores leen español con fallback inglés. Fuente: `CONTEXT.md`, `CONTEXT_maketator_web.md §10`.

[CONVENCION] **Logging:** nunca `print()`. Siempre `logger = logging.getLogger(__name__)`. En Windows, `print()` con caracteres no-ASCII puede fallar con cp1252. Fuente: `CLAUDE.md`.

[CONVENCION] **Callbacks Dash:** siempre `prevent_initial_call=True`. Usar `allow_duplicate=True` cuando múltiples callbacks escriben el mismo `Output`. Fuente: `CONTEXT.md`.

[REGLA] **Callbacks ligeros:** delegar lógica pesada a `utils/`. No acumular estado ni ejecutar cálculos costosos dentro del callback. Fuente: `CLAUDE.md`.

---

## Motor PDF

[RESTRICCION-MODULO] `engines/html_engine.py` y `utils/report_engine.py` **no se parchean localmente**. Son trasplantados de Maketator. Cualquier cambio debe coordinarse con Maketator y trasplantarse de nuevo. Fuente: `CLAUDE.md`.

[REGLA] El motor productivo es HTML/Playwright. ReportLab está retirado. Fuente: `memory/project_v2_motor_html.md`, commit `92c63be9`.

[REGLA] Las plantillas de informe se depositan en `biblioteca_plantillas/html/{nombre}/{nombre}.json`. No se crean ni editan en IncliData. Fuente: `CLAUDE.md`, `CONTRATO_motor_maketador.md §2.1`.

[REGLA] La generación PDF de `pages/graficar.py` pasa siempre por `utils/report_engine.generate_report_pdf_from_state()`. No se llama a `pdf_generator.py` directamente. Fuente: commit `1b269f32`.

[REGLA] **Dos funciones públicas en `report_engine.py`:** `generate_report_pdf(nombre_plantilla, context, output_path)` para plantillas en disco; `generate_report_pdf_from_state(editor_state, context, output_path)` para el estado en memoria del modal de Graficar. No llamar a `html_engine` directamente. Fuente: `utils/report_engine.py:79,104`.

[REGLA] **Namespace de scripts:** el campo `script` en plantillas lleva prefijo `html/` (ej. `"html/grafico_inclinometro_v2.py"`). La resolución de rutas usa siempre `script_registry` o `Path("biblioteca_graficos") / nombre_canonico`. El patrón legacy `{carpeta}/{carpeta}.py` no es compatible con el motor HTML.

[REGLA] **Las tablas Maketator** no llevan campo `script`: usan `grupo` + funciones de celda. La UI no debe construir ni exigir paths de script para elementos de tipo tabla.

[REGLA] **Sin validación Pydantic en el circuito vivo.** `html_engine` recibe la plantilla como dict crudo. No existe `Plantilla.model_validate()` en el camino activo (está en `info/legacy/`). Los campos nuevos se definen en Maketator y viajan por JSON sin schema.

[RESTRICCION-MODULO] **Mapa de minas del motor heredado** — plantillas de Maketator que usen estos tipos fallarán de forma predecible en IncliData (no son bugs a parchear localmente):
- `tipo=sinoptico`: `_elem_sinoptico` requiere `context["_server"]` para etiquetas de sensor (IncliData nunca lo provee → degrada silenciosamente). Además `utils/sinoptico_componentes.py` no existe en IncliData → `ImportError` si la plantilla usa componentes SVG.
- `tipo=mapa` (folium): `_elem_mapa_folium` degrada a placeholder si `folium`/`pyproj` no están instalados (no están en `requirements.txt` de IncliData).
- Rama BD: cubierta por stubs de `models/`. IncliData nunca envía `context["server_id"]`.

[REGLA] **Contrato de plantilla JSON:** el campo `secciones` viaja round-trip completo — no modificar su contenido. La geometría de todos los elementos se expresa en centímetros (`x`, `y`, `ancho`, `alto`). Fuente: `_migracion/CONTRATO_motor_maketador.md §2.1`, `04_DATA_MODEL.md §4`.

---

## Store `graficar-tubo`

[REGLA] Las claves raíz del store `graficar-tubo` son **exclusivamente** timestamps de campaña en ISO 8601 más las claves reservadas `info` y `umbrales`. Fuente: `CLAUDE.md`.

[REGLA] Cualquier metadato nuevo (sensor_id, nombre, etc.) va **dentro de `info{}`**, nunca como clave raíz. Fuente: commit `0e8e1251`.

[REGLA] **El store es una proyección con pérdida del JSON original.** Descarta `abs_dev_a/b`, `dev_a/b`, `raw[]` y otros campos. Nunca usarlo como fuente de datos para el motor de render: el motor lee siempre de `json_inclis/{sensor_id}.json` (JSON completo).

---

## Puente json_inclis

[REGLA] Al subir un archivo de sensor en Graficar, `pages/graficar.py` escribe el JSON completo en `json_inclis/{sensor_id}.json`. El motor HTML lo lee desde disco en tiempo de render. Fuente: `CLAUDE.md`, commit `1b269f32`.

[RESTRICCION-MODULO] `json_inclis/` está excluido del reloader werkzeug (ver arranque dual). Los archivos `.json` se generan en runtime y no se versionan (`.gitignore: json_inclis/*.json`). Fuente: `CLAUDE.md`.

[REGLA] **`sensor_id` canónico = `info.nom_sensor` del JSON del sensor; fallback al stem del nombre de archivo subido** (`graficar.py:357`). Se persiste en `info._sensor_id` del store y es el único valor válido para `context["sensor"]`.

[REGLA] **Las claves reservadas de contexto** (`sensor`, `sensores`, `fecha_*`, `ultimas_camp`) viajan solo por el dict `context`. La sustitución `$CURRENT` en parámetros de plantilla resuelve desde `context["sensor"]` vía `_CURRENT_TOKEN_MAP` (`html_engine.py:62`); `$CURRENT_fecha_*` y `$CURRENT_ultimas_camp` resuelven análogamente. Las claves raíz del store no se pasan al motor directamente. `context["sensores_1"]` es un mecanismo distinto: se usa en la iteración multi-sensor de tablas (`html_engine.py:1690`) y en `_normalize_context()` para inyectar el alias `nombre_sensor`; IncliData no lo establece en el contexto PDF (`graficar.py:2041-2048`).

---

## Arranque

[REGLA] **Estable** (por defecto): `python app.py` → `debug=False, use_reloader=False`.

[REGLA] **Dev**: `INCLIDATA_DEBUG=1 python app.py` → `debug=True, dev_tools_hot_reload=False, exclude_patterns=[...]`.

[REGLA] `exclude_patterns` es obligatorio en modo dev. Sin él, el watcher de werkzeug reinicia el servidor cuando la app escribe `json_inclis/*.json` o `_assets/registry.json` durante un callback en vuelo, matando la petición. Fuente: `CLAUDE.md`, `CONTEXT.md §arranque dual`.

[REGLA] **`exclude_patterns` es extensible.** Cualquier nueva escritura de archivo en runtime dentro del árbol del proyecto (logs, PDFs temporales, registros) debe añadirse a la lista en `app.py`. Lista actual: `json_inclis/*`, `*.log`, `*.pdf`, `registry.json`.

---

## Módulos stub

[RESTRICCION-MODULO] `models/server.py` (`class Server`) y `models/database.py` (`get_session()`) son **stubs que lanzan `RuntimeError`** si se invocan. Existen solo para satisfacer imports condicionales de `html_engine.py` en ramas de código que IncliData nunca ejecuta (requieren `context["server_id"]`, que IncliData no provee). No añadir ORM sin coordinación con Maketator. Fuente: docstrings de ambos archivos, `engines/html_engine.py:3304-3305`.

---

## UI / Estilo visual

[CONVENCION] **Tablas AgGrid:** siempre `className='ag-theme-quartz'`. Nunca `ag-theme-alpine` ni `ag-theme-alpine-dark`. El tema se adapta a modo oscuro vía CSS variables. Fuente: `Guia_estilo.md §6`.

[CONVENCION] **Colores inline:** siempre `var(--id-*)`. Nunca hex fijo en estilos inline para garantizar compatibilidad con modo oscuro. Fuente: `Guia_estilo.md §8`.

[CONVENCION] **Iconos:** DashIconify + set Lucide. En botones: `width=14`; en zonas de upload: `width=20`; standalone/decorativo: `width=24`. Fuente: `Guia_estilo.md §4`.

[CONVENCION] **Componentes UI:** Dash 4.x + Dash Mantine Components v2 + Dash Bootstrap Components. Fuente: `CLAUDE.md`.

[CONVENCION] **Paths y ficheros:** siempre `pathlib.Path`; `encoding="utf-8"` explícito en todos los `open()`.

[CONVENCION] **`dcc.Store` en layouts de página, no en `app.py`.** Un store nuevo va en el layout de la página que lo usa. Patrón: `pages/graficar_layout.py:27`. Fuente: `CONTEXT.md §9`.

[CONVENCION] **Configuración de logging:** handlers (consola + `debug_incli.log`) solo en el bloque `if __name__ == "__main__":` de `app.py`. No añadir `logging.basicConfig` ni `addHandler` a nivel de módulo. Las trazas `[PDF-TRACE]` del circuito PDF se emiten en nivel `DEBUG` y no se eliminan.

[CONVENCION] **DMC v2 — no inventar props.** En caso de duda, introspección: `python -c "import inspect, dash_mantine_components as dmc; print(sorted(p.name for p in inspect.signature(dmc.Componente.__init__).parameters.values()))"`. `dmc.TransferList` no existe en DMC v2.

[CONVENCION] **Mismo nombre ≠ mismo contrato.** Símbolos como `cargar_plantilla`, `generate_report_pdf_from_state` o `resolve_template` existen en Maketator con el mismo nombre pero firma y semántica distintas. Verificar retorno y efectos antes de asumir compatibilidad. Fuente: `CONTEXT_maketator_web.md §convenciones`.

---

## Assets de plantillas

[RESTRICCION-MODULO] `biblioteca_plantillas/_assets/registry.json` no se edita manualmente. Es gestionado por `utils/asset_manager.py`. Fuente: `CONTEXT.md`.

---

## Control de versiones

[REGLA] El usuario controla los commits y el push. Claude Code no hace commit sin instrucción explícita del usuario.

---

## Archivos protegidos

[RESTRICCION-MODULO] **Intocables — coordinación con Maketator obligatoria** (parchear localmente crea divergencia silenciosa): `utils/script_registry.py`, `biblioteca_graficos/html/*`, `biblioteca_tablas/funciones/*`, `biblioteca_grupos/*`. Cualquier cambio se hace en Maketator y se re-trasplanta. Fuente: instrucciones asesor v1.0 §3.

[RESTRICCION-MODULO] **`utils/template_service.py`** — contiene `cargar_plantilla` (retorno dict crudo, sin Pydantic) y `_encontrar_json_plantilla` (trasplantada de Maketator). Consumida directamente por `report_engine`. No modificar sin verificar contrato de retorno en Maketator.

[RESTRICCION-MODULO] **`utils/asset_manager.py`** — par de `biblioteca_plantillas/_assets/registry.json`. No modificar uno sin el otro; toda escritura va por `asset_manager`.

[RESTRICCION-MODULO] **`_migracion/CONTRATO_motor_maketador.md`** — documento normativo del contrato de integración. Modificar solo cuando el contrato real cambia en Maketator; documentar la versión del cambio.

[REGLA] **`info/legacy/` no se reimporta en código vivo.** Es solo lectura histórica. Ningún módulo activo debe hacer `from info.legacy import ...` ni `import info.legacy`.

[CONVENCION] **Archivos de alto impacto:** `pages/graficar.py`, `pages/graficar_layout.py` y `pages/correcciones.py` concentran el circuito PDF, el puente de datos `json_inclis` y las correcciones de sensor. Antes de modificar cualquiera de ellos, revisar blast-radius con CodeGraph.
