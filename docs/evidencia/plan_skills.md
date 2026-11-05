# Plan de Skills (SK-WPs) — reconciliación guía ↔ código + mini-plan

**Fecha:** 2026-08-31 · **Hito base:** Golden-1 (`a4b5d57`) · **API:** ninguna (rebanada API-free).
**Guía:** `docs/guias/guia-integracion-skills-agente (1).md`.
**Regla de aislamiento:** todo el código nuevo vive bajo `skills/` (paquete nuevo,
top-level). NO se edita ningún módulo del Grupo B ni el subsistema prompt actual
`modulos/skills/` (se **envuelve/adapta**, no se elimina).

---

## FASE A.1 — Qué pide la guía (requisitos clave)

1. **6 skills** (Documentación, Calidad de Datos, Metodología, Gobierno, Implementación, Uso)
   declaradas como **herramientas de function calling** (JSON Schema de parámetros). Guía §1–§2.
2. **Coordinador/router con bucle ReAct** (`tools=`, `tool_choice="auto"`, feeding de
   observaciones), `max_iterations=5`. Guía §3.2 → **requiere API (Día 3)**.
3. Scripts deterministas en **sandbox** (el agente invoca, no calcula a mano). Guía §2/§4.B.
4. Salida final `ValidationReport` (**structured output**). Guía §3.1.
5. **Pruebas unitarias por skill** (pytest). Guía §4.B.
6. Antipatrones: exceso de agencia → cola `PENDIENTE` (HITL); `max_retries=3`;
   multi-tenancy por `model_id`; instrucción≠dato (IPI); obsolescencia (`is_active`). Guía §5.

## FASE A.2 — Qué YA existe (no rehacer; citar)

| Requisito guía | Ya implementado | Evidencia `archivo:línea` |
|---|---|---|
| Contrato salida tipada `ValidationReport` | Sí (WP-09) | `modulos/esquemas.py:56,65,123`; `EstadoRequisito`, `field_validator`, `model_validator` |
| Obsolescencia temporal `is_active` | Sí (WP-10) | `modulos/versionado.py` (ventana `is_active`) |
| Aislamiento multi-tenant (`model_id`) | Sí (D2, WP-02) | `modulos/aislamiento.py`; test `*spillover*` PASSED |
| Instrucción≠dato (IPI) | Sí (mecanismo, D5) | `motor/Mx_02:249-267` (separación datos/instrucción) |
| Abstención β (fuera de alcance → deriva) | Sí (D3, WP-01) | `modulos/abstencion.py:50,69` (`umbral_beta`, `evaluar_abstencion`), β=0.19 |
| Circuit breakers (cap iteraciones, fusibles) | Sí (D6, WP-05) | `config/parametros.py` `MAX_ITER`/`FUSIBLE_SEG`/`FUSIBLE_TOKENS` |
| Rubro versionado (config, no hardcode) | Sí | `modulos/cuestionamientos_ext.py:91` `catalogo_activo()`; `config/cuestionamientos_*.json`; `CUESTIONAMIENTOS_METODOLOGICOS` (`motor/Mx_00:300`); estándar `Estandares_VIRF.xlsx` (`Mx_01:726`) |
| Verificación determinista de citas | Sí (D4, WP-03) | `modulos/validador_citas.py`; `modulos/skills/base.py:280` `_verificar_citas` |
| Deps deterministas (sin añadir nada) | Sí | `requirements.txt`: pandas 2.1.4, numpy 1.26.4, **scikit-learn 1.4.2**, scipy, statsmodels |

> **Dos contadores distintos, no fundir:** `max_iterations=5` (bucle ReAct de razonamiento,
> guía §3.2, follow-on) ≠ `max_retries=3`/`INTENTOS_CONTRATO` (autocorrección, ya existe).

## FASE A.3 — Mapa 8 skills-prompt actuales → 6 skills de la guía

Subsistema actual: 8 skills basadas en prompt (`modulos/skills/catalogo.py`), salida
`ResultadoSkill(dict)` (`base.py:62`, **no Pydantic**), invocadas por `ejecutar_skills`
(`modulos/skills/__init__.py:88`) desde `agente.py:1867`. **Sin test unitario por skill.**

| Skill-prompt actual (`catalogo.py`) | Mapea a (6 de la guía) | Relación |
|---|---|---|
| `coherencia` (doc↔código) | Metodología + Implementación + Documentación | **solapa 3**; núcleo "código hace lo que el manual dice" |
| `supuestos` (estadísticos) | Metodología (+ Calidad de Datos) | parcial |
| `papers` (bibliografía) | Metodología / Documentación | parcial |
| `challenger` (alternativa + genera script) | Implementación / Metodología | parcial (script → sandbox) |
| `regulatorio` (SR11-7/EBA/BCBS vía RAG) | Gobierno / Metodología | parcial (RAG normativo) |
| `gobierno` (roles, segregación, aprobación) | **Gobierno** | **directo** |
| `datos` (marco de control BCBS 239) | **Calidad de Datos** (cualitativo) | directo **pero cualitativo** — NO es el motor determinista de la guía |
| `sintesis` (consolida y prioriza) | *Answer Validator / Critic* del diagrama | no es una de las 6; es el consolidador final |

### Huecos (skills de la guía SIN equivalente operativo hoy)

- **Calidad de Datos DETERMINISTA + sandbox** (outliers/nulos/duplicados/leakage por
  **cálculo**, no por prompt) → **no existe**. `SkillDatos` es cualitativa (evalúa el
  marco, no computa). **← este es el hueco que llena B.1.**
- **Documentación** tipada (parser de versión/hash/estructura PDF) → sin skill dedicada.
- **Metodología** como herramienta tipada `retrieve_methodology_evidence` con filtro
  `model_id` → el RAG existe (`src/rag/`) pero no declarado como tool tipada.
- **Implementación** tipada (AST + sandbox tests) → parcial: `modulos/analisis_estatico.py`
  (AST) y `challenger` (script), sin skill tipada declarada.
- **Uso** (ingesta `.msg`/`.eml`/`.pptx`) → **no existe**.

**Conclusión:** las 8 actuales son análisis-prompt (siguen sirviendo, se envuelven).
Las 6 de la guía son herramientas tipadas para function calling. Solo **1 de las 6**
(Calidad de Datos) es puramente determinista y se puede completar **hoy** sin API.

---

## FASE B — SK-WPs de HOY (API-free, tras checkpoint)

| WP | Objetivo | Archivos (nuevos, bajo `skills/`) | API | Mueve | Aceptación / DoD |
|---|---|---|---|---|---|
| **SK-00** | Contrato base tipado: entrada/salida Pydantic + `declaracion_herramienta()` (JSON Schema) + **alcance de revisión declarado** (puntero a rubro por ID desde config versionada) + fuera-de-alcance → abstención/deriva | `skills/base_tipada.py` | no | A3 (fundación) | test: importa, modelos validan, expone schema válido |
| **SK-01** | **Declarar las 6 skills** tipadas (firma + schema + alcance). 5 = stub declarado con marcador "pendiente coordinador API"; 1 (Calidad de Datos) a fondo en SK-02 | `skills/documentacion.py`, `calidad_datos.py`, `metodologia.py`, `gobierno.py`, `implementacion.py`, `uso.py`, `skills/__init__.py` | no | A3 **declaradas 6/6** | test: 6 exponen schema válido; cada una resuelve su rubro desde config (no texto hardcode); consulta fuera de alcance → abstención |
| **SK-02** | **Calidad de Datos COMPLETA** determinista en **sandbox** (subprocess): `run_data_quality_check(csv_path)->DataQualityResult` (outliers IQR, nulos, duplicados, leakage train/test) | `skills/calidad_datos.py` (+ ejecutable `python -m skills.calidad_datos`) | no | A3 **operativa-probada 1/6** | test §4.B: CSV con **10 outliers + 5 duplicados exactos** → JSON reporta esos números exactos |
| **SK-03** | Cola `PENDIENTE` (HITL, anti exceso de agencia): acciones de impacto externo se **encolan**, no se ejecutan | `skills/cola_pendiente.py` (+ `web/data/pendientes.json`) | no | A4 | test: acción de impacto → queda en cola, no se ejecuta |
| **SK-04** | Arnés `pytest` por skill + patrón documentado | `tests/skills/` | no | E1 | los tests de SK-00..SK-03 corren en la suite |

**Reporte honesto de A3:** *declaradas 6/6* vs *operativas y probadas 1/6* (Calidad de Datos).
No se marca 6/6 operativas.

---

## DIFERIDO A FOLLOW-ON (NO hoy)

| WP | Objetivo | Depende de |
|---|---|---|
| **SK-05** | **Coordinador ReAct + function calling** (`tools=`, `tool_choice="auto"`, feed de observaciones, `max_iterations=5`, salida `ValidationReport`) | **API (Día 3, domingo)** |
| **SK-06..SK-10** | Las otras 5 skills a fondo (Documentación, Metodología, Gobierno, Implementación, Uso), una por WP, cada una con su test | API + validación profunda |
| **SK-11** | Observabilidad por spans (OpenTelemetry/Phoenix) — P2 | — |

> **Recordatorio:** el **coordinador con function calling es del día de API (domingo)**;
> hoy no se cablea. Las 5 skills restantes quedan **declaradas** para encenderse con él.
