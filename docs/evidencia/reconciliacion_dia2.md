# Reconciliación Día 2 — guía ↔ plan (REAL)

**Fecha:** 2026-08-28 · **Commit de partida:** `32a75bf` (cierre Día 1)
**Estado guías:** las 3 guías **YA están** en `docs/guias/`:
- `docs/guias/guia-arquitectura-agente-financiero (1).md`
- `docs/guias/guia-integracion-skills-agente (1).md`
- `docs/guias/guia-errores-loops-validacion-rag (1).md`

El Día 1 se ejecutó contra el **checklist embebido** en `docs/PLAN_MAESTRO_VIRF.md`
(las guías no existían — ver `reconciliacion_dia1.md`). Hoy se comparó el plan contra
las guías reales.

## Veredicto de la reconciliación

**El checklist embebido del plan es FIEL a las guías reales. No hay cambio material
de alcance para el Día 2.** No se añaden ni retiran gaps; no se reordenan WP por
divergencia normativa. Se procede con el Día 2 tal como lo fija el PROMPT 03.

## Confirmaciones plan↔guía (los WP del Día 2 quedan validados)

| WP Día 2 | Requisito de la guía | Cita en la guía | Veredicto |
|---|---|---|---|
| **WP-09** `ValidationReport` Pydantic | Esquema exacto `ValidationClaim`/`ValidationReport` | `guia-integracion-skills-agente:176-189` | **Coincide literal**: `requirement_id`, `status` ∈ {IMPLEMENTADO, PARCIAL, OMITIDO, DESVIADO}, `evidence_text` (página), `code_evidence` (línea), `justification`; `ValidationReport{model_id, overall_confidence, findings[], abstained, unresolved_conflicts}` |
| **WP-10** versionado | `effective_date`, `expiration_date`, `is_active` + pre-filtro duro | `guia-integracion-skills-agente:300`; `guia-arquitectura:50` (checklist 12) | **Coincide**: "pre-filtrado duro en las consultas del RAG" para no validar contra metodologías derogadas |
| **WP-10** ventana | anti "Lost in the Middle": no inundar contexto; top-5/8 | `guia-errores-loops:16-19` | **Coincide** con sustituir el `MAX_CTX=15000` fijo por detección/config explícita |
| **WP-08** `.docx` | veredicto por ámbito, pie con paginación, referencias | Plan §2.4 (GAP-44..48); guía exige reporte trazable | **Coincide** — trabajo de estructura, API-free desde `ValidationReport` mock |
| **Chunking estructural** | Chunking estructural que respete headings; no partir tablas/fórmulas | `guia-errores-loops:11-14` (§1.A "Naive Chunking") | **Coincide** — ya implementado Día 1 (opt-in); hoy toca el cutover con puerta de recall |
| **Calibración β** | abstención por umbral β calibrado, no inventado | `guia-arquitectura:45` (checklist 07); `guia-errores-loops:141` | **Coincide** — β desde distribución de scores léxicos |

## Divergencias guía↔proyecto (ya conocidas — Decisión #2 del plan; NINGUNA afecta el Día 2 API-free)

Las guías piden componentes que las restricciones vivas del proyecto prohíben o que
requieren API/nuevas deps. **Todas están fuera del alcance API-free del Día 2** y ya
estaban diferidas a P2 / día de API en el plan:

| Requisito de guía | Cita | Restricción de proyecto | Destino |
|---|---|---|---|
| RAG denso (BGE-M3 / E5) + embeddings | `guia-arquitectura:42`; `guia-integracion-skills:112` | RAG léxico **sin HuggingFace** | P2 (fuera de 3 días) |
| Reranker Cross-Encoder + RRF(k=60) | `guia-arquitectura:42`; `guia-errores-loops:19` | Sin HF | P2 |
| RAGAS/ARES (faithfulness/relevancy) | `guia-arquitectura:47`; `guia-errores-loops:54-62` | Sin crédito API; sin 2.º juez | Día 3 (API) / P2 |
| Juez de familia distinta | `guia-errores-loops:100` | Un solo deployment de chat | P2 |
| OpenTelemetry / Arize Phoenix | `guia-arquitectura:48,90` | Sin dependencias nuevas | P2 |
| Docker sandbox (código) + Dual-LLM IPI | `guia-integracion-skills:148`; `guia-arquitectura:49` | Sin nuevas deps/infra | P2 (día de API para IPI E2E) |
| NLI/entailment con clasificador | `guia-arquitectura:44,61` | Sin HF; se hace por prompt | Día 3 (API) — WP-04 |

## Nota de reordenamiento (no material)

- El plan marca **WP-06** (Precision@5 / nDCG@10, ampliar Golden Dataset) como *coste 0
  (léxico, sin API)* y lo situaba en el Día 2. El **PROMPT 03 lo reprograma al Día 3**
  (bloque de API, junto a IR/RAGAS). Se sigue el PROMPT 03: el trabajo IR API-free de
  hoy se limita a **re-correr la puerta de recall** (recall@1/@3) para el cutover del
  chunking y a la **calibración de β** con scores léxicos. WP-06 completo → Día 3.
- El orden del Día 2 (WP-09 → WP-10 → WP-08 estructura → cutover chunking → β) difiere
  del orden nominal del plan (que ubicaba WP-08/09/10 en el Día 3) porque el Día 1 ya
  cerró WP-01/02/03/05, y las piezas restantes de WP-04/06/07 necesitan API. Es un
  reordenamiento de **calendario**, no de alcance.

## Conclusión

Sin bloqueos. El alcance del Día 2 se mantiene. Se procede a ejecutar los WP API-free
en el orden del PROMPT 03.
