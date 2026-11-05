---
name: S04-web-pantallas-endpoints
description: Corregir o crear pantallas, botones y endpoints de la web de VIRF (FastAPI + JavaScript sin framework). Incluye estilos, responsive y accesibilidad.
---

# S04 — Web: pantallas y endpoints

## Cuándo usarla
- Un botón no hace nada, un panel queda vacío, aparece un error rojo en la web.
- Añadir un campo, una pantalla o un endpoint.
- Problemas de estilo, móvil o accesibilidad.

## Cómo está armada la web
- `web/app.py` registra los routers y sirve `index.html` (con versión de caché de CSS/JS).
- `web/templates/index.html`: TODAS las pantallas (`.pantalla` login/modelos/Solo Informe, `.app`
  dashboard) y vistas (`#vista-*`) en un solo HTML; se muestran/ocultan con `hidden`.
- `web/static/js/app.js`: lógica por bloques (`cablear*` al arrancar, `abrir*` por vista,
  `pedirJSON(url, opciones)` para llamar a la API). `guia.js`: guía de pasos y ayuda.
- `web/static/css/styles.css`: tokens en `:root` (`--tinta`, `--capa`, `--texto`, `--texto-mu`,
  `--sello`, `--sello-txt`, `--ok`, `--warn`…). Responsive en `@media (max-width: 760px)`.
- Cada pantalla ↔ un router `web/routers/<x>.py` (prefijo `/api/<x>`) ↔ un servicio
  `web/services/<x>.py`. Sesión: `Depends(contexto_sesion)` (requiere login + modelo activo).

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill, y:
- Pantalla: `web/templates/index.html`, `web/static/js/app.js` (o `guia.js`), `web/static/css/styles.css`.
- Backend de esa pantalla: `web/routers/<x>.py`, `web/services/<x>.py`, `web/app.py` (si es router nuevo).
- Test del área (`tests/test_<x>.py`).
Para localizar el router: busca en `app.js` la URL `/api/...` que llama el botón.

## Qué puede devolver (máx. 5)
Típico: `index.html` + `app.js` + `styles.css` + router + servicio. Si además hace falta test,
partir en dos pedidos: (1) backend + test, (2) frontend.

## Pasos
1. Backend primero: endpoint en el router (modelo Pydantic para el cuerpo, `HTTPException` con
   mensaje claro: 400 entrada, 404 no existe, 503 falta configuración), lógica en el servicio.
2. Registrar el router en `web/app.py` si es nuevo.
3. Frontend: HTML de la vista, función `abrir<X>()` y `cablear<X>()` en `app.js`, llamada a
   `pedirJSON`. Mostrar errores en un `.verificacion.is-error`.
4. Estilos reutilizando clases existentes (`panel`, `config-section`, `field`, `btn`,
   `btn--primary`, `grid-2`, `tabla-modelos`). Recordar: `.field` y `.grid-2` fijan `display`, así
   que para ocultarlos con `hidden` hace falta la regla `[hidden]{display:none!important}` del
   contenedor.
5. Comprobar en escritorio y a 375 px de ancho; foco visible con Tab.

## Reglas que no se pueden romper
- Endpoints de sesión con `Depends(contexto_sesion)`; los que escriben en rutas globales deben
  validar nombres (no aceptar `..`).
- No guardar estado importante en `localStorage`.
- Contraste de texto ≥ 4.5:1 (usar tokens, no colores sueltos).
- No llamar a la IA desde el frontend; siempre vía endpoint.

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_<area>.py -q
.venv\Scripts\python.exe -m uvicorn web.app:app --port 8000   # y probar en el navegador
```
Los tests de endpoints usan `fastapi.testclient.TestClient` y `app.dependency_overrides` para
sustituir la sesión (ver `tests/test_solo_informe.py::cliente`).

## Errores frecuentes
| Síntoma | Causa | Arreglo |
|---|---|---|
| Cambio de CSS/JS no se ve | Caché del navegador | `index()` añade `?v=<mtime>`; recargar |
| Campo no se oculta con `hidden` | La clase fija `display` | Regla `[hidden]{display:none!important}` en el panel |
| 401 tras reiniciar el servidor | Sesiones en memoria | Volver a entrar |
| 400 «no hay modelo activo» | Endpoint de sesión sin modelo | Entrar a un modelo o usar dependencia de solo login |
