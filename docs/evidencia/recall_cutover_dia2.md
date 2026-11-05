# Cutover del chunking estructural — evidencia de recall (Día 2)

**Fecha:** 2026-08-28 · **Puerta:** `recall@1 ≥ 0.85` sobre `preguntas_oro_rag`
(léxico, API-free). **Regla:** promover a default solo si pasa y no empeora
respecto al corte por tamaño; si regresa, permanece opt-in.

## Medición antes vs después

| Métrica | ANTES (corte por tamaño) | DESPUÉS (estructural) | Umbral |
|---|---|---|---|
| **recall@1** (discriminantes, n=19) | **1.0000** | **1.0000** | ≥ 0.85 |
| recall@3 (elegibles, n_r3=19) | 1.0000 | 1.0000 | ≥ 0.90 (activo) |
| adversarias que declaran alcance | 4/4 | 4/4 | 4/4 |

Chunks por dominio tras reindexar con estructural: LaR=9, VaR=15,
conductuales=6, shared=15.

## Decisión: PUERTA PASADA, pero REVERTIDO A OPT-IN por regresión

La **puerta de recall PASÓ** (recall@1 = 1.0 = baseline por tamaño, sin regresión
de recuperación, adversarias 4/4). Se promovió a default y las suites RAG
puntuales quedaron verdes (66/66). PERO la **suite completa reveló una regresión**
que las puntuales no cubren:

- `test_regresion.py::...::test_chunks_identicos_no_se_pisan_entre_si` **FALLA**
  con estructural por default: su escenario sintético (cabecera repetida sin
  estructura) colapsa a 1 chunk bajo el chunker estructural, y el test —acoplado
  al corte por tamaño— exige `len(indice) > 1`.

Conforme a la regla del prompt ("si regresa, revierte a opt-in y anota el
hallazgo"), **el chunking estructural permanece OPT-IN** (default por tamaño).
Disponible con `RAG_CHUNKING_ESTRUCTURAL=1`. El mecanismo, la puerta y esta
evidencia quedan; la promoción a default se retoma cuando el test de integridad
de índice se desacople del chunker concreto (trabajo futuro, fuera del Día 2).

**Hallazgo:** una puerta de recall verde NO garantiza ausencia de regresión; los
tests de integridad de índice deben correrse en la suite completa antes de
cualquier promoción a default.

## Mecanismo (aditivo, reversible)

- Flag EXPLÍCITO `estructural` en `IndexadorLexico.__init__`
  (`src/rag/indexador_lexico.py`), centralizado en `config`
  (`RAG_CHUNKING_ESTRUCTURAL` → `parametros.py` → `config_rag.CHUNKING_ESTRUCTURAL`),
  sin `os.getenv` fuera de `config/`.
- Default = `False` (opt-in). El operador activa el estructural con
  `RAG_CHUNKING_ESTRUCTURAL=1` sin tocar código.
- `_dividir()` usa `modulos.chunker_estructural.segmentar_estructural` cuando el
  flag está activo; si algo falla, cae con seguridad al corte por tamaño.
- Sin API. `modulos/chunker.py` (corte por tamaño) permanece intacto como
  respaldo.
