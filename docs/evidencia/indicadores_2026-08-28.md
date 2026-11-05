# Paquete de Evidencia — Dia-1

- **Proyecto:** VIRF Agent
- **Fecha:** 2026-08-28
- **Commit:** `0e68d35`
- **Resumen:** 🟢 7 · 🟡 1 · 🔴 6 · ⚪ 15

| ID | Indicador | Valor | Umbral | Estado | Evidencia |
|----|-----------|-------|--------|--------|-----------|
| A1 | Fases E2E implementadas | 0/11 completas (7 parciales) | informativo | 🔴 ROJO | PLAN §4.1; Mx_01_LecturaEvidencias.py:72-91,170-179; motor_ia.py:402-444 |
| A2 | Checklist técnico cumplido | 0/12 completos | informativo | 🔴 ROJO | PLAN §3.2 (GAP-12..GAP-23) |
| A3 | Skills operativas (declaradas+test) | 0/6 guía (existen 8 skills-prompt sin test unitario) | informativo | 🔴 ROJO | modulos/skills/catalogo.py; base.py:62 ResultadoSkill(dict) no Pydantic |
| A4 | Antipatrones neutralizados con prueba | 2/11 | informativo | 🔴 ROJO | test_regresion.py:395; test_pendientes_auditoria.py:2022 |
| B1 | Recall@10 | NA | >=0.85 | ⚪ NA | API para fase posterior (IR/RAGAS requieren corrida real) |
| B2 | Precision@5 | NA | >=0.85 | ⚪ NA | API para fase posterior (IR/RAGAS requieren corrida real) |
| B3 | nDCG@10 | NA | informativo | ⚪ NA | API para fase posterior (IR/RAGAS requieren corrida real) |
| B4 | Tamaño Golden Dataset | NA | >=30 preguntas | ⚪ NA | API para fase posterior (IR/RAGAS requieren corrida real) |
| C1 | Faithfulness/Groundedness | NA | >=0.85 | ⚪ NA | API para fase posterior (IR/RAGAS requieren corrida real) |
| C2 | Answer Relevancy | NA | >=0.80 | ⚪ NA | API para fase posterior (IR/RAGAS requieren corrida real) |
| C3 | Citation Precision/Recall | NA | >=0.80 | ⚪ NA | API para fase posterior (IR/RAGAS requieren corrida real) |
| D1 | Inyección error matemático→DESVIADO | NA-diferido (fixture listo) | PASA | ⚪ NA | E2E requiere API; fixture presente |
| D2 | Contaminación model_id | 0% | 0% | 🟢 VERDE | pytest '*spillover*' PASSED |
| D3 | Abstención activa | PASA | PASA | 🟢 VERDE | pytest '*abstencion*' PASSED |
| D4 | Citas verificadas (determinístico; NLI diferido) | det=PASA (NLI diferido) | det=PASA; NLI>=0.80 diferido | 🟢 VERDE | pytest '*cita_determinista*' PASSED (parte NLI/entailment = NA-diferido, API) |
| D5 | Resistencia a IPI | NA-diferido | PASA | ⚪ NA | E2E requiere API; separación datos/instrucción en Mx_02:249-267 |
| D6 | Circuit breakers | cap+fusibles configurados (INTENTOS_CONTRATO=2) | SÍ+valores | 🟢 VERDE | config/parametros.py: MAX_ITER + FUSIBLE_SEG + FUSIBLE_TOKENS presentes |
| D7 | Linaje reconstruible E2E | NA | PASA | ⚪ NA | test '*linaje*' aún no existe en la suite |
| E1 | pytest passed/failed/skipped | 916 passed / 0 failed / 1 skipped | 0 failed | 🟢 VERDE | docs/evidencia/pytest_*.xml (JUnit) |
| E2 | Cobertura de tests | NA | >=60% núcleo | ⚪ NA | pytest-cov ausente en requirements.txt; no se añade sin autorización |
| E3 | Linter errores/warnings | NA | informativo | ⚪ NA | linter ausente en requirements.txt |
| E4 | Arranque FastAPI (verificación) | OK | OK | 🟢 VERDE | python -c 'import web.app' → returncode 0 |
| E5 | Dependencias nuevas introducidas | 0 | mínimas y justificadas | 🟢 VERDE | requirements.txt sin cambios respecto al baseline |
| F1 | .docx genera sin error | NA-diferido | SÍ | ⚪ NA | generación real requiere corrida; test '*docx_regresion*' aún no existe en la suite |
| F2 | Veredictos por ámbito presentes | NO (semáforo/DICTAMEN) | SÍ | 🔴 ROJO | grep APROBADO/PENDIENTE/NO APROBADO en generador_word.py |
| F3 | Secciones obligatorias (n/4) | 1/4 (sin_obs=1, pie_pag=0, refs=0, concl_ámbito=0) | 4/4 | 🔴 ROJO | inspección de generador_word.py / word_estilo.py |
| G1 | WPs completados | 6/8 | informativo | 🟡 AMBAR | wps_estado del propio paquete |
| G2 | Brechas Gap Matrix cerradas | ver wps_estado | informativo | ⚪ NA | se consolida al cierre del día |
| G3 | Archivos modificados en el hito | [".env.example", "config/__init__.py", "config/parametros.py", "docs/PLAN_MAESTRO_VIRF.md", "docs/evidencia/reconciliacion_dia1.md", "modulos/abstencion.py", "modulos/aislamiento.py", "modulos/chunker_estructural.py", "modulos/fusibles.py", "modulos/validador_citas.py", "src/rag/buscador_lexico.py", "tests/fixtures/var_injection/manual_var_demo.md", "tests/fixtures/var_injection/modelo_var_demo.py", "tests/fixtures/var_injection/modelo_var_roto.py", "tests/test_abstencion.py", "tests/test_aislamiento.py", "tests/test_chunker_estructural.py", "tests/test_circuit_breakers.py", "tests/test_citas_deterministas.py", "verificador_virf.py"] | lista | ⚪ NA | git diff --name-only f0bffc0 HEAD |

## WPs

- **WP-00**: HECHO → indicadores E1, E4, A1, A2, A3
- **WP-05**: HECHO → indicadores D6
- **WP-01**: HECHO → indicadores D3
- **WP-02**: HECHO → indicadores D2
- **WP-03**: HECHO → indicadores D4
- **WP-07**: EN_CURSO → indicadores D1
- **GAP-29-chunking**: HECHO → indicadores 
- **WP-08**: PENDIENTE → indicadores F1, F2, F3

## Regresiones

- (ninguna)

## Bloqueos

- Guías normativas ausentes en docs/guias/ (Decisión #1); Día 1 ejecutado contra el checklist embebido del plan
- Indicadores B/C (IR/RAGAS), D1 (inyección E2E), D5 (IPI E2E) y F1 (.docx real) requieren API: quedan NA-diferidos hasta encender crédito. Mecanismos y fixtures ya cableados y probados con mocks.
