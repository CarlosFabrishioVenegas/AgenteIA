# Retrieval IR (size vs estructural) + decisión de chunking — Golden-1

**Fecha:** 2026-08-28 · **API:** ninguna (100% léxico/determinista).
**Arnés:** `scripts/preguntas_oro_formula.py` — 25 consultas discriminantes de
fórmula/tabla (ground truth verificado en el corpus) + 4 adversarias.
**Índices:** reconstruidos en temp por chunker (no tocan producción).

## Fase B — métricas IR sobre el golden discriminante (top_k=10)

| métrica | size-based | estructural | umbral |
|---|---|---|---|
| doc_recall@1 | 0.960 | 0.960 | — |
| doc_recall@3 | 1.000 | 1.000 | — |
| **doc_recall@10 (B1)** | **1.000** | 1.000 | ≥0.85 ✅ |
| ans@1 (chunk porta la respuesta) | **0.880** | 0.840 | — |
| ans@3 | 0.960 | **1.000** | — |
| ans@10 | 1.000 | 1.000 | — |
| **precision@5 (B2)** | **0.520** | 0.520 | ≥0.85 |
| **nDCG@10 (B3)** | **0.946** | 0.941 | informativo |

`ans@k` = fracción de consultas cuya `frase_clave` (la fórmula/tabla exacta)
aparece íntegra en algún chunk del top-k. Es el ground truth FINO: si el chunker
parte la fórmula, ningún chunk la contiene entera.

### Lectura de B1/B2/B3

- **B1 Recall@10 = 1.000** (≥0.85) → el documento correcto siempre entra en el
  top-10. Sobre un corpus de 2-4 docs/dominio, satura por construcción.
- **B2 Precision@5 = 0.520** (< 0.85). **No es un defecto de recuperación**: el
  techo estructural de Precision@5 a nivel CHUNK en este corpus es ~0.6, porque
  cada documento aporta solo 2-4 chunks (Precision@5 máx ≈ min(5, chunks_doc)/5).
  Que `ans@3 = 1.0` confirma que la respuesta SÍ se recupera; los otros puestos
  del top-5 se llenan con chunks del mismo dominio, no con ruido de otro modelo.
  Medible, por debajo del umbral por límite de granularidad → **ÁMBAR**.
- **B3 nDCG@10 = 0.946** (informativo, > 0.85): la respuesta se recupera en
  posiciones altas.

## Fase C — DECISIÓN DE CHUNKING: se queda OPT-IN (con evidencia)

**Resultado:** empate técnico. size gana marginalmente en `ans@1` (0.88 vs 0.84)
y `nDCG@10` (0.946 vs 0.941); estructural gana marginalmente en `ans@3` (1.0 vs
0.96). **Ninguna mejora consistente en los casos duros.** El criterio de
promoción ("mejora recall/nDCG en casos duros sin regresar") **no se cumple**.

### Por qué no discrimina (hallazgo raíz)

Los "casos duros" hipotetizados (fórmula partida a caballo entre chunks de
1200/200) **no se materializan en este corpus**:

1. El chunker por tamaño (`ChunkerService._dividir_en_chunks`) **no corta a
   mitad de frase** y, con documentos de ~5 KB + overlap 200, las fórmulas de una
   sola línea quedan íntegras en un chunk. Verificado: las 10 frase_clave de
   fórmula probadas están intactas en AMBOS chunkers.
2. El chunker estructural (`modulos/chunker_estructural.py`) solo obtiene ventaja
   cuando hay **encabezados markdown (`#`), tablas (`|`) o bloques cercados
   (` ``` `)** que tratar como unidades atómicas. Los PDFs del corpus, extraídos
   a **prosa plana**, no tienen ninguno de esos marcadores → el estructural
   produce esencialmente los MISMOS chunks que el de tamaño.
3. Las tablas/secciones grandes (zonas de Basilea verde/ámbar/roja, 11 principios
   de BCBS 239) exceden 1200 chars y **las parten AMBOS** chunkers por igual.

### Consecuencia

- **El chunking estructural permanece OPT-IN** (`RAG_CHUNKING_ESTRUCTURAL=1`),
  como desde el Día 2. No hay evidencia que respalde promoverlo a default.
- **NO se toca `test_chunks_identicos_no_se_pisan_entre_si`**: como no se
  promueve, su acoplamiento al corte por tamaño es irrelevante hoy. Desacoplarlo
  sin necesidad sería tocar un test ajeno sin justificación.
- **Hallazgo para el futuro** (no ejecutado hoy, fuera de alcance): el estructural
  solo se podrá discriminar con un corpus que conserve estructura markdown/tablas
  (p. ej. ingesta OCR jerárquica, GAP-13) o con documentos donde una tabla real
  supere el límite de chunk. Con el corpus actual (prosa PDF plana) la decisión
  es indiferente por construcción.

## Indicadores movidos

`B1` NA→VERDE (1.000) · `B2` NA→ÁMBAR (0.520, techo de corpus) · `B3` NA→VERDE
(0.946) · `B4` NA→VERDE (49 discriminantes: 25 fórmula/tabla + 24 base).
`C1/C2/C3` siguen NA (generación, requieren API).
