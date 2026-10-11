CLAUDE.md - Contexto de Inclidata

IncliData es un visor/corrector/impresor de datos de instrumentación geotécnica (inclinómetros).
Las plantillas de informe se diseñan en Maketator (editor eliminado de IncliData en v2.0).

---

📚 Documentación estructurada (info/)

    info/00_PROJECT.md       Propósito, stack, subsistemas, glosario, papel Claude Web/Code/CodeGraph
    info/01_RULES.md         Reglas técnicas etiquetadas [REGLA] / [CONVENCION] / [RESTRICCION-MODULO]
    info/02_ARCHITECTURE.md  Estructura de carpetas, responsabilidades, flujos, puntos de extensión
    info/03_BUSINESS_RULES.md Reglas funcionales del dominio (importación, corrección, umbrales, PDF)
    info/04_DATA_MODEL.md    Modelos JSON (sensor, plantilla, store, contexto de render)
    info/05_DECISIONS.md     Decisiones DEC-NNN con fuente verificable
    info/06_CURRENT_STATUS.md Estado actual, backlog, discrepancias, deuda técnica

    Contrato de integración con Maketator: _migracion/CONTRATO_motor_maketador.md

---

🔍 CodeGraph (antes de modificar código existente)

    Usar CodeGraph ANTES de modificar código existente para:
    - Localizar símbolos (dónde está definida una función, qué archivos la usan)
    - Evaluar blast-radius (qué se rompe si cambio X)
    - Ver fuente verbatim con números de línea sin abrir el archivo

    MCP:   codegraph_explore (query + projectPath)
    Shell: codegraph explore "<símbolo o pregunta>"

    El índice vive en .codegraph/ (excluido del repo vía .gitignore).

---

🚫 Control de versiones

    El usuario controla los commits y el push.
    Claude Code NO hace commit sin instrucción explícita del usuario.

    Knowledge Base de Claude Web:
    - El Project de Claude Web sincroniza desde GitHub exactamente 8 archivos: info/00_PROJECT.md
      a info/06_CURRENT_STATUS.md y _migracion/CONTRATO_motor_maketador.md. La sincronización
      es manual ("Sync now" del usuario).
    - Al cerrar un sprint, actualiza info/06_CURRENT_STATUS.md (y info/05_DECISIONS.md si hubo
      decisiones nuevas) y, si cambió alguno de esos 8 archivos, recuerda al usuario: commit,
      push y "Sync now" en el Project.
    - No crees documentos nuevos pensados para la KB ni propongas añadir archivos a ese conjunto
      sin que lo decida el usuario.

---

🛠 Comandos del Proyecto

    Iniciar App (estable):     python app.py
    Iniciar App (modo dev):    INCLIDATA_DEBUG=1 python app.py
    Tests:                     pytest
    Instalar dependencias:     pip install -r requirements.txt

---

📋 Reglas de Estilo y Arquitectura

    Python: snake_case para funciones/variables, PascalCase para clases.

    Componentes: Dash 4.x + Dash Mantine Components (DMC v2) + Dash Bootstrap Components.

    Callbacks ligeros: delegar lógica pesada a utils/. No acumular estado en callbacks.

    Motor productivo = HTML/Playwright (engines/html_engine.py + utils/report_engine.py).
    ESTOS ARCHIVOS NO SE PARCHEAN LOCALMENTE. Vienen trasplantados de Maketator; cualquier
    cambio se coordina con Maketator y se trasplanta de nuevo.

---

🏗️ Motor de Render

    Render: engines/html_engine.py → Playwright/Chromium → PDF
    Fachada: utils/report_engine.py (generate_report_pdf_from_state)
    Plantillas: biblioteca_plantillas/html/{nombre}/{nombre}.json
    Scripts gráficos: biblioteca_graficos/html/{namespace}/{script}.py
    Scripts tablas: biblioteca_tablas/funciones/{script}.py

    ReportLab está RETIRADO. El código ReportLab vive en info/legacy/ como referencia.

---

📦 Puente de datos: json_inclis

    Al subir un sensor, graficar.py escribe el JSON original a json_inclis/{sensor_id}.json.
    El motor HTML lee ese archivo en tiempo de render para generar gráficos.
    json_inclis/ está excluido del reloader (ver CONTEXT.md).

---

⚠️ Regla del store `graficar-tubo`

    Las claves raíz del store graficar-tubo son EXCLUSIVAMENTE timestamps de campaña (ISO 8601)
    más las claves reservadas `info` y `umbrales`.
    Cualquier metadato nuevo (sensor_id, nombre, etc.) va DENTRO de info{}, nunca en la raíz.

---

🚦 Arranque dual (app.py)

    Estable (por defecto): debug=False, use_reloader=False
    Dev (INCLIDATA_DEBUG=1): debug=True, dev_tools_hot_reload=False, exclude_patterns=[...]

    exclude_patterns es CRÍTICO en modo dev: evita que el reloader reinicie el servidor cuando
    la app escribe archivos en runtime (json_inclis, logs, registry.json). Sin esto los callbacks
    en vuelo mueren con "server did not respond".

---

📂 Archivos críticos

    app.py                     Punto de entrada. Arranque dual. Logging consola+archivo.
    pages/graficar.py          Modal de informe PDF, subida de sensor, visualización.
    utils/report_engine.py     Fachada del motor HTML.
    engines/html_engine.py     Motor HTML (NO parchear localmente).
    utils/template_service.py  Carga de plantillas y scripts.
    utils/asset_manager.py     Gestión centralizada de imágenes.
