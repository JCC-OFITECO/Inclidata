# 03 — Reglas de Negocio del Dominio
_Fecha: 2026-10-11_

Reglas funcionales del dominio de inclinometría geotécnica implementadas en IncliData. Las inferencias no verificadas directamente en código se marcan **[INFERENCIA]**.

---

## Importación de campañas

### Wizard de importación (5 pasos)
Fuente: `Guía_uso_Importar.md`, `pages/importar.py`, `utils/funciones_importar.py`.

1. **Selección del archivo base:** El usuario elige un JSON de sensor existente de la carpeta `data/`. La app lo carga en el store `dcc.Store id='tubo'`.

2. **Configuración del importador:** El sistema detecta automáticamente el tipo de importador y el `index_0` utilizados en la última campaña existente, y los ofrece como valor por defecto. El usuario puede cambiar el tipo de importador (RST / Sisgeo / Soil_dux) y el `index_0` (profundidad de inicio de la sonda).

3. **Subida de archivos de datos:** El usuario arrastra o selecciona los archivos de campo: `.csv` para RST, `.xml` para Sisgeo. Se muestra lista de confirmación.

4. **Procesamiento y previsualización:**
   - Se parsean los archivos y se invocan `import_RST`, `import_Sisgeo`, etc. en `utils/funciones_importar.py`.
   - Se extraen lecturas brutas a0, a180, b0, b180; se convierten a milímetros y se normalizan.
   - **Validación cronológica crítica:** Las fechas de las nuevas campañas no pueden ser anteriores a la primera campaña de referencia existente. Si hay conflicto, el proceso se bloquea con alerta.
   - Se calculan desplazamientos incrementales comparando con la referencia.
   - Se evalúan umbrales de alarma.
   - Se genera tabla de configuración donde el usuario puede ver y editar: fecha, hora, estado activo/cuarentena, incluir/ignorar.

5. **Guardado:** La función `insertar_camp` lee el archivo original del disco (seguridad), inserta las nuevas campañas y **reordena el JSON cronológicamente** antes de escribirlo. El orden cronológico es fundamental para la consistencia de las series temporales.

### Cálculo de desplazamientos

- `dev_a = (a0 - a180) / 2` por cada profundidad. Ídem para `dev_b`.
- Los desplazamientos en `calc[]` se expresan en **milímetros (mm)**.
- Los datos en `raw[]` son las lecturas originales del instrumento; los scripts de gráficos y tablas solo consumen `calc[]`.

### Checksum

El checksum (`checksum_a = round(a0 + a180, 4)`, `checksum_b = round(b0 + b180, 4)`) se **calcula y almacena en `calc[]`** durante la corrección de spikes (`pages/correcciones.py:465-466`). Es un indicador de calidad de la lectura del instrumento (a0 + a180 debe ser ≈ 0 si el instrumento está bien calibrado).

El checksum **no se valida durante la importación** con ningún criterio de bloqueo. Se visualiza en la página Correcciones (gráficos `incr_checksum_a/b`) y está disponible como columna seleccionable en los scripts de gráficos (`biblioteca_graficos/html/grafico_inclinometro_v*.py`). Fuente: `pages/correcciones.py:465-466`, `pages/correcciones.py:954-958`.

---

## Corrección de bias

Fuente: `utils/funciones_correcciones.py`, `pages/correcciones.py`.

El bias es una deriva sistemática en la lectura, modelada como una polilínea por tramos de profundidad.

- Se definen hasta cuatro correcciones: **Bias_1_A**, **Bias_1_B**, **Bias_2_A**, **Bias_2_B** (dos por eje A y B).
- Cada corrección tiene: `Prof_inf` (inicio del tramo), `Prof_sup` (fin del tramo), `Delta` (pendiente de corrección), `Selec` (activa o no).
- El incremento de corrección por profundidad = `Delta / n_puntos_en_tramo`.
- Los valores corregidos se almacenan en columnas `recta_a`, `recta_b`, `corr_a`, `corr_b`.
- Solo las correcciones con `Selec=True` se aplican.

---

## Corrección de spikes

Fuente: `pages/correcciones.py:412-465`.

Un spike es una lectura puntual anómala en a0, a180, b0 o b180 para una profundidad específica y campaña.

- El usuario identifica visualmente el spike en la tabla/gráfico y marca el flag A o B para esa profundidad.
- El sistema sustituye la lectura original por el valor corregido (`a0_c`, `a180_c`, `b0_c`, `b180_c`).
- La corrección se registra en la clave `"spike": {"A": [...], "B": [...]}` dentro de la campaña.
- Se recalcula `dev_a = (a0 - a180) / 2` y `dev_b` con los valores corregidos.
- Si no hay flags activos (A ni B), se restauran los valores originales sin corrección.

---

## Evaluación de umbrales

Fuente: `utils/funciones_comunes.py:490-549`.

Los umbrales definen límites de desplazamiento (alerta / alarma) para alertar de movimientos excesivos.

### Estructura de umbrales

- `umbrales.deformadas`: dict de curvas umbral. Cada curva tiene `flanco` (`sin_flanco` / `flanco_positivo` / `flanco_negativo`) y `nivel` (entero; cuanto mayor, más grave).
- `umbrales.valores`: lista de puntos `{cota_abs, umbral1_a, umbral2_a, ...}` que definen la curva por interpolación lineal.

### Lógica de evaluación

1. Para cada curva en `deformadas` (excluyendo `sin_flanco`):
   - Se interpola linealmente el valor umbral en la cota del punto de lectura.
   - Si `flanco_positivo` y `desp > umbral` → superación.
   - Si `flanco_negativo` y `desp < umbral` → superación.
2. Si hay múltiples superaciones, se devuelve la de mayor `nivel`.
3. Si no hay superaciones, devuelve `None`.

Los umbrales se importan desde Excel (`/importar_umbrales`).

---

## Visualización interactiva

Fuente: `pages/graficar.py`, `utils/funciones_graficar.py`.

Tipos de gráfico disponibles (Plotly):
- **Perfil deformada:** desplazamiento vs. profundidad (eje A y/o B)
- **Evolución temporal:** desplazamiento vs. fecha para una profundidad seleccionada
- **Vista polar:** representación 2D del vector de desplazamiento
- **Vista 3D interactiva:** visualización volumétrica del perfil a lo largo del tiempo (alternativa al polar)
- **Slider temporal:** control de campaña activa con granularidad adaptativa y marcas en la línea temporal proporcional

---

## Generación de informes PDF

Fuente: `CONTRATO_motor_maketador.md`, `pages/graficar.py`, `utils/report_engine.py`.

1. El usuario selecciona plantilla y parametriza los valores `$CURRENT` (sensor, fechas, etc.) en el modal de informe dentro de Graficar.
2. El contexto de render se construye con las claves requeridas por el motor HTML (ver `04_DATA_MODEL.md §Contexto de render`).
3. Se invoca `generate_report_pdf_from_state(editor_state, context, output_path)`.
4. El motor HTML resuelve los tokens `$CURRENT*` y llama a los scripts de gráficos/tablas.
5. El PDF resultante se descarga vía `dcc.Download`.

**Prerrequisito:** el archivo del sensor debe haber sido subido en la misma sesión para que `json_inclis/{sensor_id}.json` exista en disco. Si no existe, la generación falla con mensaje claro al usuario.

---

## Gestión de umbrales desde Excel

Fuente: `pages/importar_umbrales.py`. **[INFERENCIA: flujo detallado no verificado en código directamente; descrito a alto nivel]**

El usuario sube un archivo Excel con la definición de umbrales. El sistema parsea los valores y los almacena en la estructura `umbrales.deformadas` + `umbrales.valores` del JSON del sensor.
