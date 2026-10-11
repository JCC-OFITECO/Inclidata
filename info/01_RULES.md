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

---

## Store `graficar-tubo`

[REGLA] Las claves raíz del store `graficar-tubo` son **exclusivamente** timestamps de campaña en ISO 8601 más las claves reservadas `info` y `umbrales`. Fuente: `CLAUDE.md`.

[REGLA] Cualquier metadato nuevo (sensor_id, nombre, etc.) va **dentro de `info{}`**, nunca como clave raíz. Fuente: commit `0e8e1251`.

---

## Puente json_inclis

[REGLA] Al subir un archivo de sensor en Graficar, `pages/graficar.py` escribe el JSON completo en `json_inclis/{sensor_id}.json`. El motor HTML lo lee desde disco en tiempo de render. Fuente: `CLAUDE.md`, commit `1b269f32`.

[RESTRICCION-MODULO] `json_inclis/` está excluido del reloader werkzeug (ver arranque dual). Los archivos `.json` se generan en runtime y no se versionan (`.gitignore: json_inclis/*.json`). Fuente: `CLAUDE.md`.

---

## Arranque

[REGLA] **Estable** (por defecto): `python app.py` → `debug=False, use_reloader=False`.

[REGLA] **Dev**: `INCLIDATA_DEBUG=1 python app.py` → `debug=True, dev_tools_hot_reload=False, exclude_patterns=[...]`.

[REGLA] `exclude_patterns` es obligatorio en modo dev. Sin él, el watcher de werkzeug reinicia el servidor cuando la app escribe `json_inclis/*.json` o `_assets/registry.json` durante un callback en vuelo, matando la petición. Fuente: `CLAUDE.md`, `CONTEXT.md §arranque dual`.

---

## Módulos stub

[RESTRICCION-MODULO] `models/server.py` (`class Server`) y `models/database.py` (`get_session()`) son **stubs que lanzan `RuntimeError`** si se invocan. Existen solo para satisfacer imports condicionales de `html_engine.py` en ramas de código que IncliData nunca ejecuta (requieren `context["server_id"]`, que IncliData no provee). No añadir ORM sin coordinación con Maketator. Fuente: docstrings de ambos archivos, `engines/html_engine.py:3304-3305`.

---

## UI / Estilo visual

[CONVENCION] **Tablas AgGrid:** siempre `className='ag-theme-quartz'`. Nunca `ag-theme-alpine` ni `ag-theme-alpine-dark`. El tema se adapta a modo oscuro vía CSS variables. Fuente: `Guia_estilo.md §6`.

[CONVENCION] **Colores inline:** siempre `var(--id-*)`. Nunca hex fijo en estilos inline para garantizar compatibilidad con modo oscuro. Fuente: `Guia_estilo.md §8`.

[CONVENCION] **Iconos:** DashIconify + set Lucide. En botones: `width=14`; en zonas de upload: `width=20`; standalone/decorativo: `width=24`. Fuente: `Guia_estilo.md §4`.

[CONVENCION] **Componentes UI:** Dash 4.x + Dash Mantine Components v2 + Dash Bootstrap Components. Fuente: `CLAUDE.md`.

---

## Assets de plantillas

[RESTRICCION-MODULO] `biblioteca_plantillas/_assets/registry.json` no se edita manualmente. Es gestionado por `utils/asset_manager.py`. Fuente: `CONTEXT.md`.

---

## Control de versiones

[REGLA] El usuario controla los commits y el push. Claude Code no hace commit sin instrucción explícita del usuario.
