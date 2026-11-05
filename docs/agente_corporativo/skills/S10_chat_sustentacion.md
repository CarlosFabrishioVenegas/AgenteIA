---
name: S10-chat-sustentacion
description: Chat de sustentación (el área responde las observaciones y el agente decide APROBADO/NEGADO/PENDIENTE), consolidación de decisiones, puntaje final y acta de cierre en Excel.
---

# S10 — Chat de sustentación y consolidación

## Cómo funciona
- Endpoints en `web/routers/chat.py`: `POST /api/chat/iniciar`, `GET /api/chat/estado`,
  `GET /api/chat/observaciones`, `POST /api/chat/adjuntar` (evidencia nueva → re-evaluar),
  `POST /api/chat/consolidar`, `GET /api/chat/acta.xlsx`, WebSocket `/ws/chat`.
- Lógica: `web/services/chat_service.py` (clases `Observacion`, `Mensaje`, `SesionChat`;
  veredictos `SIN_TRATAR | APROBADO | NEGADO | PENDIENTE`).
- Consolidación: `modulos/consolidacion.py::consolidar_decisiones` (puntaje final: APROBADO =
  `VIRF_PUNTAJE_CUMPLE`, por defecto 5) y `tabla_final` (estado por actividad).
- Acta: `modulos/exportador_excel.py::acta_cierre_bytes` (3 hojas).
- Aprendizaje: las correcciones del chat se guardan como feedback (`modulos/feedback.py`).
- En el informe web, APROBADO = observación subsanada (no cuenta para la calificación).

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill, `web/routers/chat.py`, `web/services/chat_service.py`,
`modulos/consolidacion.py`, `modulos/exportador_excel.py`, `tests/test_consolidacion.py`,
`tests/test_acta_cierre.py`, `tests/test_chat_adjuntar.py`, `tests/test_aprendizaje_chat.py`.
Si afecta al informe: `web/services/informe_web.py`.

## Qué puede devolver (máx. 5)
Servicio de chat, consolidación, exportador y sus tests.

## Reglas que no se pueden romper
- La decisión final la toma el validador; el agente propone.
- El puntaje inicial del agente queda sellado y comparable; el final es aparte.
- Tests sin IA real (el chat se prueba con sesiones construidas a mano).

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_consolidacion.py tests/test_acta_cierre.py tests/test_chat_adjuntar.py tests/test_aprendizaje_chat.py -q
```
