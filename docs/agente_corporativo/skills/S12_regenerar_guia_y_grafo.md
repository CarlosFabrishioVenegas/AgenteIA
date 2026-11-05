---
name: S12-regenerar-guia-y-grafo
description: Regenerar la guía PDF del agente corporativo y el grafo de código después de cambiar VIRF, para que la documentación no mienta.
---

# S12 — Regenerar la guía y el grafo

## Cuándo usarla
Después de cualquier cambio que añada o quite módulos, endpoints, tests o pantallas.

## Cómo
```
graphify update .                                               # 1) grafo (sin IA)
.venv\Scripts\python.exe scripts\generar_guia_agente.py --pdf    # 2) guía DOCX + PDF
.venv\Scripts\python.exe scripts\generar_presentacion_guia.py    # 3) presentación .pptx
```
- La presentación (`GUIA_VIRF_PRESENTACION.pptx`) reutiliza la tabla PANTALLAS de la guía; los
  pasos «Cómo se usa» de cada pantalla están en el diccionario `USO` de
  `scripts/generar_presentacion_guia.py`. Los ejemplos de calificación se calculan con
  `modulos/reglas_informe.py` y las reglas de oro / mapa síntoma → skill se leen de
  `CONOCIMIENTO_VIRF.md`.
- Sin `--pdf` solo genera el `.docx` (por ejemplo en una máquina sin Word). El PDF se crea
  abriendo el `.docx` en Word y exportando, o con LibreOffice:
  `soffice --headless --convert-to pdf docs/agente_corporativo/GUIA_VIRF_AGENTE_CORPORATIVO.docx`.
- Las capturas de pantalla viven en `docs/agente_corporativo/capturas/` (`NN_nombre.jpg`); si
  cambia una pantalla, reemplazar la imagen con el mismo nombre y regenerar.
- El texto explicativo vive en `CONOCIMIENTO_VIRF.md`, `00_LEEME.md` y `skills/*.md`; el resto
  (fichas de módulos, endpoints, tests, variables, comunidades) se extrae del código al generar.

## Qué subir (si se pide al agente que mejore la guía)
`scripts/generar_guia_agente.py`, `tests/test_guia_agente.py`, el `.md` a cambiar.
Para la presentación: `scripts/generar_presentacion_guia.py`, `tests/test_presentacion_guia.py`.

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_guia_agente.py tests/test_presentacion_guia.py -q
```
El test comprueba, entre otras cosas, que todas las rutas de archivo citadas en las skills
existen en el repositorio.
