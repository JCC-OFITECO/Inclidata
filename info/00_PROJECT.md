# 00 — Proyecto: IncliData
_Fecha: 2026-10-11_

---

## Propósito

IncliData es una aplicación web Python/Dash para la gestión integral de datos de instrumentación geotécnica, principalmente inclinómetros de taludes y túneles. Sus funciones son:

- **Importar** lecturas de campo desde formatos RST, Sisgeo y Soil_dux (archivos CSV/XML del instrumento)
- **Corregir** datos: eliminación de bias sistemático y corrección puntual de spikes
- **Visualizar** gráficas interactivas de desplazamiento (perfil deformada, evolución temporal, vista polar, vista 3D)
- **Generar informes PDF** a partir de plantillas diseñadas en Maketator
- **Gestionar umbrales** de alerta y alarma por sensor

Las plantillas de informe **no se diseñan en IncliData**. Se crean en Maketator (editor visual externo) y se depositan en `biblioteca_plantillas/html/`. IncliData actúa como impresor: recibe plantilla + datos de sensor y produce el PDF.

---

## Stack tecnológico

| Capa | Tecnología | Versión mínima |
|---|---|---|
| Framework web | Dash | 4.x |
| Componentes UI | Dash Mantine Components v2 + Dash Bootstrap Components | 2.5.1 / 2.0.4 |
| Visualización interactiva | Plotly | 6.5 |
| Motor PDF | HTML → Playwright/Chromium (via `engines/html_engine.py`) | playwright 1.x |
| Datos | Pandas (pin <3.0) + NumPy + SciPy | 2.x / 2.4 / 1.17 |
| Validación (reducida) | Pydantic v2 | 2.0 |
| Tests | pytest | 8.0 |

**Motor activo:** HTML/Playwright, trasplantado de Maketator. ReportLab está archivado en `info/legacy/` y fuera de `requirements.txt`.

---

## Subsistemas

| Subsistema | Ruta Dash | Módulos principales |
|---|---|---|
| Bienvenida / info sensor | `/` | `pages/info.py` |
| Importación de datos | `/importar` | `pages/importar.py`, `utils/funciones_importar.py` |
| Correcciones | `/correcciones` | `pages/correcciones.py`, `pages/correcciones_layout.py`, `utils/funciones_correcciones.py` |
| Visualización | `/graficar` | `pages/graficar.py`, `pages/graficar_layout.py`, `utils/funciones_graficar.py` |
| Gestión de umbrales | `/importar_umbrales` | `pages/importar_umbrales.py` |
| Generación PDF | (modal en Graficar) | `engines/html_engine.py`, `utils/report_engine.py`, `biblioteca_plantillas/html/`, `biblioteca_graficos/html/`, `biblioteca_tablas/funciones/` |

---

## Metodología de trabajo (Claude Web / Claude Code / CodeGraph)

| Rol | Herramienta | Función |
|---|---|---|
| **Diseño y planificación** | Claude Web (Proyecto) | Análisis de arquitectura, revisión de decisiones, planificación de funcionalidades. Su Knowledge Base contiene solo los 8 documentos `info/00…06` + `_migracion/CONTRATO_motor_maketador.md`. Nunca copias de código. |
| **Implementación** | Claude Code | Actúa sobre el repositorio local. Antes de modificar código existente, usa CodeGraph para localizar símbolos, rastrear dependencias y evaluar blast-radius. |
| **Navegación estructural** | CodeGraph | Índice local del repo (`.codegraph/`). Permite localizar símbolos, ver fuentes verbatim con números de línea y trazar llamadas entre funciones. Iniciado con `codegraph init` en la raíz. |
| **Control de versiones** | Usuario | El usuario controla los commits y el push. Claude Code no hace commit sin instrucción explícita. |

---

## Relación con Maketator

IncliData y Maketator comparten el motor de renderizado (`engines/html_engine.py` + `utils/report_engine.py`). Cualquier cambio en estos archivos debe coordinarse con Maketator y trasplantarse; no se parchean localmente.

El contrato de integración está en `_migracion/CONTRATO_motor_maketador.md`.

---

## Glosario canónico

| Término | Definición |
|---|---|
| Eje A / Eje B | Direcciones ortogonales de medida del inclinómetro |
| a0 / a180 / b0 / b180 | Lecturas cara 0° y 180° por eje (base del checksum) |
| depth | Profundidad a lo largo del sondeo |
| cota_abs | Cota absoluta del punto de lectura |
| Campaña | Conjunto de lecturas de una fecha |
| Lectura de referencia (base) | Campaña base sobre la que se calculan los desplazamientos incrementales |
| Desplazamiento incremental / acumulado | Salidas de cálculo (milímetros) |
| Checksum | Control de calidad: a0 + a180 ≈ 0 |
| Token `$CURRENT*` | Tokens de resolución dinámica en plantillas: `$CURRENT` → nombre del sensor, `$CURRENT_fecha_final`, etc. |
| ScriptRegistry | Registro dinámico de scripts de gráficos y tablas (`utils/script_registry.py`) |
| Motor PDF html | Motor HTML/Playwright (activo) |
| Motor PDF reportlab | Motor ReportLab/Matplotlib (retirado, en `info/legacy/`) |
| Lámina / plantilla | JSON de configuración para regenerar un informe PDF |
| json_inclis | Puente en disco: JSON de sensor escrito por Graficar al subir archivo, leído por el motor HTML en tiempo de render |
| `params_clasificacion` | Clasificación primario/secundario de parámetros embebida en el JSON de plantilla |
| Bias | Deriva sistemática en la lectura del inclinómetro, corregible por polilínea de profundidad |
| Spike | Lectura puntual anómala, corregible manualmente por profundidad |
| Umbral | Límite de desplazamiento (alerta / alarma) definido por cota y flanco (positivo/negativo) |
