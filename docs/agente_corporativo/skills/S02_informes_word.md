---
name: S02-informes-word
description: Cambiar o corregir los informes Word de VIRF (estructura, estética, contenido, anexos, imágenes). Cubre los tres informes, que comparten la plantilla modulos/plantilla_bcp.py.
---

# S02 — Informes Word

## Cuándo usarla
- Cambiar la portada, encabezado, pie, índice, secciones I–VII, glosario o anexos.
- Un informe sale con texto mal, tablas desbordadas, imágenes faltantes o Word dice «contenido
  ilegible».
- Añadir información nueva a un control del barrido o a un anexo.

## Qué subir (máx. 20)
Siempre: `CONOCIMIENTO_VIRF.md`, esta skill, `modulos/plantilla_bcp.py`.
Más el informe afectado:

| Informe | Archivos |
|---|---|
| Solo Informe | `web/services/solo_informe_word.py`, `web/services/solo_informe_service.py`, `tests/test_solo_informe.py` |
| «Generar informe» (web) | `web/services/informe_web.py`, `modulos/capturas.py`, `tests/test_plantilla_bcp.py`, `tests/test_capturas.py` |
| Entregable del pipeline | `modulos/generador_word.py`, `modulos/word_estilo.py`, `tests/test_plantilla_bcp.py` (+ si toca firma/custodia: fragmentos de `tests/test_regresion.py` y `tests/test_pendientes_auditoria.py` que buscan esos textos) |

Opcional: `modulos/reglas_informe.py` si el cambio toca la conclusión (ver S03); un `.docx` o
PDF de ejemplo de `ejemplos/` para que el agente vea el resultado actual.

## Qué puede devolver (máx. 5)
- Cambio de **estética o estructura común** → `modulos/plantilla_bcp.py` (+ su test).
- Cambio de **contenido de un informe** → el generador de ese informe (+ su test).
- Nunca más de un generador por pedido si además cambia la plantilla.

## Pasos
1. Identificar si el cambio es común (plantilla) o de un informe (adaptador). Si los tres
   informes deben cambiar igual, va en la plantilla.
2. En la plantilla, las piezas son funciones: `_portada`, `_encabezado_pie`, `_indice`,
   `_objetivos`, `_alcance`, `_cumplimiento`, `_info_relevante`, `_ficha_documento`,
   `_cuadro_resumen`, `_glosario`, `_barrido`, y la API `generar(inf, ruta)`.
3. Los informes no pintan secciones: construyen el diccionario `inf` (ver docstring de
   `plantilla_bcp.generar`) y aportan ganchos: `detalle_control(doc, control, obs)`,
   `conclusion_extra(doc)`, `contexto`/`limitaciones` (texto o función) y `anexos` [(título, fn)].
4. Para destacar texto sin tablas usar `campo(doc, etiqueta, texto)` y `cita(doc, título, texto)`.
   Para fondos en párrafos usar `resaltar(parrafo, fill, acento)` (respeta el orden XML de Word).
5. Imágenes: `_imagen(doc, ruta, ancho_cm)` + `_caption(doc, pie)`.
6. Regenerar un ejemplo y abrirlo en Word para revisar visualmente.

## Reglas que no se pueden romper
- La calificación NO se calcula en los informes: sale de `reglas_informe.concluir`.
- Textos regulatorios que los tests exigen en el entregable del pipeline: «APROBACIÓN Y
  RESPONSABILIDAD», «CADENA DE CUSTODIA», «CORRECCIONES DEL VALIDADOR», «ANÁLISIS
  TRANSVERSALES» (sin distinguir mayúsculas), «SIN FIRMA DE VALIDADOR» si falta firmante, el hash
  del informe, «--verificar-cadena», el aviso de procedencia del corpus. No eliminarlos.
- Nunca el literal «None» en el documento.
- El logo se busca con `word_estilo.localizar_logo` vía módulo (los tests lo sustituyen).
- Orden XML de `w:pPr`: no añadir `w:shd`/`w:pBdr` con `append` al final; usar `resaltar`.

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_plantilla_bcp.py tests/test_solo_informe.py tests/test_capturas.py tests/test_informe_tabla_final.py -q
.venv\Scripts\python.exe -m pytest tests/test_regresion.py tests/test_pendientes_auditoria.py -q   # si tocó el entregable del pipeline
```
Visual: generar un ejemplo y abrirlo en Word (actualizar campos para el índice).

## Plantilla de pedido
```
Skill S02. Informe afectado: <Solo Informe | web | pipeline>.
Cambio: <qué debe verse distinto; adjunta captura/PDF actual>.
Devuélveme los archivos modificados (≤5) y el test que lo comprueba.
```

## Errores frecuentes
| Síntoma | Causa | Arreglo |
|---|---|---|
| Word: «contenido ilegible» | Elemento XML fuera de orden (p. ej. `w:shd` tras `w:jc`) | Usar `resaltar`; no `append` directo en `pPr` |
| Hoja en blanco entre anexos | La función del anexo termina con salto de página | La plantilla ya los quita (`_quitar_salto_final`); no añadir otro |
| Índice sin números | Campos sin actualizar | Abrir en Word (se actualiza solo: `updateFields`) |
| Tabla cortada entre páginas | Falta `no_partir(tabla)` | Añadirlo tras crear la tabla |
| Imagen de sustento ausente | Ruta inexistente | Ver S09 |
