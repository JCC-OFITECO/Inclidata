# 06 — Estado Actual del Proyecto
_Fecha: 2026-10-11_

---

## Versión actual

**v2.0** — tag `v2.0-motor-html` (2026-07-03)

Hito: migración completa al motor HTML/Playwright, eliminación del editor visual, archivado de ReportLab. IncliData es a partir de aquí un viewer/corrector/printer sin capacidad de edición de plantillas.

---

## Completado

| Área | Descripción |
|---|---|
| Motor HTML/Playwright | `engines/html_engine.py` + `utils/report_engine.py` trasplantados y operativos |
| Eliminación del editor | `pages/editor_plantilla.py`, `pages/editor_visual.py`, `dash_component_editor` eliminados |
| ReportLab retirado | Fuera de `requirements.txt` y del código vivo; archivado en `info/legacy/` |
| Puente json_inclis | Escritura al subir sensor; lectura por motor HTML y funciones de tabla |
| Refactorización Fase 1-3 | Limpieza, separación de layouts, extracción a utils |
| Arranque dual | Estable (`python app.py`) y dev (`INCLIDATA_DEBUG=1`) con `exclude_patterns` |
| Correcciones Rozmarin | Datos de sensores IN-02, 07, 09, 14, 21, 22 y spikes corregidos (commit `e585675d`, 2026-10-11) |
| Vista 3D y slider temporal | Vista 3D interactiva, slider temporal proporcional con snap, eje temporal independiente (commits `cea73f5f`–`dc411fc4`) |
| CodeGraph | Indexado (`.codegraph/`, 2026-10-11, 132 archivos, 1424 nodos, 2643 aristas) |

---

## Backlog activo

| Prioridad | Área | Descripción |
|---|---|---|
| 🟡 Media | `pages/graficar.py` | Callbacks `actualizar_graficos` y `actualizar_grafico_temporal` muy largos; candidatos a extracción a `utils/funciones_graficar.py` |
| 🟢 Baja | `models/server.py`, `models/database.py` | Stubs que ocultan la dependencia con Maketator; documentados pero no eliminables sin cambio en `html_engine.py` |
| 🟢 Baja | `biblioteca_plantillas/incli_L9_BCN_v0[b]/` | Corchetes en el nombre del directorio pueden romper glob/shell en algunos entornos |
| 🔵 Info | Ingesta de plantillas desde Maketator | La única vía es copiar manualmente la carpeta `{nombre}/` en `biblioteca_plantillas/html/`. El endpoint HTTP (Opción B del ESTUDIO §4.4) nunca se implementó ni se tomó decisión explícita de descartarlo. `generate_report_pdf_from_state()` es el mecanismo interno de render —el nombre es histórico del editor React ya eliminado— no un mecanismo de ingesta. |
| 🔵 Info | Gestión de assets de plantillas externas | No documentado cómo se transfieren imágenes (logos, imágenes corporativas) desde Maketator a `biblioteca_plantillas/_assets/`. `utils/asset_manager.py` conserva las funciones de lectura (`get_asset_path`, `resolve_image_path`) pero no hay procedimiento definido para la escritura desde fuera. |
| 🟢 Baja | Vista previa de informe | `report_engine.py` expone `generate_preview_html_from_state` y `render_preview_png` pero no están cableados a la UI de Graficar. Una vista previa real (sin ReportLab) es posible sin nueva infraestructura. |
| 🟢 Baja | Mantenimiento del motor | Revisar periódicamente re-trasplante si Maketator evoluciona `html_engine` (nuevos tipos de elemento, rama BD real). Método: inventario de imports → clausura → condiciones de parada. |

El backlog de la "Fase 4" descrito en `state.md` (refactorización de `editor_plantilla.py`) quedó obsoleto al eliminarse el editor en v2.0.

---

## Archivos residuales pendientes de limpieza

| Elemento | Estado git | En `.gitignore` | Detalle |
|---|---|---|---|
| `debug_bias.json` | Rastreado (`git ls-files` lo devuelve) | No | Generado en runtime por `pages/correcciones.py`. Se indicó como "eliminado" en `state.md §1.3` pero reaparece. |
| `debug_spikes.json` | Rastreado | No | Ídem. |
| `kk/grafico_inclinometro_v1.py` | Rastreado | No | Script de gráfico antiguo. `state.md §1.6` indicó la eliminación de carpetas `kk/` pero esta sigue presente. |
| `temp_lovable_reference/` | Rastreado | No | Referencia temporal del prototipo Lovable (commit `a71aa217`). No tiene uso funcional. |

> Ninguno de estos archivos se modifica en esta fase de documentación. Son candidatos a limpieza en un commit futuro.

---

## Deuda técnica activa

Fuente: `CONTEXT.md`, `state.md`.

| Item | Prioridad |
|---|---|
| `pages/graficar.py` — callbacks muy largos (`actualizar_graficos`, `actualizar_grafico_temporal`) | Media |
| `models/server.py` y `models/database.py` — stubs que ocultan dependencias de Maketator | Baja |
| `biblioteca_plantillas/incli_L9_BCN_v0[b]/` — corchetes en nombre pueden romper glob | Baja |
| `pages/spikes.json` — ruta relativa intencional (`path calculado en correcciones.py:1059`); documentar mejor | Baja |
| `README.md` — desactualizado (describe v1.0 con ReportLab/Matplotlib y editor de plantillas) | Baja |
| `CONTEXT_maketator_web.md` en el repo — describe arquitectura de Maketator Web; su presencia genera confusión | Baja |
| `print()` residuales en páginas vivas — confirmados en `graficar.py:541` (fallback leyenda) | Baja |
| Logs `[HTML][DBG]` y `[DEBUG-STATIC-FN]` del motor ruidosos en nivel `WARNING`; corrección se coordina con Maketator | Baja |
| `sensor_id` sin sanitización de nombre de archivo: si `nom_sensor` contiene caracteres inválidos en rutas, la escritura de `json_inclis/{sensor_id}.json` falla en `graficar.py:400` sin validación previa. Además, si `nom_sensor` difiere del stem del archivo subido, el nombre del archivo en `data/` y `sensor_id` divergen (no rompe el render, que usa `json_inclis/`, pero viola la convención de 04 §1). | Baja |

---

## Discrepancias detectadas

Estado tras la verificación en la Fase 0 de auditoría.

| # | Discrepancia | Estado | Verificación |
|---|---|---|---|
| 1 | `README.md` describe v1.0: Matplotlib, ReportLab, editor de plantillas como activos | **Confirmada** | README no fue actualizado en v2.0. Contenido obsoleto. |
| 2 | `info/PROJECT_SNAPSHOT.md` (2026-07-01) dice "motor activo = ReportLab" | **Confirmada** | Snapshot tomado 2 días antes de v2.0 (2026-07-03). Quedó inmediatamente obsoleto. |
| 3 | `info/ESTUDIO_impresion_independiente.md §7` lista 8 decisiones "abiertas" | **Parcialmente resuelta** | 7 de 8 tienen evidencia de resolución en código/commits (DEC-001 a DEC-009). La #6 (assets) permanece pendiente. Ver `05_DECISIONS.md`. |
| 4 | `CONTEXT_maketator_web.md` describe módulos de Maketator Web que no existen en IncliData | **Confirmada** | `dispatch_table.py`, `admin_servidores.py`, ORM SQLite completo, `engines/reportlab_engine.py`, `core/job_runner.py` no existen en IncliData. El archivo es referencia del sistema origen del trasplante. |
| 5 | `state.md §Fase 4` menciona `pages/editor_plantilla.py` como pendiente de refactorizar | **Confirmada (obsoleta)** | El archivo fue eliminado en v2.0 (commit `9af43705`). La Fase 4 del backlog quedó desactualizada. |
| 6 | `kk/` descrita como "eliminada" en `state.md §1.6` pero existe y está rastreada por git | **Confirmada** | `git ls-files kk/` devuelve `kk/grafico_inclinometro_v1.py`. La carpeta `kk/` del estado anterior (plantillas sandbox) fue eliminada, pero esta `kk/` con un script de gráfico es diferente o fue recreada. |
| 7 | `debug_bias.json` y `debug_spikes.json` descritos como "eliminados" en `state.md §1.3` pero presentes y rastreados | **Confirmada** | Son generados en runtime por `pages/correcciones.py`. La limpieza de `state.md §1.3` se refería a versiones anteriores; reaparecen como artefactos de ejecución. |
| 8 | `models/server.py` y `models/database.py` en `CONTEXT.md §stubs` vs. su contenido real | **Resuelta** | Verificado: son stubs con `class Server` y `get_session()` que lanzan `RuntimeError` explícito. Comportamiento exactamente el descrito. No hay discrepancia funcional. |

---

## Recuperación del estado anterior

| Punto de restauración | Descripción |
|---|---|
| `tag v1.0-pre-poda-reportlab` | Estado completo con editor visual React, motor ReportLab activo, `dash_component_editor` |
| `rama archive/full-editor-reportlab` | Ídem; rama de archivo permanente |
