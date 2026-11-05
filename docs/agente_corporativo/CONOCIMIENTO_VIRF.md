# Conocimiento VIRF — base operativa para el agente corporativo

> Documento corto y autosuficiente. Súbelo SIEMPRE junto con la skill del cambio. El detalle
> completo (catálogo de módulos, endpoints, tests, variables, grafo y capturas) está en
> `GUIA_VIRF_AGENTE_CORPORATIVO.pdf`.

## 1. Qué es VIRF

VIRF es un **agente de validación de modelos de riesgo financiero** (VaR, LaR, ALM…) usado por
la Gerencia de Validación Interna de Riesgo Financiero (segunda línea de defensa). Recibe la
evidencia de un modelo (manuales, código, datos, actas), la contrasta contra un **estándar de
validación** (Excel de ámbitos, controles y actividades) y produce un **informe Word** con la
calificación del modelo.

Tiene dos formas de uso:

- **Web** (FastAPI + JavaScript sin framework): el validador sigue 9 pasos guiados, conversa con
  el agente en un chat de sustentación y descarga el informe. Arranque:
  `.venv\Scripts\python.exe -m uvicorn web.app:app --port 8000`.
- **Línea de comandos**: `agente.py --prdc …` ejecuta el pipeline completo y emite el entregable
  regulatorio. La web lo lanza como subproceso.

Y un tercer modo, **Solo Informe**: genera el informe solo con el estándar y observaciones
registradas a mano, sin ejecutar el pipeline.

## 2. Arquitectura en capas

```
Navegador ── web/static/js/app.js · web/templates/index.html · web/static/css/styles.css
   │  HTTP /api/... y WebSocket /ws/...
   ▼
web/routers/*.py      ← endpoints (validan entrada, devuelven JSON o archivos)
   ▼
web/services/*.py     ← lógica de la web (sesión, estándares, chat, informes web, Solo Informe)
   ▼
modulos/*.py          ← núcleo de negocio (motor IA, informes Word, reglas, PII, custodia…)
config/*.py           ← parámetros, rutas, proveedor de IA (ÚNICO lugar que elige proveedor)
motor/Mx_0x_*.py      ← pipeline de lectura de evidencias, prompt, llamada a IA, controles de datos
src/rag/*.py          ← RAG léxico (BM25 + TF-IDF, local, sin embeddings)
agente.py             ← orquestador del pipeline (CLI)
tests/*.py            ← ~1200 tests, todos sin coste (IA simulada)
```

Regla de dependencias: `web/` puede importar `modulos/`, `config/`, `src/`; **`modulos/` no
importa `web/`** (el pipeline CLI no tiene web). Por eso la plantilla de informes y la regla de
calificación viven en `modulos/`.

## 3. Datos y rutas

- Sesiones web: `web/data/sesiones/{codigo_validador}/{SLUG_MODELO}/inputs|outputs`.
  - `inputs/documentacion/modelamiento|seguimiento|controles_datos`, `inputs/implementacion`
    (código del modelo, se descomprime del ZIP), `inputs/calidad_datos/…`, `inputs/automatizaciones`
    (Excels 00–03), `<sesión>/inputs/estandares/`.
  - `outputs/AAAA-MM-DD/resultados_*.json` (resultado del pipeline), `outputs/VIRF_Informe_*_Vn_*.docx`
    (informe web versionado), `<sesión>/outputs/capturas/` (pantallazos de evidencia).
- Solo Informe: `web/data/sesiones/{codigo}/_solo_informe/` (estándar, borrador, imágenes,
  manuales, salida).
- Globales: `inputs/Estandares_VIRF.xlsx`, `inputs/rag/` (literatura por dominio),
  `inputs/logoempresa/` (logo BCP), `inputs/repositorio_controles/<dominio>/` (controles en Python).
- Credenciales: `.env` en la raíz (**nunca** se edita desde el código ni se sube al agente);
  plantilla documentada en `.env.example`.

## 4. Funciones principales y dónde viven

| Función | Pantalla | Archivos clave |
|---|---|---|
| Login y modelos | Login, Mis Modelos | `web/routers/auth.py`, `modelos.py`, `web/services/sesion_store.py`, `modelos_store.py` |
| Paso 01 Configuración (proveedor IA, tier) | Configuración | `web/routers/config.py`, `web/services/env_service.py`, `config/ia.py`, `config/parametros.py` |
| Paso 02 Estándares | Estándares | `web/routers/estandares.py`, `web/services/estandares_service.py`, `motor/Mx_01_LecturaEvidencias.py` (`leer_estandares_virf`) |
| Paso 03 Evidencia | Evidencia | `web/routers/upload.py`, `web/services/contexto_sesion.py`, `analisis_codigo.py` |
| Paso 04 Dominio & RAG | Dominio y literatura | `web/routers/rag.py`, `src/rag/*`, `web/services/dominios_custom_store.py` |
| Pasos 05–06 Skills y cuestionamientos | Skills, Cuestionamientos | `web/routers/skills.py`, `cuestionamientos*.py`, `modulos/skills/`, `modulos/cuestionamientos_ext.py` |
| Paso 07 Pre-vuelo | Pre-vuelo | `web/routers/dominio_prevuelo.py`, `modulos/prevuelo.py` |
| Paso 08 Validar | Validar modelo | `web/routers/validacion.py`, `web/services/pipeline_runner.py`, `agente.py`, `modulos/motor_ia.py`, `motor/*` |
| Paso 09 Documentar (informe web) | Documentar | `web/routers/informe.py`, `web/services/informe_web.py`, `modulos/plantilla_bcp.py`, `modulos/capturas.py` |
| Resultados | Resultados | `web/routers/resultados.py` |
| Controles de datos / notebook | Controles de datos | `web/routers/controles.py`, `modulos/motores_datos.py`, `notebook_controles.py`, `repositorio_controles.py` |
| Chat de sustentación | Chat de validación | `web/routers/chat.py`, `web/services/chat_service.py`, `modulos/consolidacion.py` |
| Solo Informe (+ IA opcional) | Solo Informe | `web/routers/solo_informe.py`, `web/services/solo_informe_*.py` |
| Informe del pipeline (CLI) | — | `modulos/generador_word.py` → `modulos/plantilla_bcp.py` |
| Calificación (los 3 informes) | — | `modulos/reglas_informe.py` |

## 5. Los tres informes Word (misma plantilla)

`modulos/plantilla_bcp.py` define la estructura común: portada BCP, encabezado «Página | N»
con logo, pie «VALIDACIÓN INTERNA DE RIESGO FINANCIERO · Informe Final», índice con
hipervínculos, **I. Objetivos · II. Alcance · III. Cumplimiento de Estándar · IV. Información
Relevante (ficha del documento, contexto, limitaciones, fechas, conclusión) · V. Cuadro Resumen
de Observaciones · VI. Glosario · VII. Barrido de Controles** y anexos A, B, C…

- Solo Informe → `web/services/solo_informe_word.py` (adaptador fino).
- «Generar informe» de la web → `web/services/informe_web.py` (añade puntaje, evidencia,
  contradicciones manual↔código, capturas, diálogo del chat; anexos: tabla final,
  entendimiento, automatizaciones, trazabilidad).
- Entregable del pipeline → `modulos/generador_word.py::generar_word_informe` (anexos: firma,
  custodia, escala, correcciones, análisis transversales).

## 6. Regla de calificación (única para los tres)

`modulos/reglas_informe.py::concluir(observaciones)`:

- **Deficiente**: más de 2 críticas.
- **Regular**: 1 o 2 críticas; o más de 1 moderada (umbral 3 o más si hay al menos una moderada
  de Documentación y una de Gobierno).
- **Aceptable**: 1 moderada (o 2 con la excepción Doc+Gob); o 5 o más bajas/potenciales.
  Nunca lleva críticas.
- **Satisfactorio**: hasta 4 bajas/potenciales.
- Las Oportunidades de Mejora no cuentan y no dejan el control como incumplido.

Puntaje del agente → riesgo (`riesgo_desde_puntaje`): 0–2 Crítico (§6), 3 Moderado,
4 Oportunidad de Mejora, 5 sin observación, sin puntaje = no evaluada (fallo técnico, no se
califica). La falta de evidencia (`sin_evidencia`) es **falta crítica**. En la web, un APROBADO
del chat cuenta como subsanado.

## 7. IA

- Proveedor: `IA_PROVEEDOR` en `.env` (por defecto `azure_openai`; alternativas `anthropic`,
  `openai`). La selección vive SOLO en `config/ia.py` (`get_cliente_ia`, `cliente_dedicado`).
  Sin plan B automático entre proveedores (lo vigilan tests).
- VIRF PLUS: GPT + Claude con árbitro (`config/virf_plus.py`).
- Salvaguardas antes de enviar texto al modelo: `modulos/pii.py` (enmascarado de datos
  personales) y `modulos/inyeccion.py` (neutraliza instrucciones incrustadas).
- Solo Informe: IA opcional (pulir, gráficos, manuales) en `web/services/solo_informe_ia.py`;
  overrides `SOLO_INFORME_IA_PROVEEDOR`, `SOLO_INFORME_DEPLOYMENT`, `SOLO_INFORME_MODELO`.
- RAG: léxico local (`src/rag/`), **sin HuggingFace, sin embeddings, sin vector store**.

## 8. Reglas que ningún cambio puede romper

1. Usar `.venv` (el Python del sistema tiene paquetes prohibidos).
2. No editar `.env`; documentar variables nuevas en `.env.example`.
3. Tests sin coste: IA simulada (mocks/`monkeypatch`), nada de llamadas reales.
4. La selección de proveedor de IA solo en `config/`; la palabra «fallback» no puede aparecer en
   código vivo de `config/ia.py`.
5. `modulos/` no importa `web/`.
6. Sin detección automática de modos: flags explícitos.
7. Sin HuggingFace / ChromaDB / embeddings.
8. La calificación solo en `modulos/reglas_informe.py`.
9. No lanzar POST a ciegas contra la web en marcha (escriben en rutas globales).
10. Toda afirmación del informe debe poder rastrearse a la evidencia (citas verificables).

## 9. Cómo probar

```
.venv\Scripts\python.exe -m pytest -q                 # suite completa (~5 min)
.venv\Scripts\python.exe -m pytest tests/test_X.py -q # un área
```

Tests por área: Solo Informe `test_solo_informe*.py`; plantilla `test_plantilla_bcp.py`;
capturas `test_capturas.py`; informes y reglas regulatorias `test_regresion.py`,
`test_pendientes_auditoria.py`; proveedor de IA `test_estructura_config.py`, `test_azure_mocks.py`,
`test_virf_plus.py`; RAG `test_rag_lexico.py`; estándares `test_estandares_sesion.py`; chat
`test_consolidacion.py`, `test_acta_cierre.py`.

## 10. El grafo del código (graphify)

`graphify-out/` tiene un mapa del código: `GRAPH_REPORT.md` (más de 300 comunidades con nombre en
español, nodos centrales), `graph.json` (consultable) y `graph.html` (visual). Úsalo para elegir
los archivos que subir:

```
graphify query "cómo se genera el informe word"     # recorre el grafo desde la pregunta
graphify explain "concluir()"                        # vecinos de un nodo y su archivo:línea
graphify path "ia_pulir()" "cliente_dedicado()"      # camino entre dos funciones
graphify affected "riesgo_desde_puntaje()"           # qué se ve afectado si lo cambias
graphify update .                                    # actualizar tras cambiar código (sin IA)
```

Limitación: el grafo no une JavaScript con Python (hablan por HTTP). La pantalla llama a
`/api/...`; busca ese prefijo en `web/routers/`.

## 11. Mapa síntoma → skill

| Síntoma | Skill |
|---|---|
| No sé dónde está el problema / error genérico | S01 Diagnosticar un fallo |
| El Word sale mal (estructura, estética, contenido, imágenes) | S02 Informes Word |
| La calificación o el riesgo no es el esperado | S03 Regla de calificación |
| Una pantalla o botón falla, o quiero una pantalla/endpoint nuevo | S04 Web |
| Error 401/503, cambiar de modelo o proveedor de IA | S05 Proveedor de IA |
| El Excel de estándar no se lee o faltan actividades | S06 Estándar de validación |
| La literatura/RAG no aparece o cita mal | S07 RAG léxico |
| Controles de calidad de datos / notebook | S08 Calidad de datos |
| La evidencia no se sube, no se lee o las capturas fallan | S09 Evidencia y capturas |
| El chat no decide bien o la consolidación es incorrecta | S10 Chat de sustentación |
| Necesito acotar qué archivos subir | S11 Usar el grafo |
| Actualizar esta guía o el grafo | S12 Regenerar guía y grafo |
