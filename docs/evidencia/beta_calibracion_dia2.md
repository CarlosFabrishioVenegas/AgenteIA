# Calibración de β (abstención activa) — por datos, Día 2

**Fecha:** 2026-08-28 · **Señal:** `cobertura` léxica máxima por consulta (la
relevancia ABSOLUTA del buscador, comparable entre consultas). **Sin API.**

## Método

Se midió la `cobertura` máxima de los fragmentos recuperados (top_k=6) para dos
poblaciones de las preguntas-oro:
- **Respondibles:** las 24 preguntas del set, cada una a su dominio correcto.
- **No-respondibles:** las 4 adversarias, cada una al dominio equivocado.

## Distribución medida

| Población | n | mín | mediana | máx |
|---|---|---|---|---|
| Respondibles | 24 | **0.379** | 0.778 | 1.000 |
| No-respondibles (adversarias) | 4 | 0.000 | — | 1.000 |

Respondibles (ordenadas): 0.379, 0.522, 0.525, 0.621, 0.624, 0.627, 0.668,
0.675, 0.704, 0.765, 0.766, 0.775, 0.778, 0.796, 0.829, 0.876, 1.0×8.
Adversarias: **0.000, 0.000, 0.689, 1.000**.

## Hallazgo (honesto)

Las distribuciones **se solapan**: 2 de 4 adversarias puntúan cobertura alta
(0.689, 1.000) porque comparten vocabulario financiero genérico con el corpus.
`cobertura` NO separa por sí sola a esas adversarias — y no debe intentarlo: el
buscador ya las neutraliza por otra vía (declaración de alcance /
`reconocimiento_consulta`, verificada en la puerta: 4/4 declaran alcance). El
trabajo de β es distinto: distinguir **evidencia nula o marginal** de
**evidencia suficiente**.

## β elegido

**β = 0.19**, el punto medio del hueco de separación entre la evidencia ausente
(cobertura 0.0) y el mínimo de las respondibles (0.379):

- Abstiene ante evidencia nula/marginal (cobertura < 0.19).
- **No abstiene ninguna respondible** (todas ≥ 0.379 > 0.19) — verificado en
  `tests/test_beta_calibracion.py::test_todas_las_respondibles_superan_beta`.
- Configurable por el operador vía `VIRF_ABSTENCION_BETA` (0.0 = modo
  conservador previo; subirlo exige evidencia más fuerte).

Fijado en `config/parametros.py:ABSTENCION_BETA` (antes 0.0, ahora 0.19).
