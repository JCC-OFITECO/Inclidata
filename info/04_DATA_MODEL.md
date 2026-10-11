# 04 — Modelo de Datos
_Fecha: 2026-10-11_

Descripción conceptual de las estructuras de datos. No reproduce código; cita rutas y secciones del contrato. Ver `_migracion/CONTRATO_motor_maketador.md` para el esquema JSON completo y canónico.

---

## 1. Archivo de sensor inclinométrico (`data/{nombre}.json`)

Fuente: `_migracion/CONTRATO_motor_maketador.md §2.3`, `info/PROJECT_SNAPSHOT.md §Contratos`.

Estructura raíz:

| Clave | Tipo | Descripción |
|---|---|---|
| `info` | objeto | Metadatos del sensor: `nom_sensor`, `coordenadas`, `cota_1000`, etc. |
| `umbrales` | objeto | Definición de curvas de umbral. Ver §3 más abajo. |
| `<ISO 8601 timestamp>` | objeto | Una por campaña. La clave es el timestamp exacto de la medición. |

Cada campaña contiene:

| Clave | Tipo | Descripción |
|---|---|---|
| `campaign_info` | objeto | Metadatos de la campaña: `active`, `reference`, `importador`, etc. |
| `raw` | lista | Lecturas brutas del instrumento: `index`, `cota_abs`, `depth`, `a0`, `a180`, `b0`, `b180`. |
| `calc` | lista | Desplazamientos calculados: `index`, `cota_abs`, `depth`, `desp_a`, `desp_b`, `abs_dev_a`, `abs_dev_b`, `incr_dev_a`, `incr_dev_b`, etc. |
| `spike` | objeto (opcional) | Registro de correcciones de spike: `{"A": [...], "B": [...]}`. |

**Reglas:**
- Todos los desplazamientos en `calc[]` están en **milímetros (mm)**.
- Los scripts de gráficos y tablas solo consumen `calc[]`. `raw[]` es solo registro.
- El nombre del archivo (sin `.json`) debe coincidir exactamente con `context["sensor"]` en el render.
- El orden del JSON es cronológico (la importación lo garantiza en cada guardado).

---

## 2. Store Dash `graficar-tubo`

Fuente: `pages/graficar.py:344-412`, `CLAUDE.md`, commit `0e8e1251`.

El store `graficar-tubo` (`dcc.Store`) es la representación en memoria del sensor cargado en la sesión.

**Claves raíz:**

| Clave | Tipo | Descripción |
|---|---|---|
| `info` | objeto | Metadatos del sensor (copiado de `data.info`). Contiene `_sensor_id` (`info.nom_sensor` del JSON del sensor; si vacío, stem del archivo subido). |
| `umbrales` | objeto | Umbrales del sensor. |
| `<ISO 8601>` | objeto | Una por campaña activa (`active == True`). Contiene `campaign_info`, `info_readout`, `calc[]`. |

**Invariante:** las claves raíz son **solo** timestamps ISO 8601 + `info` + `umbrales`. Nunca añadir metadatos en la raíz.

**`info._sensor_id`:** valor `info.nom_sensor` del JSON del sensor; si está vacío, stem del nombre del archivo subido (`graficar.py:357`: `sensor_id = data.get("info",{}).get("nom_sensor") or Path(filename).stem`). Es la clave para localizar `json_inclis/{sensor_id}.json`.

**Posible desajuste:** si `nom_sensor` difiere del stem del archivo subido, el nombre del archivo en `data/` y el `sensor_id` divergen. El motor no se ve afectado porque lee `json_inclis/{sensor_id}.json` (no `data/`). Sin embargo, `sensor_id` se usa tal cual como nombre de archivo sin sanitización: si `nom_sensor` contiene caracteres inválidos en rutas (p.ej. `/`, `:`, `*`), la escritura de `json_inclis/{sensor_id}.json` fallará en `graficar.py:400`.

---

## 3. Estructura de umbrales

Fuente: `utils/funciones_comunes.py:490-549`.

```
umbrales
├── deformadas: {nombre_curva: {flanco, nivel}}
│     flanco: "sin_flanco" | "flanco_positivo" | "flanco_negativo"
│     nivel: int (mayor = más grave)
└── valores: [{cota_abs, nombre_curva1, nombre_curva2, ...}]
              (puntos de la curva, interpolables por cota_abs)
```

---

## 4. Plantilla de informe JSON (`biblioteca_plantillas/html/{nombre}/{nombre}.json`)

Fuente: `_migracion/CONTRATO_motor_maketador.md §2.1`. Ver ese documento para el esquema completo y canónico.

Estructura raíz:

| Campo | Valor | Descripción |
|---|---|---|
| `engine` | `"html"` | Selecciona el motor HTML/Playwright |
| `configuracion` | objeto | `nombre_plantilla` (str), `num_paginas` (int) |
| `paginas` | objeto | Dict de páginas; clave = número de página como string (`"1"`, `"2"`, …) |
| `secciones` | objeto | Encabezados/pies; round-trip completo, no modificar |

Cada página contiene un dict `elementos` donde cada elemento tiene:

| Campo | Descripción |
|---|---|
| `tipo` | `"grafico"` / `"tabla"` / `"texto"` / `"imagen"` / `"rectangulo"` / `"linea"` |
| `geometria` | Posición y tamaño en centímetros: `x`, `y`, `ancho`, `alto` |
| `estilo` | Color, fuente, borde, opacidad |
| `configuracion` | Para gráfico/tabla: `script`, `parametros`, `params_clasificacion` |
| `cuadricula` | Solo tipo tabla: niveles, columnas ancla, funciones de celda |
| `imagen_config` | Solo tipo imagen: modo estático/dinámico |

**Token `$CURRENT`** en `configuracion.parametros`: se resuelve en tiempo de render desde `context["sensor"]`. Tokens derivados: `$CURRENT_fecha_seleccionada`, `$CURRENT_fecha_inicial`, `$CURRENT_fecha_final`, `$CURRENT_ultimas_camp`.

**Rutas de scripts:** namespace `html/` en el nombre del script (`"html/grafico_inclinometro_v2.py"`).

---

## 5. Contexto de render (`context`)

Fuente: `utils/report_engine.py:104`, `pages/graficar.py:1992-2048`, `CONTRATO_motor_maketador.md §2.2`.

El dict `context` es el puente entre la UI de Graficar y el motor HTML.

| Clave | Tipo | Descripción |
|---|---|---|
| `sensor` | str | Stem del archivo del sensor (= `info._sensor_id`). El motor usa esto para leer `json_inclis/{sensor}.json`. |
| `sensores` | lista | Lista de sensores; normalmente `[sensor]`. |
| `info` | objeto | Metadatos del sensor (del store `graficar-tubo.info`). |
| `fecha_seleccionada` | str | Timestamp ISO de la campaña seleccionada en el slider. |
| `fecha_inicial` | str | Inicio del rango temporal seleccionado. |
| `fecha_final` | str | Fin del rango temporal seleccionado. |
| `ultimas_camp` | int | Número de últimas campañas a incluir. |
| `data` | objeto | Siempre `{}` en IncliData (`graficar.py:2047`). En Maketator el job runner lo rellena con datos históricos (`{historico: [...]}`) para cell functions que no usan `json_inclis/`. El motor lo lee en `html_engine.py:1568,3068,3218` pero al estar vacío no activa la rama de fetch (requeriría `server_id`). |

Si alguna clave falta, el motor rellena con valores de `DUMMY_CONTEXT` (datos de maquetación).

---

## 6. Puente json_inclis

Fuente: `pages/graficar.py:396-412`, `biblioteca_tablas/funciones/columna_incli_json.py`.

`json_inclis/` es un directorio de intercambio en disco entre la UI y el motor HTML.

- **Escritura:** `pages/graficar.py` escribe el JSON completo del sensor en `json_inclis/{sensor_id}.json` al subir un archivo de sensor.
- **Lectura:** `biblioteca_tablas/funciones/columna_incli_json.py` (`_load_sensor_json()`) lee el archivo con caché en memoria (`_json_cache`). Los scripts de tablas lo usan para obtener los datos de `calc[]` del sensor.
- **No versionado:** `json_inclis/*.json` está en `.gitignore`. La carpeta se preserva con `.gitkeep`.
- **Caché:** inmediatamente después de escribir el nuevo JSON en disco, `pages/graficar.py:404-405` importa `columna_incli_json` y llama `_json_cache.clear()` (limpieza total del dict de módulo, acceso directo). La caché se rellena de nuevo en la siguiente llamada a `_load_sensor_json()`. Persiste entre renders PDF de la misma sesión mientras no se suba un sensor nuevo. Fuente: `pages/graficar.py:404-408`.

---

## 7. Scripts de gráficos (`biblioteca_graficos/html/`)

Fuente: `utils/script_registry.py`, `CONTRATO_motor_maketador.md §2.4`.

Cada script expone:

| Elemento | Descripción |
|---|---|
| `PARAMETER_METADATA` | Lista de dicts que describen los parámetros del script: nombre, tipo, label, default |
| `metadata` | Instancia `ScriptMetadata` registrada con `@register_script` |
| `generate(params, figsize)` | Función principal: recibe parámetros resueltos y tamaño en cm; devuelve HTML string (Plotly) |

Los nombres de script incluyen namespace: `"html/grafico_inclinometro_v2.py"`.

---

## 8. Funciones de celda (`biblioteca_tablas/funciones/`)

Fuente: `utils/cell_function_registry.py`, `CONTEXT_maketator_web.md §convenciones`.

Cada función de celda expone:

| Elemento | Descripción |
|---|---|
| `CELL_FUNCTION_METADATA` | Dict con nombre, tipo de retorno y lista de parámetros |
| `evaluate(params, data, context)` | Función principal: parámetros ya resueltos, `data={"historico":[...]}`, context del render |

Las funciones que trabajan con datos de sensor leen `json_inclis/` directamente: `columna_incli_json`, `ultima_lect_incli`, `penultima_lect_incli`, `antepenultima_lect_incli`, `nesima_lect_incli` (todas definen `_JSON_INCLIS_DIR` y leen `{sensor}.json` del disco). El motor les pasa los parámetros ya resueltos desde el contexto (`sensor`, `fecha`…) pero no les inyecta los datos; cada función hace su propia lectura. Las funciones que no trabajan con datos de sensor (p.ej. funciones de texto/formato) solo usan los `params` resueltos.
