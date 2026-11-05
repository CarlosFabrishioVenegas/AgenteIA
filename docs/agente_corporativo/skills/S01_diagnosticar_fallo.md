---
name: S01-diagnosticar-fallo
description: Triaje de cualquier fallo de VIRF cuando no se sabe en qué parte está. Devuelve qué skill aplicar y qué archivos pedir. Usar primero si el error es genérico o no se sabe qué subir.
---

# S01 — Diagnosticar un fallo

## Cuándo usarla
- Aparece un error y no está claro si es de la web, del pipeline, de la IA o del informe.
- Un resultado «se ve raro» (calificación, informe, pantalla vacía).
- Antes de cualquier otra skill, si el pedido no encaja claramente en una.

## Qué subir (máx. 20)
1. `CONOCIMIENTO_VIRF.md` y esta skill.
2. **La evidencia del fallo** (pégala en el mensaje, no cuenta como archivo): texto completo del
   error, la URL o pantalla, la captura y qué se pulsó.
3. Según dónde ocurrió (elige UNA fila, máx. 6 archivos):

| Dónde ocurre | Archivos |
|---|---|
| Pantalla de la web (botón, panel vacío, error rojo) | `web/templates/index.html`, `web/static/js/app.js`, el router de esa pantalla (tabla del §4 de CONOCIMIENTO), su servicio |
| Validación (paso 08) no termina o da error | `web/services/pipeline_runner.py`, `agente.py`, `modulos/motor_ia.py`, el log de la corrida (`outputs/…/*.log` o consola) |
| Informe Word | `modulos/plantilla_bcp.py` + el generador de ese informe (ver S02) |
| Error 401/403/503 de IA | `config/ia.py`, `config/parametros.py`, `.env.example` (nunca el `.env`) |
| Lectura de documentos o Excel | `motor/Mx_01_LecturaEvidencias.py`, `web/services/estandares_service.py` |

4. El test del área (sección 9 de CONOCIMIENTO).

## Qué puede devolver
Normalmente **ningún archivo**: un diagnóstico con (a) causa probable, (b) skill a aplicar,
(c) lista exacta de archivos a subir en el siguiente pedido. Si el arreglo es de 1–2 líneas y
evidente, puede devolver hasta 2 archivos corregidos.

## Pasos
1. Leer el mensaje de error de abajo arriba: la última línea dice QUÉ falló; las líneas
   `File "…", line N` dicen DÓNDE. Ese archivo:línea es el primer sospechoso.
2. Clasificar el código HTTP si lo hay: 400 entrada inválida (router), 401 sesión caducada,
   404 recurso inexistente, 502 la IA respondió mal, 503 faltan credenciales de IA, 500 error
   del servidor (traza en la consola de uvicorn).
3. Ubicar la función con el grafo (S11): `graphify explain "<función>"` da archivo y vecinos.
4. Reproducir con un test sin coste si es posible (los tests usan `tmp_path` y IA simulada).
5. Responder con la plantilla de diagnóstico.

## Reglas que no se pueden romper
Las 10 reglas del §8 de CONOCIMIENTO. En diagnóstico, además: no proponer «probar con la API
real» como primer paso; primero reproducir con mocks.

## Cómo verificar
El diagnóstico se valida cuando el test del área reproduce el fallo (falla antes del arreglo y
pasa después).

## Plantilla de diagnóstico
```
Síntoma: …
Causa probable: archivo:línea — explicación
Skill a aplicar: Sxx
Archivos para el siguiente pedido (≤18): …
Test que lo reproduce / debe pasar: …
```

## Errores frecuentes
| Error | Causa habitual | Skill |
|---|---|---|
| `sesión de login expirada` | El servidor se reinició (las sesiones viven en memoria) | Volver a entrar; no es un fallo |
| `no hay modelo activo` | Se llamó a un endpoint de sesión sin elegir modelo | S04 |
| `rechazó la clave (401)` | Clave mal copiada en `.env` (p. ej. sin la «s» inicial) | S05 |
| `faltan columnas obligatorias` | Cabecera del Excel distinta | S06 |
| Word sin imágenes de evidencia | El documento citado no está en la sesión o la cita no es literal | S09 |
| `ModuleNotFoundError` (torch, chromadb…) | Se usó el Python del sistema en vez de `.venv` | Usar `.venv` |
