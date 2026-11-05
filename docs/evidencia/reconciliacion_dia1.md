# Reconciliación Día 1 — guía ↔ plan

**Fecha:** 2026-08-28 · **Commit baseline:** `f0bffc0`

## Estado de las guías normativas
Las tres guías **NO están en el repositorio** al arrancar el Día 1:
- `docs/guias/guia-arquitectura-agente-financiero.md` — **AUSENTE**
- `docs/guias/guia-integracion-skills-agente.md` — **AUSENTE**
- `docs/guias/guia-errores-loops-validacion-rag.md` — **AUSENTE**

Verificado: `docs/guias/` no existe (solo `docs/PLAN_MAESTRO_VIRF.md`).

## Decisión humana aplicada
Consultado el bloqueo, la instrucción fue **"Proceder con el plan"**: ejecutar el Día 1
(WP-00 + fundaciones P0 API-free) usando como fuente el **checklist normativo embebido**
ya codificado en `docs/PLAN_MAESTRO_VIRF.md` (§2.1–§2.7 del Prompt Maestro 01).

## Consecuencia / deuda abierta
- No se pudo hacer la reconciliación real guía↔plan (las guías no existen).
- Si al llegar las guías reales difieren del checklist embebido, se re-audita el alcance.
- Este documento queda como constancia de que el Día 1 se ejecutó contra el checklist
  embebido, no contra las guías originales. **Bloqueo activo para el Día 2.**

## Entorno de medición detectado (para WP-00)
- `pytest-cov` / `coverage`: **ausentes** en `requirements.txt` → indicador `E2` = NA (no se
  añade la dependencia sin autorización).
- Linter (flake8/ruff/pylint/mypy): **ausente** en `requirements.txt` → indicador `E3` = NA.
- Verificador de integridad existente: `validate_project.py` (148 checks, coste 0).
