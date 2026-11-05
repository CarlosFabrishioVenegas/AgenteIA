---
name: S09-evidencia-y-capturas
description: Subida y lectura de evidencia (manuales, ZIP de código, datasets, Excels 00–03) y las capturas («pantallazos») de portada, página citada y código que se insertan en el informe web.
---

# S09 — Evidencia: subida, lectura y capturas

## Cómo funciona
- Subida: `web/routers/upload.py` (destinos definidos en
  `web/services/contexto_sesion.py::_DESTINOS_SESION`; el ZIP de implementación se descomprime con
  límites anti ZIP-bomb). Entendimiento del código: `web/services/analisis_codigo.py`.
- Lectura para el pipeline: `motor/Mx_01_LecturaEvidencias.py` (`_leer_pdf`, `_leer_docx`, …),
  caché `modulos/document_cache.py`, troceo `modulos/chunker.py` / `chunker_estructural.py`.
- Capturas (sin IA): `modulos/capturas.py`
  - `localizar(nombre, raices)` busca el archivo citado (sesión y, de respaldo, `inputs/`).
  - `captura_portada(pdf)` → primera página.
  - `captura_cita(doc, cita)` → página del PDF con la cita **literal** resaltada; en .md/.docx,
    el pasaje con líneas numeradas. Si la cita no es literal en el PDF, no hay captura (evita
    presentar una página equivocada como sustento).
  - `captura_codigo(fragmento, archivo)` → código estilo editor con números de línea reales.
- Uso en el informe web: `web/services/informe_web.py` (`_Capturas`, `_portadas`,
  `_captura_cita`, `_detalle_control`): portada en Documentación (una sola vez), evidencia,
  y en cada contradicción «Sustento A» (manual) y «Sustento B» (código u otro documento), máx. 3
  contradicciones con captura por control.

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill y:
- Subida: `web/routers/upload.py`, `web/services/contexto_sesion.py`, `tests/test_upload_zip.py`.
- Lectura: `motor/Mx_01_LecturaEvidencias.py`, `modulos/document_cache.py`, `modulos/chunker.py`.
- Capturas: `modulos/capturas.py`, `web/services/informe_web.py`, `tests/test_capturas.py`, y si es
  posible el documento citado (o un extracto) y el fragmento del JSON de resultados con la cita.

## Qué puede devolver (máx. 5)
`modulos/capturas.py`, `web/services/informe_web.py`, `tests/test_capturas.py` (+ lectores si aplica).

## Pasos (capturas que no salen)
1. ¿El nombre del archivo citado coincide con uno subido? (`localizar` compara el nombre exacto,
   sin distinguir mayúsculas; tolera conectores como «y archivo.md»).
2. ¿La cita aparece literal en el PDF? Probar `captura_cita(ruta, cita, carpeta)`.
3. ¿Es un PDF escaneado sin texto? Entonces no hay texto que buscar (hace falta OCR, fuera de alcance).
4. Revisar `<sesión>/outputs/capturas/` (caché por contenido).

## Reglas que no se pueden romper
- No capturar páginas «aproximadas» como sustento (`solo_exacta=True` por defecto).
- Las capturas no usan IA y nunca tumban el informe (devuelven None si fallan).
- No ejecutar nada de lo subido (el código del modelo se lee, no se ejecuta).

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_capturas.py tests/test_upload_zip.py -q
```
