# Guía Avanzada de Ingeniería para Agentes RAG: Diagnóstico de Errores, Verificación del Retrieval y Bucles de Autocorrección

Este documento técnico de arquitectura e ingeniería establece los lineamientos para auditar, verificar y robustecer el agente de validación de modelos financieros (VIRF Agent) [492, 558]. En entornos altamente regulados de gestión de riesgo de modelos (p. ej., SR 11-7 en EE. UU. o las guías de riesgo de modelo del BCE) [562], la veracidad fáctica y la trazabilidad absoluta de la evidencia matemática son imperativos ineludibles [553, 562]. Esta guía condensa la evidencia de la literatura y el diseño de sistemas para eliminar alucinaciones y asegurar que cada juicio del agente esté respaldado matemáticamente por los documentos fuente [494, 563].

---

## 1. Catálogo de Errores Comunes en Agentes RAG Financieros

La construcción de agentes financieros basados en RAG comúnmente sufre de antipatrones que destruyen la confianza del validador humano. A continuación, se detallan los fallos críticos identificados y cómo evitarlos:

### A. Fragmentación Arbitraria (Naive Chunking)
*   **El Error:** Dividir documentos normativos o códigos de Python basándose en un conteo estricto de tokens o caracteres (ej., bloques fijos de 512 tokens) [85, 615].
*   **La Consecuencia:** Las fórmulas matemáticas complejas, las relaciones de celdas en tablas financieras y los supuestos quedan partidos a la mitad, destruyendo su sentido semántico [614, 615].
*   **La Solución:** Implementar **Chunking Estructural** que respete las secciones lógicas del texto (Markdown/PDF headings) [576, 615] o **Recuperación Contextual**, donde un modelo de lenguaje ligero (ej., Claude 3 Haiku) pre-procesa cada fragmento y le antepone un resumen aclaratorio de 50-100 tokens del documento base (ej., "Este fragmento detalla los supuestos del Modelo de Estrés Q3 del banco ACME...") [616].

### B. El Fenómeno "Lost in the Middle" (Pérdida de Atención)
*   **El Error:** Inundar la ventana de contexto del LLM con un top-k elevado (ej., top-20 o top-30 chunks de texto) creyendo que a mayor contexto, mejor respuesta [82, 576].
*   **La Consecuencia:** Estudios demuestran que los LLMs atienden significativamente mejor a la información colocada al inicio y al final de su ventana de contexto, ignorando o distorsionando la evidencia situada en la sección intermedia [576, 515, 554].
*   **La Solución:** Limitar el contexto inyectado a unos pocos fragmentos de alta relevancia (ej., top-5 o top-8), utilizando un **Reranker Cross-Encoder** para reordenarlos por similitud semántica real y colocar los chunks más críticos en los extremos del prompt [575, 576, 819].

### C. Citas Decorativas y Alucinación de Fuentes
*   **El Error:** Confiar en que el LLM citará correctamente solo porque se le instruyó en el prompt [565, 577].
*   **La Consecuencia:** El modelo genera una respuesta plausible basada en su conocimiento general y le asigna números de cita inventados (ej., `[Doc A: Página 12]`) que no corresponden a la realidad o que no contienen la evidencia afirmada [580, 584, 657].
*   **La Solución:** Implementar un validador determinístico separado que descomponga la respuesta en afirmaciones (claims) atómicas y valide mediante implicación lógica (entailment/NLI) que el chunk citado efectivamente contiene el hecho descrito [580, 581, 584].

### D. Efecto Cascada de Errores en Sistemas de Agentes (MAS)
*   **El Error:** Diseñar un sistema de múltiples agentes sin puntos de control intermedios, donde el output crudo del Agente A alimenta directamente las decisiones del Agente B [109, 316, 317].
*   **La Consecuencia:** En redes complejas de agentes, un error del 5% de imprecisión en el paso 1 se multiplica rápidamente en las fases de análisis, mapeo y generación, degradando drásticamente el resultado final (ley de impacto compuesto de errores) [109, 316, 317].
*   **La Solución:** Implementar **Checkpoints de Razonamiento** y "Fusibles" (Circuit Breakers) en transiciones de estado críticas, exigiendo validación automática y, si es necesario, aprobación humana antes de propagar decisiones de alto impacto financiero [318, 406, 408].

### E. Envenenamiento del Retrieval (Feedback Poisoning)
*   **El Error:** Permitir que las interacciones diarias o correcciones crudas del desarrollador del modelo financiero actualicen automáticamente la base de datos de RAG para "aprender" con el uso [590, 544].
*   **La Consecuencia:** Datos incorrectos o sesgados se indexan directamente, provocando una contaminación permanente del retrieval semántico para futuros usuarios [590, 544].
*   **La Solución:** Ningún feedback directo modifica la base de conocimiento sin pasar por un comité humano (Human-in-the-loop) y almacenarse en un índice aislado de anulaciones prioritarias (Tier 0 Overrides) [590, 592, 638].

---

## 2. Cómo Verificar que el RAG Funciona Verdaderamente (Deep Dive)

La única forma de garantizar que un sistema RAG opera de forma confiable es aislar la evaluación de sus dos fases: la recuperación de información (Retrieval) y la generación de la respuesta (Generation) [546, 622].

### A. Verificación de la Recuperación Documental (Métricas IR)
La calidad de la recuperación debe evaluarse de forma offline sobre un set de prueba representativo ("Golden Dataset") anotado por expertos humanos, el cual define para cada consulta qué documentos/chunks son la única fuente de verdad (Ground Truth) [514, 585, 623].

1.  **Recall@K (Tasa de Recuperación):**
    *   *Fórmula:* $	ext{Recall@K} = rac{	ext{Docs relevantes recuperados en top-K}}{	ext{Total de docs relevantes esperados}}$ [623, 624]
    *   *Significado:* Si el Recall@K cae por debajo de un umbral aceptable (ej., < 0.85), el generador jamás podrá responder bien porque carecerá de la evidencia matemática necesaria. Es la métrica más crítica de la Fase 1.
2.  **Precision@K (Precisión de Recuperación):**
    *   *Significado:* Mide la proporción de documentos recuperados que son realmente relevantes [624]. Una precisión baja inunda el contexto de la IA con ruido que diluye la atención [624].
3.  **nDCG@K (Normalized Discounted Cumulative Gain):**
    *   *Significado:* Es la métrica de ordenamiento más avanzada [625]. Penaliza logarítmicamente si un documento altamente crítico se recupera en una posición baja (ej., posición 9 en lugar de la posición 1), garantizando que lo más útil esté siempre al principio de la lectura [625].

### B. Verificación de la Generación de Respuestas
Se utilizan frameworks de evaluación basados en LLM como jueces (LLM-as-a-Judge, ej., Ragas y ARES) calibrados críticamente para evaluar la respuesta generada [580, 795, 801]:

1.  **Faithfulness (Fidelidad / Groundedness):**
    *   *Mecanismo:* El LLM descompone la respuesta en afirmaciones lógicas e individuales [798, 626]. Para cada una, se verifica si es deducible únicamente de los contextos recuperados [798, 626].
    *   *Score:* $	ext{Faithfulness} = rac{	ext{Afirmaciones atómicas respaldadas}}{	ext{Total de afirmaciones generadas}}$ [626, 627]
2.  **Answer Relevancy (Relevancia de la Respuesta):**
    *   *Mecanismo:* Se le pide a un LLM independiente generar múltiples preguntas hipotéticas que serían respondidas por la salida obtenida, y se mide la similitud del coseno de esas preguntas simuladas contra la duda original del usuario [627, 798]. Evita que el agente entregue rollos innecesarios o evasivas sin responder la pregunta exacta [627, 799].
3.  **Citation Recall & Precision:**
    *   *Mecanismo:* Evaluar si cada afirmación factual sobre el modelo financiero cuenta con su respectiva cita y si la fuente referenciada realmente soporta la afirmación semántica [580, 690].

---

## 3. Revisión y Diseño de una Validación de Calidad Correcta

Un sistema de validación financiera no puede ser una simple instrucción en el prompt del LLM o depender enteramente de un juez probabilístico [561, 580]. Debe estructurarse mediante un nodo de orquestación lógica independiente (Answer Validator / Critic) [559, 581]:

```
                                +-------------------+
                                | Respuesta del LLM |
                                +-------------------+
                                          |
                                          v
                         +---------------------------------+
                         |      Validador Determinista     |
                         | (¿Existe cita? ¿Regex de Hash?) |
                         +---------------------------------+
                                          |
                                          v
                         +---------------------------------+
                         |      Verificador Semántico      |
                         |  (NLI / LLM Juez Independiente) |
                         +---------------------------------+
                                          |
                        +-----------------+-----------------+
                        |                                   |
                        v                                   v
             [Suficiente Evidencia]              [Falla / Sin Evidencia]
                        |                                   |
                        v                                   v
                 (Enviar al Usuario)               (Loop / Abstenerse)
```

### A. Mitigar los Sesgos del "LLM-as-a-Judge"
Los modelos evaluadores introducen distorsiones sistemáticas que deben neutralizarse [542, 580]:
*   **Sesgo de Posición:** Los modelos tienden a calificar mejor la primera opción que leen en una comparación [580, 542]. *Mitigación:* Alterar el orden de los contextos de entrada durante la validación o intercambiar las opciones de respuesta y promediar los puntajes [580].
*   **Sesgo de Verbosidad:** El LLM prefiere textos largos y complejos sobre explicaciones directas y concisas [580, 542]. *Mitigación:* Establecer rúbricas estrictas de penalización por excesiva verbosidad y requerir salidas estructuradas estrictas (JSON Schema) [580, 665, 771].
*   **Sesgo de Auto-reconocimiento:** Los modelos tienden a evaluar mejor respuestas que ellos mismos generaron [580, 542]. *Mitigación:* Usar un modelo evaluador de un proveedor o familia de pesos neuronales diferente al modelo de generación [580].

### B. Mapeo de Linaje y Trazabilidad Extrema
Para que una validación sea auditable, el sistema debe registrar en su esquema de datos el grafo completo de origen [563, 588]:

$$	ext{Respuesta final} \longrightarrow 	ext{Afirmaciones atómicas} \longrightarrow 	ext{IDs de Chunks} \longrightarrow 	ext{Páginas / Secciones} \longrightarrow 	ext{ID del PDF} \longrightarrow 	ext{Versión temporal}$$

---

## 4. Bucles de Debate y Autocorrección (Reflexion & Multi-Agent Loops)

Cuando el validador automático detecta una discrepancia entre la documentación (PDF) y el código de Python analizado, el agente no debe simplemente rendirse o lanzar un error crudo [559, 563]. El sistema debe activar bucles de reflexión y debate interno para refinar el diagnóstico [114, 125]:

### A. Implementación de Frameworks de Autocorrección

#### 1. ReAct (Reasoning and Acting)
El agente intercala secuencias de **Pensamiento, Acción y Observación** de forma interactiva [124, 669]:
*   *Pensamiento:* "Necesito validar la función del VaR en el script `var_model.py`. El documento PDF de validación dice que la fórmula debe usar la covarianza de EWMA con un factor de decaimiento de 0.94" [562].
*   *Acción:* Invocar el linter/intérprete para inspeccionar las líneas de código del script [124, 669].
*   *Observación:* "El código implementa `df.cov()` simple sin ponderación exponencial".
*   *Pensamiento corrector:* "Esto contradice la documentación. Procederé a redactar el desvío metodológico exacto citando la página del PDF y la línea de código" [125, 563].

#### 2. Reflexion (Shinn et al.)
Este framework divide la autocorrección en tres roles lógicos [126]:
*   **Actor (Generador):** Produce el reporte preliminar de match documentación/código [126].
*   **Evaluador (Validator):** Ejecuta pruebas de exactitud y groundedness [126].
*   **Autoreflexión:** Analiza el reporte del evaluador (ej., "El reporte afirma que la variable X no está en el código, pero el buscador léxico no revisó el archivo auxiliar de imports. El diagnóstico es incompleto") y reprograma la "trayectoria" del actor para una nueva ejecución corregida [126].

#### 3. Debate Multi-Agente (Actor-Critic / Peer Review)
Las investigaciones (ej., de MIT y Google Brain) demuestran que cuando se implementa un esquema de debate donde dos agentes con diferentes prompts (ej., un Agente de Validación de Código vs. un Agente Metodológico de Riesgo) discuten y critican los resultados del otro, **las tasas de error fáctico se reducen en más de un 22%** [315]. El debate obliga a refinar afirmaciones vagas en conclusiones robustas basadas exclusivamente en citas fácticas [315, 466].

### B. Mecanismos de Control contra Bucles Infinitos y Costos Descontrolados
El uso de ciclos de debate de IA incrementa significativamente la latencia percibida por el usuario y los costos por consumo de tokens [68, 127]. Por ello, se deben programar los siguientes mecanismos de defensa:

*   **Límite de Iteraciones (Recursion Depth Cap):** Configurar un parámetro estricto de máximo 2 o 3 reintentos en el bucle de validación [570, 646]. Si el agente no logra un consenso de groundedness o coincidencia de fórmulas en la tercera vuelta, debe abortar y escalar la consulta al revisor humano con los hallazgos intermedios registrados [407, 408].
*   **Fusibles de Consumo (Circuit Breakers):** Detener de inmediato el proceso si el costo estimado del token o el tiempo de ejecución supera los límites de tolerancia definidos (ej., más de 10 segundos de espera o 20,000 tokens consumidos) [395, 408, 409].
*   **Promoción al Tier 0 Overrides:** Cuando un debate sea resuelto y aprobado por el validador humano (HITL), la solución fáctica curada debe agregarse al corpus de "respuestas pre-aprobadas" para optimizar futuras búsquedas semánticas directas de cero latencia [590, 592, 638].

---

### Mensaje Final del Arquitecto
> *"La confiabilidad de un agente financiero no se mide por la elocuencia de sus respuestas, sino por la solidez de sus mecanismos de defensa: aislamiento estricto de datos, validadores determinísticos de citas, y la valentía matemática para decir 'no tengo suficiente evidencia para concluir'."* [561, 569, 583]
