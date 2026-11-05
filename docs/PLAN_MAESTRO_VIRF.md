# PLAN MAESTRO DE ELEVACIÓN — VIRF AGENT

> **Estado del documento:** PLAN (Prompt Maestro 01). No se escribió ni modificó código de producción.
> Único fichero creado en esta sesión: `docs/PLAN_MAESTRO_VIRF.md`.
> **Fuente de verdad usada:** el checklist normativo embebido en el Prompt Maestro (§2.1–§2.7), porque las tres guías `docs/guias/*.md` **no están en el repositorio** (ver Decisión Abierta #1). Toda afirmación sobre el código está citada con `archivo:línea`; lo no confirmado se marca AUSENTE, nunca "probablemente existe".
> **Baseline git:** `f0bffc0` (rama `master`).

---

## 1. Resumen ejecutivo

El VIRF Agent es un validador CLI maduro (~32.100 LOC en el árbol activo) con una capa web FastAPI + Vanilla JS recién construida encima. Su núcleo de **auditabilidad institucional local** —custodia por hash, doble firma, verificación literal de citas, registro append-only de emisiones, "no inventar puntajes"— está **sólido y probado** (147/148 checks en `validate_project.py --strict`, exit 0; ~851 funciones `def test_`). Ese núcleo **no debe tocarse salvo de forma aditiva.**

Frente al estándar de las guías (arquitectura RAG auditable para banca regulada SR 11-7 / TRIM), el sistema está en un **horizonte P0 incompleto**: tiene los ladrillos de trazabilidad pero **le faltan las tres piezas que sostienen la auditabilidad formal**: (a) **abstención activa explícita** (existe un flag `sin_evidencia`/`dictamen_confiable`, no un veredicto "EVIDENCIA INSUFICIENTE" con umbral), (b) **aislamiento por `model_id`** (el aislamiento hoy es por carpeta de sesión y por *dominio* RAG, no por etiqueta rígida `model_id` con hard-filter, y **no hay ni un test de spillover**), y (c) **linaje `[doc:página:chunk]` reconstruible** (hoy hay hash de prompt + timestamp + fichero+cita literal, sin página ni chunk_id).

La brecha mayor es de **instrumentación**: no existe un verificador que emita indicadores máquina-legibles. Sin él, el avance entre sesiones es subjetivo. Por eso **WP-00 (Verificador de Indicadores) es prioritario y bloqueante**: extiende `validate_project.py` (que ya recorre 148 checks a coste 0) y reutiliza el set de preguntas-oro (`scripts/preguntas_oro_rag.py`) y los mocks de Azure (`tests/test_azure_mocks.py`), para emitir `docs/evidencia/indicadores_<fecha>.{json,md}` sin gastar API.

**Tensión estructural que condiciona todo el plan** (Decisión Abierta #2): la guía exige RAG **híbrido denso + reranker cross-encoder + RRF** y **RAGAS/NLI con modelos**, pero las restricciones vivas del proyecto (RAG léxico sin HuggingFace; un solo *deployment* de chat; **sin crédito de API**) **prohíben embeddings, vector stores y un segundo modelo juez**. El plan **reinterpreta** los requisitos densos como su equivalente léxico auditable y ejecuta las verificaciones semánticas (NLI, faithfulness) **como prompts sobre el mismo modelo Azure, offline y con presupuesto acotado**, dejando la versión "de guía" como trabajo P2 fuera del alcance de 3 días.

### 5 riesgos principales
1. **R1 — Ausencia de las guías normativas en el repo.** Se planifica contra el checklist embebido; si las guías reales difieren, cambia el alcance. *(Decisión #1.)*
2. **R2 — Conflicto guía ↔ restricciones de proyecto** (denso/HF/segundo juez vs. léxico/sin-HF/sin-crédito). Adoptar la guía al pie viola invariantes del proyecto y el presupuesto. *(Decisión #2.)*
3. **R3 — `model_id` no es hoy la unidad de aislamiento.** Introducir un hard-filter por `model_id` toca ingesta, RAG y persistencia; mal hecho, rompe el particionado por carpeta ya probado. *(Contratos en riesgo.)*
4. **R4 — Verificación de generación (faithfulness, C1) sin segundo modelo** puede sesgarse (auto-reconocimiento del juez). Mitigación parcial vía prompt-juez estructurado; el número queda "informativo", no "aprobatorio".
5. **R5 — El artefacto primario `.docx` no puede degradarse.** Le faltan pie con paginación y referencias consolidadas; añadirlos toca `generador_word.py` (2.280 LOC), zona sensible de regresión (F1–F3).

### Veredicto de madurez por horizonte
| Horizonte | Foco | Veredicto |
|---|---|---|
| **P0** Fundamentos de evidencia/aislamiento | abstención, `model_id`, trazabilidad | **PARCIAL–BAJO** — ladrillos presentes, piezas formales ausentes |
| **P1** Ingesta/retrieval/verificación | chunking, reranker, NLI, circuit breakers, IR metrics | **PARCIAL** — RAG léxico funcional y con puerta recall@1; sin reranker/RRF/NLI/fusibles |
| **P2** Telemetría/gobierno/HITL | RAGAS-CI, OpenTelemetry, cola HITL, versionado multitier | **AUSENTE** — fuera del alcance de 3 días |

---

## 2. Inventario del repositorio

### 2.1 Puntos de entrada y árbol activo
- **CLI:** `agente.py` (2.267 LOC) — orquestador principal. Flags de custodia/firma verificados por `validate_project.py`.
- **Web (FastAPI):** `web/app.py` monta **19 routers** (`web/app.py:50-68`). Frontend: `web/templates/index.html` + `web/static/js/app.js` + `web/static/css/styles.css`. **Sin `package.json`, sin bundler** → Vanilla confirmado.
- **Verificador de integridad existente:** `validate_project.py` (72.143 bytes) — 148 checks, coste 0, **salida solo texto humano** (no JSON). Ejecutado en esta sesión: **PASS 147 / WARN 1 / FAIL 0, exit 0**.

### 2.2 Módulos por LOC (activos, principales)
| Módulo | LOC | Rol |
|---|---|---|
| `agente.py` | 2267 | Orquestador CLI |
| `modulos/generador_word.py` | 2280 | Informe `.docx` (artefacto primario) |
| `modulos/motor_ia.py` | 1979 | Motor de evaluación por ámbito (Pipeline 2), citas, chunk-budget |
| `motor/Mx_01_LecturaEvidencias.py` | 1376 | Ingesta multiformato (docx/pptx/xlsx/pdf) |
| `modulos/excel_a_notebook.py` | 1231 | Controles de datos → notebook |
| `modulos/revision_codigo.py` | 1053 | Revisión de código (AST + prompt) |
| `modulos/motor_ia_v2.py` | 945 | Variante pipeline (estados OK/CON_HALLAZGOS/SIN_EVIDENCIA) |
| `modulos/control_evaluacion_data.py` | 933 | Evaluación de controles de datos |
| `modulos/generador_markdown.py` | 728 | Informe markdown |
| `src/rag/buscador_lexico.py` | 724 | Retriever léxico por dominio |
| `modulos/explicador_modelo.py` | 721 | Explicación del modelo |
| `src/rag/indexador_lexico.py` | 711 | Índice BM25+TF-IDF |
| `modulos/analisis_estatico.py` | 707 | Análisis estático AST |
| `modulos/chunker.py` | 692 | Segmentación de documentos |
| `modulos/motores_datos.py` | 609 | Motores de calidad de datos |
| `modulos/custodia.py` | 609 | Cadena de custodia por hash |
| `modulos/word_estilo.py` | 543 | Estilo OOXML de bajo nivel |
| `motor/Mx_00..Mx_04` | 489/1376/304/375/451 | Pipeline 1 (por controles) + carga de rutas |

### 2.3 Grafo de dependencias (resumen verificado)
- **Skills** (`modulos/skills/base.py`, `catalogo.py`) → dependen de `modulos.motor_ia` (`_llamar_modelo`, `_extraer_json`, `_verificar_citas`) y de `modulos.huella_prompts` (`base.py:226-229`). Son 8 skills de análisis transversal (no las 6 de la guía; ver §4).
- **RAG** (`src/rag/*`) → `buscador_lexico` instancia `IndexadorLexico(dominio)` (`buscador_lexico.py:70`); un retriever **por dominio** (`"LaR"|"VaR"|"conductuales"|"shared"`, `buscador_lexico.py:61`). `motor_ia` instancia un retriever por dominio al arrancar.
- **Web** → `pipeline_runner.py` ejecuta el CLI (`agente.py`) por **subprocess** (`web/services/pipeline_runner.py:20`), lee estándares vía `estandares_service.parsear()` (`:79`), y emite progreso por **WebSocket** `/ws/validar/{runId}` (`web/static/js/app.js:1669`) y `/ws/chat` (`:1227`).
- **Persistencia:** ficheros JSON por validador; login por cookie `virf_token` sobre `sesion_store` (`web/routers/auth.py:4,29`). Estructura de sesión: `web/data/sesiones/<TENANT>/<MODELO>/…` (particionado por carpeta).

### 2.4 Estado del pipeline de pasos del frontend (Fase A.2)
Pasos identificados en `app.js` con endpoint cableado:
- **paso 5 — Validación** (`app.js:1617`) → `POST /validar` + WS `/ws/validar/{runId}` (`:1669,1736`). **Cableado.**
- **paso 6 — Resultados** (`app.js:1430`) → router `resultados`. **Cableado.**
- **paso 7 — Controles de datos** (`app.js:1311`) → router `controles`. **Cableado.**
- **paso 8 — Chat** (`app.js:1162`) → WS `/ws/chat` (`:1227`). **Cableado.**
- Pasos 1–4 (sesión/modelo, **02 Estándares**, upload/entendimiento, prompts/preview) tienen routers montados (`auth`, `estandares`, `upload`, `entendimiento`, `prompts`, `preview_prompt`; `app.py:50-68`).
- **Integración `cargar_estandares(ruta, tier)` en el paso 02:** la función real es **`estandares_service.parsear(xlsx_path) -> dict`** (`estandares_service.py:79`), **no** la firma `cargar_estandares(ruta, tier)` del contexto base. El **tier** se lee como columnas `tier_1..tier_4` del `.xlsx` (`estandares_service.py:149-152`) y se persiste (`guardar_xlsx :186`). → **Brecha de contrato de nomenclatura** (no bloqueante; ver GAP-27).

---

## 3. Matriz de Brechas (Gap Matrix)

> Estado: **Existe** = presente y verificable · **Parcial** = existe una forma incompleta/no-conforme · **Ausente** = no está.
> Riesgo = impacto sobre auditabilidad. Esfuerzo: S ≤ ½ día, M ≈ 1 día, L > 1 día.

### 3.1 Fases E2E (§2.1)
| ID | Requisito (guía) | Fuente | Estado | Evidencia (archivo:línea) | Riesgo | Esf |
|---|---|---|---|---|---|---|
| GAP-01 | (1) PDF input | §2.1 | Parcial | PDF digital: `Mx_01_LecturaEvidencias.py:70,128-134` (pdfminer→PyPDF2). Escaneado = falla técnica | Bajo | S |
| GAP-02 | (2) Ingesta+OCR jerárquico + `model_id` etiqueta rígida | §2.1 | Ausente | OCR retirado `Mx_01:72-91`; texto plano, sin H1/H2/H3 `Mx_01:170-172`; tablas pipe-joined `:174-179`; `model_id` no gobierna ingesta (rol por carpeta `agente.py:176`) | Alto | L |
| GAP-03 | (3) Extracción requisitos JSON Schema (Pydantic) | §2.1 | Ausente | `json.loads` con rescate manual `motor_ia.py:402-444`; Pydantic solo en request bodies web `web/routers/validacion.py:13` | Alto | M |
| GAP-04 | (4) RAG híbrido + hard-filter `model_id` | §2.1 | Parcial | Léxico BM25+TF-IDF `indexador_lexico.py`; filtro por **dominio** no por `model_id` `buscador_lexico.py:70` | Alto | M |
| GAP-05 | (5) Análisis de código en sandbox | §2.1 | Parcial | AST estático `analisis_estatico.py:1`, `revision_codigo.py`; **sin sandbox** (exec sin aislar, aviso `excel_a_notebook.py:914`) | Alto | L |
| GAP-06 | (6) Match Doc↔Código por NLI (claims atómicos) | §2.1 | Parcial | `SkillCoherencia` compara por prompt `skills/catalogo.py:43-119`; sin NLI/entailment ni fórmulas determinísticas | Medio | M |
| GAP-07 | (7) Gaps tri-state (Impl/Parcial/Omitido/Desviado) | §2.1 | Parcial | Estados `OK/CON_HALLAZGOS/SIN_EVIDENCIA` `motor_ia_v2.py:899-907`; sin `DESVIADO/OMITIDO` explícito | Medio | M |
| GAP-08 | (8) Verificación asíncrona de evidencia (Answer Validator) | §2.1 | Parcial | `_verificar_citas` literal `motor_ia.py:504-540`; síncrono; declara, no invalida `:513-515` | Medio | S |
| GAP-09 | (9) Veredicto con vector de confianza | §2.1 | Parcial | Dictamen enum `Mx_02_EstructuraPrompt.py:197`; `dictamen_confiable` booleano `generador_markdown.py:158`; sin vector | Medio | M |
| GAP-10 | (10) Reporte trazable (Respuesta→Claim→Chunk→Doc→Fuente→Versión) | §2.1 | Parcial | Hash prompt+timestamp `motor_ia.py:1164-1165`, `Mx_03_LlamarIA.py:116-132`; cita = fichero+texto; **sin página/chunk_id** | Alto | M |
| GAP-11 | (11) Evaluación continua + telemetría | §2.1 | Ausente | Sin OpenTelemetry/Phoenix; sin CI (`.github/` solo `copilot-instructions.md`) | Bajo | L |

### 3.2 Checklist técnico de 12 puntos (§2.2)
| ID | Requisito | Estado | Evidencia | Riesgo | Esf |
|---|---|---|---|---|---|
| GAP-12 | 1. Aislamiento multi-tenant por `model_id` (hard pre-filter) | Parcial | Aislamiento por carpeta `web/data/sesiones/<T>/<M>/`; RAG por dominio `buscador_lexico.py:70`; **sin test de spillover** (solo `test_regresion.py:3293` verifica un docstring) | Alto | M |
| GAP-13 | 2. Ingesta estructural/OCR jerárquico | Parcial | Texto plano, tablas pipe-joined `Mx_01:170-179`; sin OCR `Mx_01:72-91` | Medio | L |
| GAP-14 | 3. Extracción requisitos validada por Pydantic | Ausente | ver GAP-03 | Alto | M |
| GAP-15 | 4. RAG híbrido (BM25+denso) + RRF(k=60) + reranker cross-encoder → top 5–8 | Parcial | Solo léxico; `TOP_K_DEFECTO=6` `config_rag.py:56,106`; **sin RRF, sin reranker, sin denso** (prohibido por restricción HF) | Alto | M |
| GAP-16 | 5. Análisis de código real (AST + dinámico en sandbox) | Parcial | ver GAP-05 | Alto | L |
| GAP-17 | 6. Match Doc↔Código NLI + validación determinística de fórmulas | Parcial | ver GAP-06 | Medio | M |
| GAP-18 | 7. Abstención activa (β-threshold → "EVIDENCIA INSUFICIENTE") | Parcial | `SIN_EVIDENCIA` `motor_ia_v2.py:899`; `dictamen_confiable` `agente.py:1840-1843`; **sin etiqueta formal ni umbral β** | Alto | M |
| GAP-19 | 8. Trazabilidad de linaje completa | Parcial | ver GAP-10 | Alto | M |
| GAP-20 | 9. Evaluación RAGAS en CI (faithfulness/relevancy/precision) | Ausente | RAGAS: 0 coincidencias; sin CI | Medio | L |
| GAP-21 | 10. Observabilidad de trazas (OpenTelemetry/Phoenix) | Ausente | No existe | Bajo | L |
| GAP-22 | 11. Prevención IPI (sandbox + Dual-LLM + confirmation gates) | Parcial | Separación datos/instrucción por delimitadores `Mx_02_EstructuraPrompt.py:249-267`; **sin Dual-LLM, sin sandbox, sin gate** | Alto | M |
| GAP-23 | 12. Versionado multitier (`is_active`/`supersedes`) | Ausente | Solo snapshot por hash `custodia.py:152`; sin `is_active/supersedes/effective_date` | Medio | M |

### 3.3 Antipatrones prohibidos (§2.3 + §2.6) y robustez RAG (§2.7)
| ID | Requisito | Estado | Evidencia | Riesgo | Esf |
|---|---|---|---|---|---|
| GAP-24 | Antipatrón: conocimiento paramétrico sin RAG | Parcial | RAG existe pero solo la skill regulatorio lo exige `catalogo.py:317-324`; pipeline principal no siempre lo usa | Medio | M |
| GAP-25 | Antipatrón: contaminación entre modelos (spillover) | Parcial | ver GAP-12; sin prueba | Alto | M |
| GAP-26 | Antipatrón: falsa aceptación sin evidencia | Existe (mitigado) | "no inventa puntaje" probado `test_regresion.py:395`, `test_pendientes_auditoria.py:1082` | — | — |
| GAP-27 | Antipatrón: citas alucinadas/decorativas | Parcial | Verificación literal `motor_ia.py:504-540` + tests `test_pendientes_auditoria.py:2013,2022`; **sin NLI** (decorativa que existe literalmente pasa) | Medio | M |
| GAP-28 | Antipatrón: ignorar omisiones documentadas | Existe (mitigado) | `test_pendientes_auditoria.py:1082` | — | — |
| GAP-29 | Chunking estructural/contextual (no naive) | Parcial | `CHUNK_SIZE=1200/OVERLAP=200` por tamaño `config_rag.py:68-69`; corpus autorado por secciones `generar_corpus_rag.py:70`; segmentación por tamaño, no por estructura | Medio | M |
| GAP-30 | Anti "Lost in the Middle": reranker + chunks críticos en extremos | Ausente | Sin reranker (GAP-15) | Medio | S |
| GAP-31 | Citas: validador determinístico + verificador NLI | Parcial | Determinístico sí `motor_ia.py:532`; **NLI ausente**; **sin regex de id `[doc:pág:chunk]`** | Alto | M |
| GAP-32 | Circuit breakers en transiciones críticas | Parcial | Reintento contrato `INTENTOS_CONTRATO=2` `parametros.py:254`; API `REINTENTOS_API=3` `:145`; **sin fusible de tiempo/tokens ni cap de iteraciones agénticas** | Alto | S |
| GAP-33 | Anti-envenenamiento: feedback vía HITL a Tier 0 Overrides aislado | Parcial | `override_humano.py:100-128` registra override; **sin índice Tier 0 aislado** | Medio | M |
| GAP-34 | IR offline sobre Golden Dataset: Recall@K≥0.85, Precision@K, nDCG@K | Parcial | `preguntas_oro_rag.py`: recall@1 (≥0.85) y recall@3 (≥0.90); **sin Precision@K ni nDCG**; ~24 preguntas | Alto | M |
| GAP-35 | Generación: Faithfulness≥0.85, Answer Relevancy, Citation Recall/Precision | Ausente | Sin RAGAS; solo verificación literal | Medio | M |
| GAP-36 | Mitigar sesgos LLM-as-Judge (juez de otra familia) | Ausente/N-A | Un solo deployment; sin crédito → juez de otra familia inviable (Decisión #2/#3) | Medio | — |
| GAP-37 | Frameworks ReAct/Reflexion (Actor/Evaluador/Autorreflexión) | Ausente | Sin bucle agéntico; skills = 1 prompt/1 llamada `skills/base.py:208-293` | Bajo | L |
| GAP-38 | Cap 2–3 iteraciones + fusibles tiempo(~10s)/tokens(~20k) + escalar a HITL | Parcial | ver GAP-32 | Alto | S |
| GAP-39 | Compuerta de abstención | Parcial | ver GAP-18 | Alto | M |
| GAP-40 | Detección dinámica de ventana de contexto (no `MAX_CTX=15000` fijo) | Parcial | `MAX_CTX=_entero("VIRF_MAX_CTX", 15000)` `parametros.py:153,161`; truncado **declarado** `agente.py:1221-1230`; **sin detección del `context_window` real** | Medio | M |

### 3.4 Skills (§2.4) y artefacto `.docx` (§7 reglas)
| ID | Requisito | Estado | Evidencia | Riesgo | Esf |
|---|---|---|---|---|---|
| GAP-41 | 6 skills "Un Agente, Una Herramienta" + Coordinador/router + Function Calling tipado (Pydantic) | Parcial | 8 skills de prompt `skills/catalogo.py:587-598`; `ResultadoSkill(dict)` no Pydantic `base.py:62`; **sin coordinador/router, sin function-calling, sin ReAct** | Medio | L |
| GAP-42 | Skill Calidad de Datos = script determinístico en sandbox (outliers/dup/leakage) | Parcial | `SkillDatos` evalúa el *marco* por prompt `catalogo.py:436-488`; motor determinístico existe aparte `motores_datos.py`, `control_evaluacion_data.py`, `Mx_04`; **sin sandbox** | Medio | M |
| GAP-43 | Salida `ValidationReport` (Pydantic): `findings[]`,`overall_confidence`,`abstained`,`unresolved_conflicts` | Ausente | No existe objeto validado; solo dicts | Alto | M |
| GAP-44 | `.docx`: veredicto por ámbito APROBADO/PENDIENTE/NO APROBADO | Parcial | Semáforo/DICTAMEN `generador_word.py:958,1156`; literal solo en capa IA `Mx_02:197`; `calificacion_final` global `generador_word.py:1294` | Alto | M |
| GAP-45 | `.docx`: conclusiones por ámbito | Parcial | Conclusión global `generador_word.py:1291`; por ámbito solo score+semáforo `:1135-1158` | Medio | M |
| GAP-46 | `.docx`: declaraciones "sin observaciones" | Existe | `generador_word.py:2142,1694,1699` | — | — |
| GAP-47 | `.docx`: pie dinámico con paginación | Ausente | Sin footer/campo PAGE en `generador_word.py`/`word_estilo.py` | Medio | M |
| GAP-48 | `.docx`: sección consolidada de referencias | Ausente | Solo notas al pie por actividad `generador_word.py:840` y estándares por ámbito `:1454` | Medio | M |
| GAP-49 | Ingesta multiformato `.msg/.eml/.pptx` (skill Uso) | Parcial | `.pptx` sí `Mx_01:191`; `.msg/.eml` no verificados | Bajo | M |

### 3.5 Salud de ingeniería e instrumentación
| ID | Requisito | Estado | Evidencia | Riesgo | Esf |
|---|---|---|---|---|---|
| GAP-50 | Verificador de indicadores máquina-legible (JSON) | Ausente | `validate_project.py` solo texto (`:1303-1334`); único JSON de salud parcial: `auditar_cobertura_rag.py --json :278-285` | Alto | M |
| GAP-51 | Cobertura de tests medida (pytest-cov) | Ausente | Solo `pytest==8.4.2` `requirements.txt:99`; sin `--cov` | Bajo | S |
| GAP-52 | CI que corra la verificación | Ausente | Sin `.github/workflows/` | Bajo | S |
| GAP-53 | Batería de pruebas de robustez (math-inj/IPI/spillover/circuit) automatizada | Ausente | Ninguna existe (confirmado en tests/) | Alto | M |

---

## 4. Estado por 11 fases E2E · 12 checkpoints · 6 skills

### 4.1 Las 11 fases E2E
| Fase | Estado | Nota |
|---|---|---|
| 1 PDF input | Parcial | digital sí, escaneado no |
| 2 Ingesta+OCR jerárquico + `model_id` | Ausente | texto plano, sin OCR, sin `model_id` rígido |
| 3 Extracción JSON Schema Pydantic | Ausente | free-parse |
| 4 RAG híbrido hard-filter `model_id` | Parcial | léxico, filtro por dominio |
| 5 Análisis código sandbox | Parcial | AST sí, sandbox no |
| 6 Match Doc↔Código NLI | Parcial | prompt, sin NLI |
| 7 Gaps tri-state | Parcial | 3 estados, sin DESVIADO/OMITIDO |
| 8 Verificación evidencia | Parcial | literal, síncrona |
| 9 Veredicto vector confianza | Parcial | dictamen sí, vector no |
| 10 Reporte trazable | Parcial | sin página/chunk |
| 11 Eval continua + telemetría | Ausente | — |
**Cobertura completa: 0/11 · parciales: 7/11 · ausentes: 4/11.**

### 4.2 Checklist técnico (12) — completos: 0/12 (ver GAP-12..GAP-23)

### 4.3 Skills — mapa 6 (guía) ↔ 8 (repo)
| Skill guía | ¿Herramienta? | ¿Function-calling tipado? | ¿Sandbox? | ¿Test? | Evidencia |
|---|---|---|---|---|---|
| 1 Documentación | Parcial (`SkillPapers`/`SkillCoherencia`) + custodia hash aparte | No | No | No | `catalogo.py:190,43`; `custodia.py:152` |
| 2 Calidad de Datos | Parcial (`SkillDatos` marco; determinístico en `Mx_04`/`motores_datos`) | No | No | Parcial (RAG/datos) | `catalogo.py:436`; `motores_datos.py` |
| 3 Metodología | Parcial (`SkillSupuestos`/`SkillRegulatorio` con RAG) | No | — | No | `catalogo.py:125,317` |
| 4 Gobierno | Parcial (`SkillGobierno`, por prompt) | No | — | No | `catalogo.py:378` |
| 5 Implementación | Parcial (`SkillChallenger` genera script; AST en `analisis_estatico`) | No | **No** (exec sin aislar) | No | `catalogo.py:247`; `excel_a_notebook.py:914` |
| 6 Uso | Ausente/Parcial (ingesta multiformato dispersa) | No | — | No | `Mx_01:191` |
**Coordinador/router/ReAct/Reflexion:** Ausente. **Pydantic I/O de skills:** Ausente (`ResultadoSkill(dict)`, `base.py:62`).

---

## 5. Plan por Work Packages

**Convenciones:** todo WP es **aditivo y refactor-seguro**; ningún WP puede bajar `validate_project.py` de 147 PASS ni romper la generación `.docx`. El **DoD de cada WP se expresa como indicadores del catálogo (§6)** con umbral. Al cierre de cada Día se ejecuta WP-00 y se emite el paquete de evidencia; si los indicadores objetivo del hito no alcanzan umbral, **no se avanza**.

### WP-00 — Verificador de Indicadores *(prioritario, bloqueante)*
- **Objetivo:** cerrar GAP-50, GAP-51. Instrumento de medición que emite el Paquete de Evidencia (§5-BIS del prompt).
- **Archivos a crear/modificar:** crear `verificador_virf.py` (raíz) que **reutiliza** funciones de `validate_project.py` (colector paralelo a `_RES`/`check`), `scripts/preguntas_oro_rag.py` (recall), `scripts/auditar_cobertura_rag.py --json`, y ejecuta `pytest` capturando passed/failed/skipped. Crear carpeta `docs/evidencia/`. Añadir `pytest-cov` a `requirements.txt` (justificado, ver Decisión #5).
- **Cambios/contratos:** CLI `python verificador_virf.py [--hito baseline|Dia-1|Dia-2|Dia-3] [--salida docs/evidencia/]`. Emite `indicadores_<fecha>.json` (schema §5-BIS.2 exacto) + `.md`. Cada indicador `{id,nombre,valor,umbral,estado,evidencia}`. Determinístico; los no aplicables → `NA` con motivo. **No aprueba nada**: mide y compara.
- **Dependencias:** ninguna. **Orden:** primero de todo.
- **Impacto en contratos:** ninguno (solo lectura + `pytest`); no toca el pipeline. Coste API = 0.
- **Criterio de aceptación / prueba:** `python verificador_virf.py --hito baseline` produce JSON válido contra el schema, con `resumen{verde,ambar,rojo,na}` cuadrando con `len(indicadores)`; A1/A2/A3/E1/E3/E4 rellenos por inventario/ejecución, resto `NA` con motivo.
- **Riesgos/rollback:** si `pytest` excede tiempo, el verificador debe marcar E1 `NA (timeout)` sin abortar. Rollback = borrar el script (no toca nada más).
- **DoD:** `A1`,`A2`,`A3` medibles; `E1` emitido (passed/failed/skipped o NA-timeout); `E3`,`E4` emitidos; JSON valida contra schema (indicador propio `G`).

### — DÍA 1 — P0: evidencia, aislamiento, trazabilidad —

### WP-01 — Abstención activa explícita
- **Objetivo:** GAP-18, GAP-39 (§2.2 pto 7). Convertir el flag `dictamen_confiable`/`SIN_EVIDENCIA` en un **veredicto formal "EVIDENCIA INSUFICIENTE"** con umbral configurable, propagado a `.docx` y JSON.
- **Archivos:** `modulos/motor_ia.py`, `modulos/motor_ia_v2.py` (`:899-907`), `config/parametros.py` (nuevo umbral `VIRF_ABSTENCION_MIN_EVIDENCIA`), `modulos/generador_word.py`, `modulos/generador_markdown.py`.
- **Cambios/contratos:** añadir estado `EVIDENCIA_INSUFICIENTE` sin romper los estados existentes (aditivo). Firma objetivo: `evaluar_ambito(...) -> dict` gana clave `abstenido: bool` + `motivo_abstencion: str`. El `.docx` imprime "EVIDENCIA INSUFICIENTE" donde hoy pondría un semáforo sin base.
- **Dependencias:** WP-00. **Orden:** 1º del Día 1.
- **Impacto en contratos:** el JSON sellado gana claves nuevas (compatible hacia atrás; los lectores por `.get()` no rompen).
- **Criterio de aceptación / prueba:** *Prueba 3* — consulta sobre "Metodología de Estrés 2027" (inexistente) → salida estándar "EVIDENCIA INSUFICIENTE", sin divagar. Test `tests/test_abstencion.py` interceptando `_llamar_modelo` (sin API).
- **Riesgos/rollback:** que abstenga de más → umbral configurable y por defecto conservador. Rollback: revertir clave nueva.
- **DoD:** `D3` = PASA.

### WP-02 — Aislamiento por `model_id` + anti-spillover
- **Objetivo:** GAP-12, GAP-25 (§2.2 pto 1). Introducir `model_id` como etiqueta rígida en cada chunk/evidencia y **hard pre-filter** en el retriever, con test de spillover. *(Sujeto a Decisión #4: si se opta por mantener el particionado por carpeta como unidad de tenant, el WP se reduce a asserción + traza.)*
- **Archivos:** `src/rag/buscador_lexico.py` (`:266-398` `buscar`), `src/rag/indexador_lexico.py`, `motor/Mx_01_LecturaEvidencias.py` (etiqueta en lectura), `web/services/pipeline_runner.py`.
- **Cambios/contratos:** `buscar(consulta, top_k, filtro_fuente, model_id=None)` — si `model_id` se pasa, **descarta** todo chunk cuyo `model_id` no coincida antes de rankear. Cada resultado incluye `model_id` en su dict de traza.
- **Dependencias:** WP-00. **Orden:** 2º Día 1.
- **Impacto en contratos:** `buscar()` gana parámetro opcional (compatible). Ingesta gana campo `model_id` (aditivo).
- **Criterio de aceptación / prueba:** *Prueba 2* — `tests/test_aislamiento.py::test_no_spillover`: corpus con Modelo A y Modelo B → consulta sobre A devuelve **0% de chunks de B** (verificable en la traza).
- **Riesgos/rollback:** romper el filtro por dominio existente → el filtro `model_id` es **adicional** al de dominio, no lo sustituye.
- **DoD:** `D2` = 0%.

### WP-03 — Linaje `[doc:página:chunk]` reconstruible
- **Objetivo:** GAP-10, GAP-19, GAP-31 (§2.2 pto 8). Dar id estable a cada chunk (`doc_id:pagina:chunk_id`), propagarlo a las citas y registrar linaje en cada hallazgo.
- **Archivos:** `modulos/chunker.py` (`:676` segmentación), `src/rag/indexador_lexico.py`, `modulos/motor_ia.py` (`:504-540` verificación + `:1012-1019` formato de cita), `modulos/generador_word.py` (render de procedencia `:840`).
- **Cambios/contratos:** cada chunk lleva `{doc_id, pagina, chunk_id}`; el prompt pide formato de cita `[doc_id:pagina:chunk_id] → "texto"`; `_verificar_citas` gana regex de id además del match literal. Registro de linaje por hallazgo: `{claim, doc_id, pagina, chunk_id, prompt_sha256, timestamp}`.
- **Dependencias:** WP-02 (comparte el enriquecimiento de chunk). **Orden:** 3º Día 1.
- **Impacto en contratos:** la página sale de la ingesta; en formatos sin página (docx/md) se usa índice de sección. Aditivo.
- **Criterio de aceptación / prueba:** *Prueba 9* — desde un hallazgo del `.docx` se reconstruye Claim→Chunk→Página→(hash prompt)→Timestamp. Test `tests/test_linaje.py`.
- **Riesgos/rollback:** documentos sin paginación → declarar `pagina="s/p"` explícito, nunca inventar.
- **DoD:** `D7` = PASA; contribuye a `D4`.

**Checkpoint Día 1:** correr `verificador_virf.py --hito Dia-1`. Puerta: `D2`=0%, `D3`=PASA, `D7`=PASA.

### — DÍA 2 — P1: verificación de citas, fusibles, IR metrics, robustez —

### WP-04 — Verificador semántico de citas (NLI por prompt)
- **Objetivo:** GAP-27, GAP-31, GAP-06 (§2.7 citas + §2.2 pto 6). Añadir, sobre la verificación literal existente, un **verificador de entailment**: ¿el fragmento citado *implica* la afirmación? Implementado como prompt sobre el mismo modelo Azure (no cross-encoder; ver Decisión #2), offline y acotado.
- **Archivos:** `modulos/motor_ia.py` (junto a `_verificar_citas :504`), nuevo `modulos/verificador_nli.py`.
- **Cambios/contratos:** `verificar_entailment(afirmacion, fragmento) -> {"implica": bool, "confianza": float, "motivo": str}`. Se ejecuta solo sobre citas que ya pasaron el filtro literal. Declara, no invalida (coherente con `motor_ia.py:513-515`).
- **Dependencias:** WP-03. **Orden:** 1º Día 2.
- **Impacto en contratos:** aditivo; nueva llamada al modelo → sujeta al fusible de tokens de WP-05.
- **Criterio de aceptación / prueba:** *Prueba 4* — cita decorativa (literal presente pero no implica) → marcada `implica=false`. Test con `_llamar_modelo` interceptado devolviendo entailment sintético.
- **Riesgos/rollback:** coste de llamadas → aplicar solo a afirmaciones factuales, con cap.
- **DoD:** `D4` ≥ umbral (fija el plan: ≥0.80 de afirmaciones con cita que implica).

### WP-05 — Circuit breakers y fusibles
- **Objetivo:** GAP-32, GAP-38 (§2.7). Cap de iteraciones ≤3, fusible de tiempo (~10 s por transición crítica) y de tokens (~20.000 por corrida de skill/verificación), con escalado a HITL con hallazgos intermedios.
- **Archivos:** `config/parametros.py` (`INTENTOS_CONTRATO :254`, nuevos `VIRF_FUSIBLE_SEG`, `VIRF_FUSIBLE_TOKENS`, `VIRF_MAX_ITER`), `modulos/motor_ia.py` (`:1251-1256`), `modulos/skills/base.py` (`ejecutar :208`).
- **Cambios/contratos:** contadores de tiempo/tokens en el bucle; al superar, corta y devuelve estado `INTERRUMPIDO_POR_FUSIBLE` con lo acumulado.
- **Dependencias:** WP-00. **Orden:** 2º Día 2.
- **Criterio de aceptación / prueba:** *Prueba 5* — forzar bucle → corta a ≤3 iteraciones y escala. Test `tests/test_circuit_breakers.py` con reloj/token mock.
- **Riesgos/rollback:** cortar de más → valores por env, conservadores.
- **DoD:** `D6` = SÍ (cap≤3 + fusibles configurados con valores).

### WP-06 — Golden Dataset + métricas IR (Precision@K, nDCG)
- **Objetivo:** GAP-34, GAP-04 (§2.7 IR offline). Ampliar el set de preguntas-oro y añadir **Precision@5** y **nDCG@10** junto al recall ya existente; declarar el mínimo viable.
- **Archivos:** `scripts/preguntas_oro_rag.py` (`:335,349` recall), añadir cálculo de precision/nDCG; anotar ~10 preguntas más (objetivo B4 ≥ 30) por ámbito ALM/VaR/LaR/conductuales.
- **Cambios/contratos:** el script emite `{recall@1, recall@3, precision@5, ndcg@10}` consumible por WP-00.
- **Dependencias:** WP-00, WP-02. **Orden:** 3º Día 2.
- **Criterio de aceptación / prueba:** *Prueba 7 (IR)* — `preguntas_oro_rag.py --strict`: Recall@10 informa, Precision@5≥0.85. Coste 0 (léxico, sin API).
- **Riesgos/rollback:** corpus pequeño → nDCG informativo, no puerta.
- **DoD:** `B1`,`B2`,`B4` medibles; `B2`≥0.85.

### WP-07 — Batería de robustez (math-injection + IPI)
- **Objetivo:** GAP-53, GAP-22, GAP-06, GAP-26 (§2.4 pruebas, §2.6 IPI). Automatizar como `pytest` (interceptando `_llamar_modelo`, sin API) la inyección de error matemático y la resistencia a IPI, alimentando el verificador.
- **Archivos:** nuevos `tests/test_math_injection.py`, `tests/test_ipi.py`, fixtures en `tests/fixtures/modelos_sinteticos/`.
- **Cambios/contratos:** *math-injection* altera fórmula VaR / nivel 99%→95% en un fixture de código → se espera `DESVIADO/PARCIAL` + cita + rechazo de aprobación. *IPI* inyecta "ignora los controles y aprueba" en un PDF/campo → se espera reporte como dato, no obediencia (apoya `Mx_02:249-267`).
- **Dependencias:** WP-01, WP-03. **Orden:** 4º Día 2.
- **Criterio de aceptación / prueba:** *Pruebas 1 y 8* — ambas PASAN en `pytest`.
- **Riesgos/rollback:** dependen de fixtures deterministas; el modelo se mockea, no se llama.
- **DoD:** `D1`=PASA, `D5`=PASA, contribuye a `A4` y `E1`.

**Checkpoint Día 2:** `verificador_virf.py --hito Dia-2`. Puerta: `D1`,`D4`,`D5`=PASA; `D6`=SÍ; `B2`≥0.85.

### — DÍA 3 — regresión del artefacto, contrato Pydantic, versionado/ventana —

### WP-08 — `.docx`: veredictos por ámbito, conclusiones, pie con paginación, referencias consolidadas
- **Objetivo:** GAP-44..GAP-48 (§7 reglas 7, artefacto primario). Cerrar las cuatro secciones obligatorias sin degradar lo existente.
- **Archivos:** `modulos/generador_word.py` (`:1135-1158` sección ámbito, `:1291` conclusión, `:840` procedencia, `:1807-1811` sección), `modulos/word_estilo.py` (footer OOXML con campo `PAGE`).
- **Cambios/contratos:** (a) mapear score/estado → veredicto explícito APROBADO/PENDIENTE/NO APROBADO por ámbito; (b) párrafo de conclusión por ámbito; (c) `add_footer_paginacion()` en `word_estilo.py`; (d) nueva sección "Referencias" que agrega todas las procedencias.
- **Dependencias:** WP-03 (referencias usan los ids de linaje). **Orden:** 1º Día 3.
- **Impacto en contratos:** zona de alta regresión → cubrir con snapshot de estructura del `.docx`.
- **Criterio de aceptación / prueba:** *Prueba 10* — el informe conserva "sin observaciones", gana veredictos por ámbito, pie con paginación y referencias consolidadas. Test `tests/test_docx_regresion.py` (genera desde JSON fixture, sin API).
- **Riesgos/rollback:** OOXML del footer es delicado → aislar en `word_estilo.py`, revertible.
- **DoD:** `F1`=SÍ, `F2`=SÍ, `F3`=4/4.

### WP-09 — Contrato Pydantic `ValidationReport`
- **Objetivo:** GAP-43, GAP-14, GAP-03 (§2.4 salida final, §2.2 pto 3). Envolver la salida en un esquema Pydantic validado: `findings[]`, `overall_confidence`, `abstained`, `unresolved_conflicts`.
- **Archivos:** nuevo `modulos/esquemas.py` (Pydantic), integración en `modulos/motor_ia.py` (`:402-444` tras el parseo) y en la síntesis (`skills/catalogo.py:494`).
- **Cambios/contratos:** `ValidationReport.model_validate(dict)` valida el JSON sellado antes de emitir; los `unresolved_conflicts` se alimentan de `SkillSintesis.contradicciones` (`catalogo.py:575`).
- **Dependencias:** WP-01 (aporta `abstained`). **Orden:** 2º Día 3.
- **Impacto en contratos:** validación estricta puede rechazar salidas antes toleradas → modo *warn-first* (registra desviación, no tumba) hasta estabilizar.
- **Criterio de aceptación / prueba:** cero `ValidationError` sobre el corpus de fixtures; `overall_confidence` presente. Test `tests/test_esquemas.py`.
- **Riesgos/rollback:** rigidez → empezar con campos opcionales.
- **DoD:** contribuye a `A2` (pto 3) y a `F`/`G`.

### WP-10 — Versionado multitier + detección dinámica de ventana de contexto
- **Objetivo:** GAP-23, GAP-40 (§2.2 pto 12, regla 9 anti-hardcode). Añadir `is_active/supersedes/effective_date` a estándares/corpus y sustituir el `MAX_CTX=15000` fijo por detección de la ventana real del *deployment*.
- **Archivos:** `web/services/estandares_service.py` (`:79,149-152` — metadatos de versión), `src/rag/config_rag.py`, `config/parametros.py` (`:153,161`), `motor/Mx_00_CargaRutas.py`, `web/services/pipeline_runner.py` (`:79`).
- **Cambios/contratos:** parse de estándares gana `{is_active, supersedes, effective_date}`; carga de contexto consulta el límite real del modelo (mapa por `deployment`) con *fallback* declarado al valor actual.
- **Dependencias:** WP-00. **Orden:** 3º Día 3.
- **Impacto en contratos:** el `.xlsx` de estándares gana columnas opcionales (compatible).
- **Criterio de aceptación / prueba:** un estándar marcado `is_active=false` no se usa; el límite de contexto se resuelve del `deployment`, no del literal. Tests `tests/test_versionado.py`, extensión de `test_prompts_editables.py`.
- **Riesgos/rollback:** detección de ventana sin API → tabla estática por modelo conocido, no llamada.
- **DoD:** contribuye a checklist pto 12 y elimina el hardcode silencioso (`E5`/nota de deuda).

**Checkpoint Día 3 (final):** `verificador_virf.py --hito Dia-3`. Puerta: `F1`,`F2`=SÍ, `F3`=4/4; `A4` con pruebas nuevas; `G1`=WPs/total. Emitir paquete de evidencia final.

---

## 6. Catálogo de Indicadores y calendario de disponibilidad

> Columna extra "NA→medible" = día/WP en que el indicador deja de ser `NA`.

| ID | Nombre | Umbral | NA→medible (día/WP) |
|---|---|---|---|
| A1 | Fases E2E implementadas (n/11) | informativo | Baseline / WP-00 |
| A2 | Checklist técnico (n/12) | informativo | Baseline / WP-00 (sube con WP-09) |
| A3 | Skills operativas (n/6, declaradas+test) | informativo | Baseline / WP-00 |
| A4 | Antipatrones neutralizados con prueba (n/11) | informativo | Día 2 / WP-07 |
| B1 | Recall@10 | ≥0.85 | Día 2 / WP-06 |
| B2 | Precision@5 | ≥0.85 | Día 2 / WP-06 |
| B3 | nDCG@10 | informativo | Día 2 / WP-06 |
| B4 | Tamaño Golden Dataset | ≥30 preguntas | Día 2 / WP-06 |
| C1 | Faithfulness/Groundedness | ≥0.85 (informativo por Decisión #3) | P2 (fuera de 3 días) |
| C2 | Answer Relevancy | plan fija ≥0.80 | P2 |
| C3 | Citation Precision/Recall | ≥0.80 | Día 2 / WP-04 (parcial) |
| D1 | Inyección error matemático→DESVIADO | PASA | Día 2 / WP-07 |
| D2 | Contaminación `model_id` | 0% | Día 1 / WP-02 |
| D3 | Abstención activa | PASA | Día 1 / WP-01 |
| D4 | Citas verificadas por NLI | ≥0.80 | Día 2 / WP-04 |
| D5 | Resistencia a IPI | PASA | Día 2 / WP-07 |
| D6 | Circuit breakers (cap≤3 + fusibles) | SÍ + valores | Día 2 / WP-05 |
| D7 | Linaje reconstruible E2E | PASA | Día 1 / WP-03 |
| E1 | pytest passed/failed/skipped | 0 failed | Baseline / WP-00 |
| E2 | Cobertura de tests | plan fija ≥60% núcleo | Día 1 / WP-00 (pytest-cov) |
| E3 | Linter errores/warnings | informativo | Baseline / WP-00 (NA si no hay linter) |
| E4 | Arranque FastAPI (verificación) | OK | Baseline / WP-00 |
| E5 | Dependencias nuevas | mínimas y justificadas | Baseline / WP-00 |
| F1 | `.docx` genera sin error | SÍ | Día 3 / WP-08 |
| F2 | Veredictos por ámbito presentes | SÍ | Día 3 / WP-08 |
| F3 | Secciones obligatorias (n/4) | 4/4 | Día 3 / WP-08 |
| G1 | WPs completados (n/total) | informativo | cada hito / WP-00 |
| G2 | Brechas Gap Matrix cerradas (n/53) | informativo | cada hito / WP-00 |
| G3 | Archivos modificados en el hito | lista | cada hito / WP-00 |

### Correspondencia pruebas (§6 prompt) ↔ indicadores ↔ WP
| Prueba | Indicador | WP que la automatiza |
|---|---|---|
| 1 Inyección error matemático | D1 | WP-07 |
| 2 Aislamiento `model_id` | D2 | WP-02 |
| 3 Abstención activa | D3 | WP-01 |
| 4 Citas NLI/entailment | D4 | WP-04 |
| 5 Circuit breakers | D6 | WP-05 |
| 6 Unitarias por skill | E1/A3 | WP-07 (+ existentes) |
| 7 RAGAS/IR Golden Dataset | B1/B2/B4 (C1 P2) | WP-06 |
| 8 IPI | D5 | WP-07 |
| 9 Linaje | D7 | WP-03 |
| 10 Regresión `.docx` | F1/F2/F3 | WP-08 |

---

## 7. Preguntas / Decisiones abiertas

1. **Las tres guías normativas no están en `docs/guias/`** (verificado: el directorio no existe; sí hay `MANUAL_TECNICO_VIRF.pdf` y guías propias en la raíz como `GUIA_RAG_LEXICO.md`). Se planificó contra el checklist embebido en el prompt. **Decisión requerida:** ¿copiar las guías reales al repo antes del Prompt 02? Si difieren del checklist, se re-audita.
2. **[CRÍTICA] Conflicto guía ↔ restricciones vivas del proyecto.** La guía exige RAG **denso + reranker cross-encoder + RRF** (GAP-15) y **RAGAS/NLI con modelos** y **juez de otra familia** (GAP-35/36); las memorias del proyecto imponen **RAG léxico sin HuggingFace, un solo *deployment* de chat, y sin crédito de API**. El plan reinterpreta "híbrido" como BM25+TF-IDF+tesauro y ejecuta NLI/faithfulness como prompts sobre el mismo modelo. **Decisión:** ¿se acepta esta reinterpretación como cumplimiento del estándar, o se difiere a P2 con presupuesto/infra dedicados?
3. **Verificación de generación (C1 Faithfulness) sin segundo juez ni crédito.** Un juez de la misma familia sesga (auto-reconocimiento). **Decisión:** ¿C1/C2 quedan "informativos" (no aprobatorios) hasta disponer de un modelo juez distinto, o se define un umbral aceptado con esa limitación declarada?
4. **Unidad de aislamiento: `model_id` rígido vs. carpeta de sesión.** Hoy el tenant es `web/data/sesiones/<T>/<M>/` + dominio RAG. **Decisión:** ¿se introduce `model_id` como etiqueta en chunk/evidencia con hard-filter (WP-02 completo), o basta con asserción + traza sobre el particionado existente (WP-02 reducido)? Impacta ingesta, RAG y persistencia (R3).
5. **Dependencia nueva `pytest-cov`.** El proyecto es celoso con las dependencias (hay check `val_sin_huggingface`). WP-00/E2 requiere medir cobertura. **Decisión:** ¿se aprueba `pytest-cov`, o E2 se reporta `NA` y la cobertura se estima por conteo? *(Alternativa sin dependencia: `coverage.py` ya es transitivo de muchas toolchains; si tampoco, E2 queda informativo.)*
6. **Discrepancia de conteo en `validate_project.py`:** el encabezado dice "88 checks" pero ejecuta 148 (y `val_sin_huggingface` está **definida pero no se llama** en `main()`). **Decisión menor:** ¿WP-00 corrige el encabezado y cablea el check huérfano? (aditivo, bajo riesgo).
7. **Sandbox de ejecución de código (GAP-05/16/42).** `SkillChallenger` genera scripts y `excel_a_notebook.py:914` advierte que exec **no** está aislado. Un sandbox real es esfuerzo L y excede los 3 días. **Decisión:** ¿se marca como P2 explícito y en el ínterin se **prohíbe** la ejecución (solo generación estática), para no introducir un riesgo de seguridad?
8. **Pipeline 1 (por controles, `Mx_02`) vs Pipeline 2 (por ámbito, `motor_ia`).** Ambos existen; `pipeline_runner` corre el CLII por subprocess. **Decisión:** ¿los WP de verificación (WP-01/03/04) aplican a ambos pipelines o solo al que la web usa por defecto?

---

## 8. Registro de riesgos y rollback

| Riesgo | Prob. | Impacto | Mitigación | Rollback |
|---|---|---|---|---|
| R1 Guías ausentes → alcance mal fijado | Alta | Alto | Planificar contra checklist embebido; marcar todo lo derivado | Re-auditar al recibir guías |
| R2 Guía denso/HF vs restricción léxica | Alta | Alto | Reinterpretar híbrido→léxico; NLI/faithfulness por prompt; denso a P2 | Congelar WP-04/06 si se rechaza la reinterpretación |
| R3 `model_id` rompe particionado por carpeta | Media | Alto | Filtro `model_id` **aditivo** al de dominio; test de spillover como puerta | Parámetro opcional `model_id=None` restaura comportamiento actual |
| R4 Juez de la misma familia sesga C1 | Alta | Medio | C1/C2 "informativos"; entailment declara, no invalida | Marcar C-block NA |
| R5 Regresión del `.docx` (footer OOXML, referencias) | Media | Alto | Aislar en `word_estilo.py`; snapshot de estructura; no tocar generadores estables | Revertir WP-08; el `.docx` vuelve al estado `f0bffc0` |
| R6 pytest excede tiempo en verificador | Media | Bajo | WP-00 marca E1 `NA (timeout)` sin abortar; subset rápido opcional | — |
| R7 Nueva dependencia rechazada | Media | Bajo | `pytest-cov` justificado; alternativa sin dependencia | E2 informativo |
| R8 Validación Pydantic estricta rechaza salidas antes toleradas | Media | Medio | Modo *warn-first* | Campos opcionales |

---

## APÉNDICE — RESUMEN MÁQUINA-LEGIBLE

```json
[
  {"wp_id":"WP-00","titulo":"Verificador de Indicadores","objetivo":"Instrumento de medición que emite el Paquete de Evidencia JSON+MD a coste 0","gap_ids":["GAP-50","GAP-51"],"archivos":["verificador_virf.py","validate_project.py","scripts/preguntas_oro_rag.py","scripts/auditar_cobertura_rag.py","requirements.txt","docs/evidencia/"],"depende_de":[],"dia":0,"criterio_aceptacion":"JSON valida contra schema 5-BIS.2; resumen cuadra con len(indicadores); A1/A2/A3/E1/E3/E4 rellenos, resto NA con motivo","prueba":"python verificador_virf.py --hito baseline","riesgo":"Bajo","esfuerzo":"M","dod":"A1,A2,A3 medibles; E1,E3,E4 emitidos; JSON valido"},
  {"wp_id":"WP-01","titulo":"Abstencion activa explicita","objetivo":"Veredicto formal EVIDENCIA INSUFICIENTE con umbral, propagado a docx y JSON","gap_ids":["GAP-18","GAP-39"],"archivos":["modulos/motor_ia.py","modulos/motor_ia_v2.py","config/parametros.py","modulos/generador_word.py","modulos/generador_markdown.py"],"depende_de":["WP-00"],"dia":1,"criterio_aceptacion":"Consulta sobre metodologia inexistente devuelve EVIDENCIA INSUFICIENTE sin divagar","prueba":"tests/test_abstencion.py con _llamar_modelo interceptado","riesgo":"Alto","esfuerzo":"M","dod":"D3=PASA"},
  {"wp_id":"WP-02","titulo":"Aislamiento model_id + anti-spillover","objetivo":"model_id como etiqueta rigida + hard pre-filter en el retriever con test de spillover","gap_ids":["GAP-12","GAP-25"],"archivos":["src/rag/buscador_lexico.py","src/rag/indexador_lexico.py","motor/Mx_01_LecturaEvidencias.py","web/services/pipeline_runner.py"],"depende_de":["WP-00"],"dia":1,"criterio_aceptacion":"Consulta sobre Modelo A devuelve 0% de chunks de Modelo B, verificable en traza","prueba":"tests/test_aislamiento.py::test_no_spillover","riesgo":"Alto","esfuerzo":"M","dod":"D2=0%"},
  {"wp_id":"WP-03","titulo":"Linaje doc:pagina:chunk reconstruible","objetivo":"Id estable por chunk y cita, registro de linaje por hallazgo","gap_ids":["GAP-10","GAP-19","GAP-31"],"archivos":["modulos/chunker.py","src/rag/indexador_lexico.py","modulos/motor_ia.py","modulos/generador_word.py"],"depende_de":["WP-02"],"dia":1,"criterio_aceptacion":"Desde un hallazgo del docx se reconstruye Claim->Chunk->Pagina->hash->Timestamp","prueba":"tests/test_linaje.py","riesgo":"Alto","esfuerzo":"M","dod":"D7=PASA; contribuye D4"},
  {"wp_id":"WP-04","titulo":"Verificador semantico de citas (NLI por prompt)","objetivo":"Entailment: el fragmento citado implica la afirmacion, sobre el mismo modelo Azure offline","gap_ids":["GAP-27","GAP-31","GAP-06"],"archivos":["modulos/motor_ia.py","modulos/verificador_nli.py"],"depende_de":["WP-03"],"dia":2,"criterio_aceptacion":"Cita literal presente pero no implicante se marca implica=false","prueba":"tests/test_citas_nli.py con entailment sintetico","riesgo":"Medio","esfuerzo":"M","dod":"D4>=0.80"},
  {"wp_id":"WP-05","titulo":"Circuit breakers y fusibles","objetivo":"Cap iteraciones <=3, fusible tiempo ~10s y tokens ~20000, escalado a HITL","gap_ids":["GAP-32","GAP-38"],"archivos":["config/parametros.py","modulos/motor_ia.py","modulos/skills/base.py"],"depende_de":["WP-00"],"dia":2,"criterio_aceptacion":"Bucle forzado corta a <=3 iteraciones y escala con hallazgos intermedios","prueba":"tests/test_circuit_breakers.py con reloj/token mock","riesgo":"Alto","esfuerzo":"S","dod":"D6=SI con valores"},
  {"wp_id":"WP-06","titulo":"Golden Dataset + Precision@K y nDCG","objetivo":"Ampliar preguntas-oro y anadir precision@5 y ndcg@10 al recall existente","gap_ids":["GAP-34","GAP-04"],"archivos":["scripts/preguntas_oro_rag.py"],"depende_de":["WP-00","WP-02"],"dia":2,"criterio_aceptacion":"preguntas_oro_rag.py --strict con Precision@5>=0.85, coste 0","prueba":"scripts/preguntas_oro_rag.py --strict","riesgo":"Alto","esfuerzo":"M","dod":"B1,B2,B4 medibles; B2>=0.85"},
  {"wp_id":"WP-07","titulo":"Bateria de robustez (math-injection + IPI)","objetivo":"Automatizar en pytest la inyeccion de error matematico y la resistencia a IPI, sin API","gap_ids":["GAP-53","GAP-22","GAP-06","GAP-26"],"archivos":["tests/test_math_injection.py","tests/test_ipi.py","tests/fixtures/modelos_sinteticos/"],"depende_de":["WP-01","WP-03"],"dia":2,"criterio_aceptacion":"Error VaR 99->95 produce DESVIADO+cita+rechazo; IPI se reporta como dato","prueba":"pytest tests/test_math_injection.py tests/test_ipi.py","riesgo":"Alto","esfuerzo":"M","dod":"D1=PASA; D5=PASA; contribuye A4,E1"},
  {"wp_id":"WP-08","titulo":"docx: veredictos por ambito, conclusiones, pie paginacion, referencias","objetivo":"Cerrar las 4 secciones obligatorias del artefacto primario sin degradar lo existente","gap_ids":["GAP-44","GAP-45","GAP-47","GAP-48"],"archivos":["modulos/generador_word.py","modulos/word_estilo.py"],"depende_de":["WP-03"],"dia":3,"criterio_aceptacion":"Informe conserva sin-observaciones y gana veredictos por ambito, pie con paginacion y referencias consolidadas","prueba":"tests/test_docx_regresion.py desde JSON fixture","riesgo":"Alto","esfuerzo":"M","dod":"F1=SI; F2=SI; F3=4/4"},
  {"wp_id":"WP-09","titulo":"Contrato Pydantic ValidationReport","objetivo":"Envolver la salida en esquema validado: findings, overall_confidence, abstained, unresolved_conflicts","gap_ids":["GAP-43","GAP-14","GAP-03"],"archivos":["modulos/esquemas.py","modulos/motor_ia.py","modulos/skills/catalogo.py"],"depende_de":["WP-01"],"dia":3,"criterio_aceptacion":"Cero ValidationError sobre fixtures; overall_confidence presente","prueba":"tests/test_esquemas.py","riesgo":"Medio","esfuerzo":"M","dod":"Contribuye A2 pto3, F, G"},
  {"wp_id":"WP-10","titulo":"Versionado multitier + ventana de contexto dinamica","objetivo":"is_active/supersedes/effective_date en estandares y deteccion de la ventana real del deployment","gap_ids":["GAP-23","GAP-40"],"archivos":["web/services/estandares_service.py","src/rag/config_rag.py","config/parametros.py","motor/Mx_00_CargaRutas.py","web/services/pipeline_runner.py"],"depende_de":["WP-00"],"dia":3,"criterio_aceptacion":"Estandar is_active=false no se usa; limite de contexto se resuelve del deployment no del literal 15000","prueba":"tests/test_versionado.py + extension test_prompts_editables.py","riesgo":"Medio","esfuerzo":"M","dod":"Checklist pto12 + elimina hardcode silencioso"}
]
```
