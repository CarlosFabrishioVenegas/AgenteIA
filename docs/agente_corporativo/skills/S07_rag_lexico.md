---
name: S07-rag-lexico
description: La literatura por dominio (VaR, LaR, ALM, dominios custom) no aparece, cita fragmentos irrelevantes, el índice está desactualizado o hay que añadir un dominio o un paper.
---

# S07 — RAG léxico y literatura por dominio

## Cómo funciona
- Corpus por dominio en `inputs/rag/<DOMINIO>/` (PDF/MD). Índice local por dominio.
- `src/rag/indexador_lexico.py` tokeniza (minúsculas, sin acentos, sin stopwords, stem) y guarda
  el índice; `src/rag/buscador_lexico.py` puntúa con **BM25 (0.7) + TF-IDF coseno (0.3)**;
  `src/rag/tesauro.py` expande sinónimos; parámetros en `src/rag/config_rag.py`
  (y `RAG_TOP_K`, `RAG_MIN_SCORE` en `config/parametros.py`).
- Cada dominio tiene `src/rag/<DOM>/indexer.py` y `retriever.py`; los custom usan
  `custom_retriever.py` / `base_indexer.py`.
- El pipeline consulta por actividad en `modulos/motor_ia.py::_enriquecer_con_rag`. El dominio se
  declara con `--dominio-rag` (si no, se infiere del título y el informe lo marca como inferido).
- Web: paso 04 «Dominio & RAG» (`web/routers/rag.py`): subir papers, reconstruir índice,
  dominios personalizados (`web/services/dominios_custom_store.py`).
- Controles extra dentro de `buscar`: versionado (`modulos/versionado.py`, descarta chunks
  derogados) y aislamiento opcional por modelo (`modulos/aislamiento.py`).

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill, `src/rag/buscador_lexico.py`, `src/rag/indexador_lexico.py`,
`src/rag/config_rag.py`, `src/rag/tesauro.py`, `src/rag/base_retriever.py`,
`src/rag/<DOM>/retriever.py`, `web/routers/rag.py`, `tests/test_rag_lexico.py`,
`tests/test_rag_desactualizado.py`. Para diagnóstico: salida de
`.venv\Scripts\python.exe scripts\explicar_rag.py` (o del endpoint de diagnóstico) pegada en el mensaje.

## Qué puede devolver (máx. 5)
Buscador/indexador/tesauro/config + su test.

## Pasos
1. Reproducir la consulta: `BuscadorLexico("<DOM>").diagnostico_consulta("<texto>")` muestra
   tokens, expansión y puntajes.
2. Si no hay resultados: ¿existe el índice? ¿se reconstruyó tras añadir papers? (el panel avisa si
   está desactualizado).
3. Si los resultados son irrelevantes: revisar tesauro, pesos, `RAG_MIN_SCORE`.
4. Añadir un caso de prueba con un corpus pequeño en `tmp_path`.

## Reglas que no se pueden romper
- **Sin HuggingFace, sin embeddings, sin ChromaDB** (`validate_project.py` y
  `tests/test_validador_entorno_hf.py` lo vigilan).
- Indexado y consulta usan la MISMA función `tokenizar` (si difieren, BM25 devuelve ceros).
- El corpus es material de apoyo, no fuente normativa: el informe lo declara.

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_rag_lexico.py tests/test_rag_desactualizado.py tests/test_validador_entorno_hf.py -q
```
Reconstruir índices: botón del paso 04 o `scripts/rebuild_rag_databricks.py`.

## Errores frecuentes
| Síntoma | Causa | Arreglo |
|---|---|---|
| Sin literatura en el informe | Índice no construido / dominio inferido mal | Reconstruir; declarar `--dominio-rag` |
| Resultados repetidos | Corpus con párrafos de plantilla | Deduplicar en el buscador |
| BM25 = 0 con corpus pequeño | IDF nulo en Okapi con pocos documentos | Usar BM25Plus o más documentos |
