---
name: S03-regla-calificacion
description: Cambiar o corregir la calificación del informe (Deficiente/Regular/Aceptable/Satisfactorio), la traducción puntaje→riesgo o qué cuenta como control incumplido. Un solo archivo de reglas para los tres informes.
---

# S03 — Regla de calificación y riesgos

## Cuándo usarla
- La conclusión de un informe no coincide con lo esperado.
- La gerencia cambia los umbrales (p. ej. «Deficiente con 2 críticas»).
- Cambia la traducción del puntaje del agente a riesgo, o el trato de las Oportunidades de Mejora.

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill, `modulos/reglas_informe.py`, `tests/test_solo_informe.py`
(clase `TestConcluir`, `TestControl`), `tests/test_plantilla_bcp.py` (`test_riesgo_desde_puntaje`).
Opcional: `modulos/plantilla_bcp.py` (cómo se muestra), `web/services/informe_web.py`
(cómo se aplica el chat).

## Qué puede devolver (máx. 5)
`modulos/reglas_informe.py` + los tests actualizados (2). Nada más: la regla es pura.

## Pasos
1. Leer `concluir()`: evalúa de lo más grave a lo menos grave; la primera condición gana.
   Los casos ambiguos están documentados como supuestos A1–A6 (se devuelven en `supuestos`).
2. Cambiar la condición y su texto `motivo`.
3. Actualizar/añadir casos en `TestConcluir` (parametrizados): un caso por umbral.
4. Si cambia `riesgo_desde_puntaje`, actualizar `test_riesgo_desde_puntaje` y la tabla de la
   «Escala de evaluación del agente» (anexo C del entregable, en `generador_word.py::escala`).

## Reglas que no se pueden romper
- La regla es una función pura (sin E/S, sin IA). Ningún informe calcula la calificación por
  su cuenta.
- «Aceptable» nunca lleva críticas (decisión del usuario).
- La falta de evidencia es falta crítica (política de la entidad, protegida en
  `test_regresion.py::TestAuditoriaSinEvidenciaEsFaltaCriticaNoAusenciaDeDictamen`).
- Una actividad sin puntaje por fallo técnico no se califica (no se culpa al validado).

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_solo_informe.py tests/test_plantilla_bcp.py -q
```

## Plantilla de pedido
```
Skill S03. Regla actual: <cita>. Regla nueva: <texto exacto de la gerencia>.
Casos de ejemplo que deben dar cada nivel: <lista>.
```

## Errores frecuentes
| Síntoma | Causa | Arreglo |
|---|---|---|
| Sale Regular con 3 moderadas Doc+Gob | Supuesto A2 (hueco entre «más de 3» y «menos de 3») | Confirmar con la gerencia y cambiar la condición |
| Una OM deja el control en rojo | `OM_INCUMPLE = True` | Debe ser `False` (decisión del usuario) |
