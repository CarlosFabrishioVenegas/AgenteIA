---
name: S08-calidad-datos
description: Controles de calidad de datos (motores genéricos y repositorio de controles por dominio), su ejecución y el notebook entregable de controles.
---

# S08 — Calidad de datos, motores y notebook de controles

## Cómo funciona
- **Motores genéricos** (`modulos/motores_datos.py`): completitud, duplicados, integridad
  referencial, missing, rangos… Catálogo visible en «Controles de datos». Ejecución:
  `ejecutar_motor`, `motores_disponibles`.
- **Motores de entidad** (`modulos/motores_entidad.py`): resultados del área (no del agente).
- **Repositorio por dominio**: controles como funciones Python en
  `inputs/repositorio_controles/<dominio>/*.py` (p. ej. `inputs/repositorio_controles/var/christoffersen.py`,
  `inputs/repositorio_controles/alm/delta_eve.py`); `modulos/repositorio_controles.py` los carga.
- **Notebook entregable**: `modulos/notebook_controles.py` (8 columnas) y
  `modulos/controles_a_notebook.py` / `excel_a_notebook.py` (desde documento o Excel); endpoint
  `GET /api/motores/notebook` en `web/routers/controles.py`.
- Pipeline por control: `agente.py --control` + `motor/Mx_04_MotorControlesData.py`.
- Pipeline de calidad de datos por script: `scripts/pipeline_calidad_datos.py`.

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill, y según el caso:
- Motor genérico: `modulos/motores_datos.py`, `tests/test_regresion.py` (clase `TestMotoresDatos`; puede pegarse solo esa clase).
- Control de un dominio: el `.py` del control, `modulos/repositorio_controles.py`, `modulos/notebook_controles.py`, `tests/test_notebook_controles.py`.
- Notebook: `modulos/notebook_controles.py`, `modulos/controles_a_notebook.py`, `web/routers/controles.py`, `tests/test_endpoint_notebook_controles.py`.
- Un dataset de muestra pequeño (CSV, sin datos personales reales).

## Qué puede devolver (máx. 5)
El motor o control + el generador de notebook + su test.

## Pasos
1. Un control nuevo: copiar la estructura de uno existente del mismo dominio (función
   `control(...)` con docstring de una línea que describe la prueba).
2. Registrar su metadato (nombre, requeridos, opcionales, ejemplo) igual que los demás.
3. Test con dataset sintético en `tmp_path`.

## Reglas que no se pueden romper
- Los controles son deterministas (sin IA); la IA solo redacta, nunca calcula el resultado.
- Un dataset ausente no revienta: el control informa «no ejecutable».
- Sin datos personales reales en tests ni en ejemplos.

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_notebook_controles.py tests/test_endpoint_notebook_controles.py -q
.venv\Scripts\python.exe -m pytest tests/test_regresion.py -q -k "Motores"
```
