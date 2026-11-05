---
name: S11-usar-el-grafo
description: Usar el grafo de código (graphify) para encontrar qué archivos subir al agente, qué se rompe si se cambia algo y cómo se conectan las piezas, respetando el límite de 20 archivos.
---

# S11 — Usar el grafo para acotar archivos

## Qué es
`graphify-out/` contiene un mapa del código construido sin IA a partir del árbol sintáctico:
- `GRAPH_REPORT.md`: resumen legible — más de 300 comunidades con nombre en español
  («Informes · Plantilla común BCP», «Configuración · Clientes IA y proveedor»…), nodos
  centrales («god nodes»), conexiones sorprendentes y preguntas sugeridas.
- `graph.json`: el grafo (nodos = funciones, clases, módulos; aristas = llama, importa, contiene).
- `graph.html`: vista visual por comunidades (ábrelo en el navegador).

## Comandos (en la raíz del proyecto, no necesitan IA)
| Necesito… | Comando |
|---|---|
| Ver dónde vive una función y con qué se conecta | `graphify explain "concluir()"` |
| Saber qué se rompe si cambio algo | `graphify affected "riesgo_desde_puntaje()"` |
| Ver cómo llega un endpoint a una función | `graphify path "ia_pulir()" "cliente_dedicado()"` |
| Responder una pregunta en lenguaje natural | `graphify query "cómo se capturan las páginas del manual"` |
| Los nodos más conectados (cambiarlos es delicado) | `graphify god-nodes --top 15` |
| Actualizar el grafo tras cambiar código | `graphify update .` |

Si `graphify` no está instalado en la máquina corporativa, usar `GRAPH_REPORT.md` (texto) y la
sección «Comunidades del grafo» del PDF.

## Receta: elegir los ≤18 archivos de código
1. Empezar por la función o endpoint del síntoma (`explain`).
2. `affected` sobre ella: los archivos que aparecen son los candidatos a romperse.
3. Quedarse con: el archivo de la función + los que la llaman directamente + su test.
4. Si salen más de 18, priorizar: (a) el archivo a modificar, (b) su test, (c) los que la
   llaman, (d) los que ella llama. Los demás se describen en el mensaje (nombre + qué hacen,
   según la ficha del PDF).

## Limitaciones
- No une JavaScript con Python (HTTP). Buscar la URL `/api/...` de `app.js` en `web/routers/`.
- Se queda desactualizado si no se corre `graphify update .` tras los cambios.
- Las carpetas `archivo/` y `codigo_muerto_*` están en el grafo pero NO son código vivo.
