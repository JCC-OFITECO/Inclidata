# 05 — Decisiones de Diseño
_Fecha: 2026-10-11_

Registro de decisiones relevantes. Formato: **Estado** / **Decisión** / **Motivo** / **Consecuencia** / **Fuente**. Las decisiones inferidas (sin documento explícito de decisión) se marcan **[INFERENCIA]**.

---

## DEC-001 — Motor PDF HTML/Playwright como motor productivo único

**Estado:** Aprobada (2026-07-03)

**Decisión:** Migrar el motor de generación PDF de ReportLab/Matplotlib a HTML/Playwright, trasplantando `html_engine.py` de Maketator.

**Motivo:** Unificar el motor con Maketator; eliminar la dependencia de ReportLab; aprovechar Plotly (interactivo) también para gráficos PDF; HTML/CSS es más expresivo para maquetación que el canvas de ReportLab.

**Consecuencia:** Motor más potente y mantenido en Maketator. Requiere `playwright install chromium` post-instalación. `html_engine.py` no se parchea localmente (coordinación con Maketator). ReportLab eliminado de `requirements.txt`.

**Fuente:** Commits `177cb4ab` (trasplante inicial), `92c63be9` (consolidación); `memory/project_v2_motor_html.md`.

---

## DEC-002 — Eliminación del editor visual de plantillas

**Estado:** Aprobada (2026-07-03)

**Decisión:** Eliminar `pages/editor_plantilla.py`, `pages/editor_visual.py`, `dash_component_editor`, y todas las funciones de guardado/edición de plantillas de `utils/template_service.py`.

**Motivo:** Las plantillas se diseñan en Maketator (editor visual externo con React). Mantener un editor local en IncliData es redundante, costoso de mantener y duplica funcionalidad.

**Consecuencia:** IncliData pasa a ser exclusivamente viewer/corrector/printer. El código eliminado queda archivado en `info/legacy/`. La función `cargar_plantilla_para_editor()` (que inyectaba `chartScripts`/`tableScripts` en el estado del editor) también fue eliminada en el mismo commit.

**Fuente:** Commit `9af43705` ("poda de UI de creación/edición de plantillas"). Verificado con `git log -S "cargar_plantilla_para_editor"`: la función aparece en `1271ca04` (foto fija pre-poda) y desaparece en `9af43705`.

---

## DEC-003 — ReportLab archivado en `info/legacy/`

**Estado:** Aprobada (2026-07-03)

**Decisión:** Mover motor ReportLab (`motor_reportlab/`), scripts de gráficos Matplotlib (`graficos_reportlab/`), modelos Pydantic del editor (`template_models.py`), y funciones de informe legacy a `info/legacy/`. Retirar `reportlab` de `requirements.txt`.

**Motivo:** Eliminar deuda técnica y dependencias obsoletas tras migración a HTML. Conservar código como referencia histórica.

**Consecuencia:** Recuperación del estado anterior posible en `tag v1.0-pre-poda-reportlab` o `rama archive/full-editor-reportlab`. Sin camino de vuelta sin esa rama.

**Fuente:** Commit `92c63be9` ("consolidación migración motor HTML — ReportLab retirado, código muerto archivado").

---

## DEC-004 — Puente `json_inclis` como mecanismo de integración sensor→motor

**Estado:** Aprobada (2026-07-03)

**Decisión:** Al subir un archivo de sensor en Graficar, `pages/graficar.py` escribe el JSON completo en `json_inclis/{sensor_id}.json`. El motor HTML y las funciones de tabla leen ese archivo desde disco.

**Motivo:** El motor HTML necesita acceso a los datos del sensor en tiempo de render sin modificar `html_engine.py` (que no puede recibir los datos por parámetro sin cambiar el contrato). El fichero en disco es la solución más simple.

**Consecuencia:** El sensor debe haberse subido en la misma sesión para que el archivo exista. La regla de claves del store (solo timestamps + `info` + `umbrales`) y la ubicación de `_sensor_id` dentro de `info{}` son consecuencia de esta decisión (ver DEC-006).

**Fuente:** Commits `1b269f32` ("puente json_inclis en la subida"), `0e8e1251` ("_sensor_id movido a info{}").

---

## DEC-005 — Stubs de compatibilidad `models/server.py` y `models/database.py`

**Estado:** Aprobada

**Decisión:** Mantener `class Server` y `get_session()` como stubs que lanzan `RuntimeError`. Existen solo para satisfacer los imports condicionales de `html_engine.py` (líneas 3304-3305 y 3817-3818).

**Motivo:** `html_engine.py` se trasplanta completo desde Maketator (donde sí existe ORM). No es viable modificarlo para eliminar esos imports sin desincronizarlo de Maketator.

**Consecuencia:** IncliData no tiene ORM. La rama SQL de `html_engine` nunca se ejecuta porque `context` nunca lleva `server_id`. Si alguna plantilla intentara usar datos SQL, el error sería explícito.

**Fuente:** Docstrings de `models/server.py` y `models/database.py`; `CONTEXT.md §stubs`.

---

## DEC-006 — Regla de claves raíz del store `graficar-tubo`

**Estado:** Aprobada

**Decisión:** Las claves raíz del store `graficar-tubo` son exclusivamente timestamps ISO 8601 + `info` + `umbrales`. Cualquier metadato va dentro de `info{}`.

**Motivo:** Permite iterar campañas directamente sobre las claves del store sin filtrar metadatos. El `_sensor_id` estaba originalmente en la raíz y rompía este invariante.

**Consecuencia:** `_sensor_id` vive en `info._sensor_id`. Todo código que accede al sensor_id debe buscarlo ahí.

**Fuente:** Commit `0e8e1251` ("_sensor_id movido a info{} — las claves raíz del store deben ser solo timestamps de campaña"); `CLAUDE.md`.

---

## DEC-007 — Arranque dual con `exclude_patterns`

**Estado:** Aprobada

**Decisión:** Dos modos de arranque: estable (`debug=False, use_reloader=False`) y dev (`INCLIDATA_DEBUG=1`, con `exclude_patterns` protegiendo `json_inclis/` y `_assets/`).

**Motivo:** El watcher de werkzeug reiniciaba el servidor cuando la app escribía `json_inclis/*.json` durante un callback en vuelo, matando la petición con "server did not respond". Desactivar el reloader completo en dev haría perder el hot-reload de cambios de código.

**Consecuencia:** En modo dev: sin hot-reload (`dev_tools_hot_reload=False`), pero el reloader detecta cambios de código mientras protege los archivos de runtime.

**Fuente:** `CONTEXT.md §arranque dual`, `CLAUDE.md`.

---

## DEC-008 — Adopción del esquema de plantilla Maketator (HTML)

**Estado:** Aprobada

**Decisión:** IncliData adopta el esquema JSON de Maketator para plantillas: campo `engine:"html"`, rutas de scripts con namespace `html/`, tokens `$CURRENT*`, estructura `secciones` round-trip.

**Motivo:** El motor HTML se trasplanta completo desde Maketator; adoptar su esquema evita una capa de conversión. Las plantillas anteriores (esquema ReportLab) son incompatibles.

**Consecuencia:** Plantillas antiguas (ReportLab) no funcionan con el motor HTML sin conversión. El contrato vigente es `_migracion/CONTRATO_motor_maketador.md`.

**Fuente:** `CONTRATO_motor_maketador.md §2.1`; commit `177cb4ab`.

---

## DEC-009 — Consolidación del trigger PDF de `graficar.py` en `report_engine`

**Estado:** Aprobada

**Decisión:** El callback `generar_informe_pdf` en `pages/graficar.py` llama a `report_engine.generate_report_pdf_from_state()`, no a `pdf_generator.py` directamente.

**Motivo:** Unificar la vía de generación PDF en `report_engine.py`; eliminar dependencia directa de `pdf_generator.py` (ya archivado). Las dos vías que existían en v1.0 (editor visual → `from_state`; graficar → `pdf_generator` directo) se consolidan en una.

**Consecuencia:** Toda generación PDF pasa por `report_engine.py`. `pdf_generator.py` ya no forma parte del código vivo.

**Fuente:** Commit `1b269f32` ("Graficar recableado al motor HTML"); `pages/graficar.py:1992-2048`.

---

## Pendiente de confirmar (desde `ESTUDIO_impresion_independiente.md §7`)

Las siguientes preguntas del estudio no tienen evidencia de resolución explícita:

| # | Pregunta original | Estado | Evidencia encontrada |
|---|---|---|---|
| 6 | Gestión de assets de imágenes en plantillas externas (logos, imágenes corporativas) | **Pendiente confirmar** | `utils/asset_manager.py` se conserva con funciones de lectura (`get_asset_path`, `resolve_image_path`). No hay documentación de cómo se transfieren assets desde Maketator a `biblioteca_plantillas/_assets/`. |
