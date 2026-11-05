# Paquete de Evidencia — Dia-2

- **Proyecto:** VIRF Agent
- **Fecha:** 2026-08-28
- **Commit:** `5a7f015`
- **Resumen:** 🟢 9 · 🟡 2 · 🔴 3 · ⚪ 15

| ID | Indicador | Valor | Umbral | Estado | Evidencia |
|----|-----------|-------|--------|--------|-----------|
| A1 | Fases E2E implementadas | 0/11 completas (7 parciales) | informativo | 🔴 ROJO | PLAN §4.1; Mx_01_LecturaEvidencias.py:72-91,170-179; motor_ia.py:402-444 |
| A2 | Checklist técnico cumplido | 1/12 completos (pto12 versionado=1; pto3 salida tipada parcial=1) | informativo | 🟡 AMBAR | PLAN §3.2 (GAP-12..GAP-23); WP-10 modulos/versionado.py; WP-09 modulos/esquemas.py (salida tipada, extracción aún free-parse) |
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
| E1 | pytest passed/failed/skipped | 950 passed / 0 failed / 1 skipped | 0 failed | 🟢 VERDE | docs/evidencia/pytest_*.xml (JUnit) |
| E2 | Cobertura de tests | NA | >=60% núcleo | ⚪ NA | pytest-cov ausente en requirements.txt; no se añade sin autorización |
| E3 | Linter errores/warnings | NA | informativo | ⚪ NA | linter ausente en requirements.txt |
| E4 | Arranque FastAPI (verificación) | OK | OK | 🟢 VERDE | python -c 'import web.app' → returncode 0 |
| E5 | Dependencias nuevas introducidas | 0 | mínimas y justificadas | 🟢 VERDE | requirements.txt sin cambios respecto al baseline |
| F1 | .docx genera sin error (pipeline real) | NA-diferido | SÍ | ⚪ NA | generación E2E desde el pipeline requiere API; estructura probada por WP-08 (reporte_docx.py) desde mock [pytest '*docx_regresion*' PASSED] |
| F2 | Veredictos por ámbito presentes | SÍ | SÍ | 🟢 VERDE | APROBADO/PENDIENTE/NO APROBADO en reporte_docx.py + test docx_regresion |
| F3 | Secciones obligatorias (n/4) | 4/4 (sin_obs=1, pie_pag=1, refs=1, concl_ámbito=1) | 4/4 | 🟢 VERDE | inspección de reporte_docx.py / word_estilo.py + test docx_regresion |
| G1 | WPs completados | 5/8 | informativo | 🟡 AMBAR | wps_estado del propio paquete |
| G2 | Brechas Gap Matrix cerradas | ver wps_estado | informativo | ⚪ NA | se consolida al cierre del día |
| G3 | Archivos modificados en el hito | [".env.example", "config/__init__.py", "config/parametros.py", "docs/PLAN_MAESTRO_VIRF.md", "docs/evidencia/beta_calibracion_dia2.md", "docs/evidencia/indicadores_2026-08-28.json", "docs/evidencia/indicadores_2026-08-28.md", "docs/evidencia/recall_cutover_dia2.md", "docs/evidencia/reconciliacion_dia1.md", "docs/evidencia/reconciliacion_dia2.md", "docs/evidencia/wps_dia1.json", "docs/evidencia/wps_dia2.json", "modulos/abstencion.py", "modulos/aislamiento.py", "modulos/chunker_estructural.py", "modulos/esquemas.py", "modulos/fusibles.py", "modulos/reporte_docx.py", "modulos/validador_citas.py", "modulos/ventana_contexto.py", "modulos/versionado.py", "src/rag/buscador_lexico.py", "src/rag/config_rag.py", "src/rag/indexador_lexico.py", "tests/fixtures/var_injection/manual_var_demo.md", "tests/fixtures/var_injection/modelo_var_demo.py", "tests/fixtures/var_injection/modelo_var_roto.py", "tests/test_abstencion.py", "tests/test_aislamiento.py", "tests/test_beta_calibracion.py", "tests/test_chunker_estructural.py", "tests/test_circuit_breakers.py", "tests/test_citas_deterministas.py", "tests/test_docx_regresion.py", "tests/test_esquemas.py", "tests/test_versionado.py", "verificador_virf.py"] | lista | ⚪ NA | git diff --name-only f0bffc0 HEAD |

## WPs

- **WP-09**: HECHO → indicadores A2
- **WP-10**: HECHO → indicadores A2
- **WP-08-estructura**: HECHO → indicadores F2, F3
- **cutover-chunking**: HECHO → indicadores 
- **beta-calibracion**: HECHO → indicadores D3
- **WP-04**: PENDIENTE → indicadores D4
- **WP-06**: PENDIENTE → indicadores B1, B2, B4
- **WP-07**: PENDIENTE → indicadores D1, D5

## Regresiones

- (ninguna)

## Bloqueos

- Indicadores B/C (IR/RAGAS), D1 (inyección E2E), D5 (IPI E2E) y F1 (.docx real) requieren API: quedan NA-diferidos hasta encender crédito. Mecanismos y fixtures ya cableados y probados con mocks.
