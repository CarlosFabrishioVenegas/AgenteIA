# Guía de Arquitectura del Agente VIRF: Validación de Modelos Financieros con RAG Avanzado y Observabilidad

Este documento establece las especificaciones técnicas y operativas para el desarrollo, validación y despliegue del **VIRF Agent** (Validation, Integration, Retrieval, and Feedback). La arquitectura está diseñada para automatizar la revisión de extremo a extremo de modelos financieros en entornos bancarios altamente regulados (en cumplimiento de marcos de gestión de riesgos como SR 11-7 en EE. UU. o las guías TRIM del Banco Central Europeo) [214]. El objetivo principal es disociar la lógica de razonamiento del LLM de la persistencia de datos factuales, garantizando un proceso de validación basado estrictamente en evidencia auditable [211, 214, 267].

---

## 1. Flujo de Trabajo Ideal del Agente (E2E)

El agente opera como un sistema autónomo de toma de decisiones ordenado en once fases secuenciales con compuertas lógicas y controles de retroceso:

```
[PDF del Modelo] ──> (Ingesta e Indexación) ──> [Extracción de Requisitos]
                                                       │
[Grafo de Código] <── (Análisis de Código) <── [RAG Específico (Hard-Filter)]
        │
        └──> [Match Doc vs Código (NLI)] ──> [Detección de Gaps (Tri-State)]
                                                      │
[Reporte Trazable] <── (Auto-Evaluación RAGAS) <── [Comprobación de Evidencia]
```

1. **PDF del Modelo (Input):** Carga de guías metodológicas, documentos de seguimiento y manuales técnicos específicos del modelo financiero [214].
2. **Ingesta y Clasificación:** Análisis jerárquico del documento mediante Reconocimiento Óptico de Caracteres (OCR) avanzado para preservar tablas y estructuras visuales [268]. Se extrae un identificador único de modelo (`model_id`) que se inyecta como etiqueta rígida en los metadatos [211, 288].
3. **Extracción Estructurada de Requisitos:** Un agente de extracción analiza el texto completo e identifica de manera estructurada (JSON Schema) las variables, supuestos, controles y criterios de aceptación declarados en el PDF [215, 229].
4. **RAG Específico del Modelo:** Búsqueda híbrida (léxica BM25 + vectorial densa) restringida mediante filtrado estricto por metadatos utilizando el `model_id` [211, 221, 271, 288].
5. **Análisis del Código Implementado:** Procesamiento estático y/o dinámico del código fuente del modelo financiero (scripts en Python, clases, librerías) en un entorno aislado de sandbox para extraer variables reales, llamadas de librerías y fórmulas lógicas ejecutadas [266, 287].
6. **Match Documentación vs. Código:** Un agente comparador realiza un análisis de implicación semántica (Natural Language Inference o NLI) por claim individual, evaluando si el fragmento de código correspondiente implementa la especificación documental [232, 233].
7. **Detección de Gaps e Incumplimientos:** Clasificación tri-state de cada requisito: *Implementado*, *Parcialmente Implementado* (con desvíos o fórmulas discrepantes), u *Omitido* (ausencia del requisito en el código) [213, 233, 282].
8. **Verificación Asíncrona de Evidencia:** Un agente crítico (Answer Validator) valida si las citas extraídas del PDF realmente respaldan las afirmaciones de validación realizadas por el agente principal, descartando referencias espurias o "decorativas" [215, 230, 232].
9. **Conclusión y Generación de Veredictos:** Emisión del juicio de validación técnica respaldado por un vector de confianza derivado de la groundedness de la respuesta [233, 235].
10. **Generación del Reporte Trazable:** Emisión de un reporte enriquecido con enlaces inequívocos en formato jerárquico: `Respuesta -> Claim -> Chunk -> Documento -> Fuente original -> Versión` [236, 240].
11. **Evaluación Continua y Registro:** Almacenamiento de trazas detalladas (métrica de fidelidad, latencia, costos y prompt caching) bajo estándares abiertos para análisis de regresión y auditoría [195, 241, 290].

---

## 2. Checklist Técnica y Operativa de Implementación

| ID | Componente de Arquitectura | QUÉ DEBE EXISTIR (Especificación Técnica) | QUÉ DEBE HACER (Flujo de Control) | QUÉ PROBLEMA EVITA (Mitigación) | CÓMO COMPROBAR QUE FUNCIONA (Prueba de Aceptación) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **01** | **Aislamiento Multitest (Multi-tenancy)** [239] | Filtros determinísticos basados en listas de control de acceso a nivel de documento (Document-Level ACLs) indexadas en el payload vectorial [288]. | Filtrar el espacio de búsqueda vectorial y léxica mediante cláusulas de igualdad duras (`model_id` = UUID) antes de la búsqueda aproximada (ANN) [211, 288]. | Evita la contaminación cruzada entre modelos; impide que se usen supuestos de un modelo de riesgo crediticio para validar uno de trading [211, 239]. | Ejecutar una consulta sobre el Modelo A y validar en las trazas que el payload devuelto por la Vector DB contenga un 100% de registros con el identificador del Modelo A [211]. |
| **02** | **Ingesta Estructural y OCR jerárquico** [268] | Parseadores basados en OCR con conciencia de diseño físico y tablas estructuradas (ej. preservando jerarquías visuales en secciones H1, H2, H3) [268, 269]. | Extraer texto manteniendo la relación espacial de las matrices de datos financieras y asumiendo etiquetas HTML/Markdown para denotar encabezados y páginas [140, 268]. | Evita la fragmentación ciega de texto que corta tablas o rompe fórmulas matemáticas distribuidas en múltiples líneas [268]. | Verificar que los fragmentos de tablas complejas recuperados mantengan su estructura semántica de filas y columnas legibles por el LLM [268]. |
| **03** | **Extracción de Requisitos en JSON Schema** [229] | Salidas estructuradas de Azure OpenAI (Structured Outputs/JSON Schema) validadas estricta e internamente mediante Pydantic [229, 294]. | Forzar al agente extractor a entregar un arreglo JSON con atributos específicos: `id_requisito`, `tipo`, `sección`, `página`, `métricas_de_control` y `criterios_aceptación` [215, 229]. | Previene que el agente omita variables de control clave, abrevie requisitos complejos o devuelva formatos de texto plano ilegibles por scripts aguas abajo [229, 311]. | Ejecutar la extracción del PDF y pasar la respuesta por un validador de esquemas Pydantic; el job debe completarse con cero fallos de validación de tipado [446, 491]. |
| **04** | **Búsqueda RAG Híbrida y Reranking** [212, 221] | Recuperación fusionada BM25 (frecuencia de términos) + embeddings densos (BGE-M3 o Multilingual-E5) + Reranker Cross-Encoder (ej. BGE-Reranker) [221, 225, 227]. | Recuperar un top-50 de chunks candidatos, aplicar Fusión de Rangos Recíprocos (RRF con $k=60$) y reordenar mediante el re-clasificador para inyectar al contexto solo los mejores 5-8 chunks [221, 227, 271, 272]. | Soluciona el bajo recall semántico en variables con acrónimos (ej. VaR, PD, LGD) y el fenómeno "Lost in the Middle" (dilución en contextos extensos) [221, 228]. | Ejecutar una consulta técnica con jergas financieras complejas; comprobar que la Precision@5 supere el 0.85 comparada con una búsqueda densa pura [230, 295]. |
| **05** | **Análisis de Código Fuente Real** [266] | Un sandbox de ejecución Python aislado con acceso condicional al script del modelo a través de linters estáticos y análisis dinámico de grafos [266, 287]. | Escanear el archivo `.py` o Jupyter Notebook de producción para mapear la existencia de funciones, variables asignadas y librerías importadas reales [266]. | Evita que el agente asuma que el modelo está correctamente codificado basándose solo en los comentarios textuales del código creados por el desarrollador [220]. | Suministrar código con nombres de variables correctos pero con lógica matemática errónea; el analizador debe reportar la discrepancia exacta [213, 231]. |
| **06** | **Match Documento vs Código con NLI** [232] | Algoritmo de implicación semántica por claims atómicos y validación de fórmulas determinísticas [232]. | Descomponer la descripción del PDF en aserciones atómicas y verificar si el código analizado lógicamente las implica (*entailment*), contradice o las ignora [231, 232]. | Evita falsos positivos de cumplimiento donde se detecta la función pero la lógica de cálculo (ej. factor de descuento) difiere de la especificación técnica [213]. | Probar con un código que implementa la fórmula al revés (ej. suma en lugar de resta); el comparador debe clasificar el requisito como "Desviado/Discrepancia" [213]. |
| **07** | **Mecanismo de Abstención Activa** [213, 235] | Compuerta lógica externa (Answer Validator Gate) basada en umbrales de relevancia híbrida (score mínimo de vector y proximidad semántica) [233, 280, 281]. | Detener la generación y emitir "EVIDENCIA INSUFICIENTE" si la relevancia de los chunks recuperados cae por debajo del umbral mínimo calibrado ($\beta$) [235, 281, 282]. | Bloquea alucinaciones y extrapolaciones del LLM basadas en su entrenamiento general de internet ante la ausencia de documentación local [220, 235]. | Ejecutar una consulta sobre "Metodología de Estrés 2027" (inexistente en el corpus); el agente debe responder con la abstención estándar y negarse a divagar [235]. |
| **08** | **Trazabilidad Lineage Completa** [236, 240] | Estructura de base de datos relacional/Sidecar metadata persistente que asocia unívocamente cada elemento de la validación [240, 257, 285]. | Enlazar de forma secuencial y auditable: `ID_Hallazgo -> Claim_Específico -> Chunk_ID -> Página_PDF -> Hash_Archivo_Código -> Línea_Ejecución -> Timestamp` [236, 240, 284]. | Elimina la "caja negra" regulativa; permite a los auditores internos y externos reconstruir manualmente la validación de cualquier cálculo en segundos [214]. | Dar clic en un hallazgo del reporte y verificar que redirija al fragmento exacto del documento de origen y a la línea exacta de código asociada [236, 240]. |
| **09** | **Evaluación Automatizada (RAGAS)** [230, 291] | Un pipeline de pruebas de regresión automatizadas en el CI/CD alimentado con un dataset de evaluación de oro (Golden Dataset) anotado por expertos [237, 249, 291]. | Medir de manera independiente las dimensiones de RAG: Faithfulness (fidelidad), Answer Relevancy (relevancia de respuesta) y Context Precision (precisión de contexto) [230, 231, 232]. | Impide la degradación silenciosa del agente ante cambios de versión del LLM base, cambios de prompts o variaciones en los índices de embeddings [226, 254]. | El flujo de despliegue en GitHub Actions debe fallar de inmediato si una actualización de prompt reduce la fidelidad semántica (faithfulness) a menos de 0.85 [16, 291]. |
| **10** | **Observabilidad de Trazas (Telemetry)** [241, 290] | Esquema de instrumentación unificado con OpenTelemetry y semánticas GenAI (ej. integrado con Arize Phoenix de forma self-hosted) [241, 249, 290]. | Registrar todas las fases del ciclo de inferencia: consulta original, consulta reescrita por el agente, IDs de chunks recuperados, scores del reranker y la justificación del crítico [241, 290]. | Permite el análisis de fallos en producción para detectar si el error de validación provino de un mal recall de búsqueda o de una pobre capacidad de síntesis del modelo [241]. | Visualizar el dashboard de telemetría y corroborar la reconstrucción del árbol jerárquico completo para una consulta de validación multietapa compleja [52, 241]. |
| **11** | **Prevención de Ataques de Inyección (IPI)** [286] | Entorno sandbox protegido (Sandboxing), Dual LLM Pattern y pasarelas de seguridad (Confirmation Gates) [287]. | Limpiar y filtrar instrucciones ejecutables embebidas en los PDF usando un LLM de bajo costo antes de pasar el texto limpio al agente supervisor principal [287]. | Neutraliza la Inyección Indirecta de Prompts (IPI); evita que un documento envenenado con instrucciones ocultas obligue al agente a ignorar fallos u omitir métricas [286, 287]. | Introducir en el PDF un texto oculto que ordene "ignorar controles de riesgo"; verificar que el agente reporte el texto como dato limpio y mantenga su flujo de revisión [287]. |
| **12** | **Versionado Multitier** [240] | Registro de versiones integrado en tres niveles: Documentos del corpus, Embeddings y Archivos de código fuente analizados [240, 285]. | Estampar marcas temporales (`timestamps`), hash MD5 y etiquetas de superación (`supersedes_version_id`) a cada elemento cargado en los índices de la Vector DB [240, 258, 284]. | Evita que el agente valide código de producción contra metodologías obsoletas o use índices vectoriales corruptos tras una actualización de modelos [240, 254]. | Modificar un documento técnico (creando la V2); verificar que el agente marque de forma automática la V1 como inactiva e indexe la nueva versión [240]. |

---

## 3. Sección "NO DEBERÍA ESTAR PASANDO" (Antipatrones Críticos)

Para garantizar la fiabilidad del agente de validación en entornos financieros de alto riesgo, los siguientes comportamientos se clasifican como **fallos catastróficos del sistema** y su presencia anula cualquier informe de validación:

*   **Responder consultas técnicas sin consultar el RAG (Uso de Conocimiento Paramétrico):** El agente responde de forma persuasiva sobre parámetros o metodologías del modelo utilizando su conocimiento general de entrenamiento en finanzas, en lugar de extraerlo del PDF asignado [220]. **Control:** Prohibir explícitamente el uso de conocimiento general para validar claims y forzar al sistema a declarar "Evidencia Insuficiente" si no hay chunks mapeados en el contexto [235].
*   **Contaminación de Modelos (Model Spillover):** El agente utiliza fragmentos recuperados pertenecientes al Modelo B para generar juicios de conformidad o verificar variables del Código A [211]. **Control:** Implementación de filtrado duro determinístico por metadatos utilizando `model_id` en el nivel de base de datos vectorial; prohibir búsquedas "globales" abiertas en la base de conocimientos [211, 288].
*   **Juicios de Cumplimiento sin Evidencia Explícita (Falsa Aceptación):** El agente declara que una variable "está implementada en cumplimiento con el modelo" basándose en el supuesto de que "es un estándar del mercado" o "se infiere de la estructura general", sin citar el párrafo del PDF ni la línea de código [220, 235]. **Control:** El Answer Validator debe rechazar cualquier output con un veredicto de conformidad que carezca de un mapeo bidireccional (Cita de Documento $\leftrightarrow$ Línea de Código) [213, 231, 232].
*   **Citas Alucinadas o "Decorativas":** El agente añade referencias de página y secciones existentes en el documento (ej. "Página 12, Sección 4"), pero al inspeccionar esa sección, el texto recuperado no tiene ninguna relación semántica con la conclusión expresada por el agente [197, 232, 236]. **Control:** Ejecutar validación de implicación lógica (entailment) mediante clasificadores NLI entrenados contra las citas exactas mapeadas [232].
*   **Ignorar Omisiones Documentadas (Sesgo de Confirmación del Desarrollador):** El agente valida con éxito la ejecución del código sin alertar que tres secciones de control metodológico críticas (especificadas en el manual del modelo) no existen en absoluto en el repositorio analizado [213]. **Control:** El flujo de validación debe iniciarse con la extracción de los requisitos de control del documento y comprobar la existencia de cada uno en el código, en lugar de limitarse a clasificar lo que sí está programado [213, 215].
*   **Alimentación Automática y Envenenamiento por Feedback (Data Poisoning):** Permitir que la retroalimentación o el feedback de los usuarios del sistema modifique en tiempo de ejecución los pesos del modelo o ingrese directamente al índice vectorial del RAG sin auditoría humana [212, 242]. **Control:** Establecer un flujo asíncrono estricto donde el feedback de baja confianza se envíe a una cola de revisión de expertos para su incorporación segura (Índice de Sobrescritura Tier 0) [242, 243, 289].

---

## 4. Priorización de Mejoras y Roadmap de Arquitectura

Para implementar el sistema de manera incremental y mitigar el riesgo de complejidad y latencia descontrolada, se prescribe un roadmap dividido en tres horizontes lógicos de desarrollo:

### Fase P0: Fundamentos de Evidencia y Aislamiento (Semanas 1 a 4) [182, 184]
*   **Enfoque:** Establecer el aislamiento total de los datos y el mecanismo primario de trazabilidad sin introducir software no verificado [165, 211, 250].
*   **Entregables:**
    1.  **Aislamiento determinístico por `model_id`:** Implementar el filtrado duro de metadatos (pre-filtering) en la Vector DB [211, 288, 294].
    2.  **Mecanismo de Abstención Activa:** Programar la compuerta de suficiencia de evidencia ($\beta$-threshold) antes del proceso de generación [233, 235, 281].
    3.  **Trazabilidad Extrema:** Diseñar el esquema relacional que vincula el output con las citas de páginas y hashes del documento original [240].
*   **Criterio de Éxito:** El agente alcanza una tasa de alucinación semántica igual a 0% sobre consultas huérfanas de evidencia de prueba (el sistema prefiere abstenerse antes que especular) [235].

### Fase P1: Inteligencia de Ingesta, Recuperación Híbrida y Verificación (Semanas 5 a 8) [183, 184]
*   **Enfoque:** Optimizar el recall técnico y automatizar el contraste documental-código mediante validadores determinísticos y de implicación semántica [166, 167, 251, 252].
*   **Entregables:**
    1.  **Pipeline de Búsqueda Híbrida + Reranking:** Integrar indexación dispersa (BM25) y densa (BGE-M3) unificada por Reciprocal Rank Fusion ($k=60$) y filtrado por Cross-Encoder [221, 225, 227].
    2.  **Extractor Estructurado:** Pipeline de generación en formato JSON Schema con tipado estricto mediante Azure OpenAI Structured Outputs [229, 294].
    3.  **Answer Validator NLI (Claim-to-Source Alignment):** Implementar la descomposición de la respuesta en hechos atómicos y verificar la implicación lógica de las aserciones contra los fragmentos de código recopilados [231, 232, 233].
*   **Criterio de Éxito:** El Recall@10 sobre el Golden Dataset en español de modelos financieros supera el 0.85; la precisión de las citas identificadas se eleva un 50% [230, 291, 296].

### Fase P2: Telemetría de Grado Industrial, Gobierno y HITL (Semanas 9 a 12) [183, 184]
*   **Enfoque:** Proveer capacidades de auditoría de grado militar, observabilidad de producción y resiliencia ante ciberataques [168, 252, 253, 297].
*   **Entregables:**
    1.  **Trazabilidad de Telemetría (OpenTelemetry):** Diseñar el pipeline de trazas por Span (Retrieval $\rightarrow$ Rerank $\rightarrow$ Generation $\rightarrow$ Criticism) utilizando Arize Phoenix de forma local [241, 249, 290].
    2.  **Gobernanza del Conocimiento (HITL Loop):** Crear la cola de revisión manual y el Índice Vectorial de Sobrescritura (Tier 0 Overrides) para incorporar feedback aprobado [243, 289].
    3.  **Sandboxing del Parser de Código:** Aislar el flujo de análisis estático del código del modelo mediante aislamiento a nivel de sistema operativo para bloquear la Inyección Indirecta de Prompts (IPI) [287].
*   **Criterio de Éxito:** El sistema supera con éxito las pruebas de simulación de ciberataques (Red Teaming/IPI con robo de credenciales) sin comprometer datos confidenciales de la base de conocimiento o del servidor de inferencia [181, 287].

---

## 5. Regla de Oro de la Validación VIRF

> **«El agente NO debe afirmar que un modelo financiero cumple, incumple o implementa un requisito basándose únicamente en su conocimiento paramétrico o de entrenamiento general. Cada conclusión crítica debe poder rastrearse de manera transparente hasta el documento específico del modelo, su evidencia textual exacta (Sección, Página, Fragmento) y su traducción e implementación real encontrada en el código fuente. Si la evidencia no es concluyente o presenta contradicciones insalvables, el sistema debe indicarlo, abstenerse de emitir veredictos sesgados y escalar el caso al flujo de confirmación humana (HITL)» [213, 220, 235, 243].**

### Configuración del Prompt de Control del Agente (System Prompt de Mitigación) [229, 235]

```python
SYSTEM_PROMPT = """
Usted es un validador senior de modelos financieros y actúa bajo las reglas estrictas de gestión de riesgos del modelo (SR 11-7 / TRIM).
Su objetivo principal es auditar que el código implementado ejecute exactamente los requisitos especificados en los manuales del modelo.

REGLAS DE OPERACIÓN CRÍTICAS:
1. EXCLUSIVIDAD DE ENTORNO: Solo se permite fundamentar hallazgos en la evidencia del contexto de documentos inyectados.
   Si el contexto no proporciona suficiente información técnica para verificar la variable, su veredicto de auditoría DEBE ser 'EVIDENCIA INSUFICIENTE PARA VALIDACIÓN'. No invente variables ni lógicas.
2. CITACIÓN OBLIGATORIA: Cada afirmación fáctica de cumplimiento, discrepancia u omisión debe citar la fuente exacta del documento usando la etiqueta estructurada [doc_id:chunk_id] asociada al metadato original.
3. ADVERSARIALIDAD DE CÓDIGO: No asuma que el código funciona solo porque los comentarios textuales dicen que lo hace. Verifique la lógica matemática y estructural.
4. CERO CONOCIMIENTO PARAMÉTRICO: Queda terminantemente prohibido usar explicaciones teóricas ausentes en la guía para justificar la conformidad del código. El conocimiento general del modelo LLM sirve solo para estructurar la sintaxis de salida y el razonamiento del flujo.
"""
```
