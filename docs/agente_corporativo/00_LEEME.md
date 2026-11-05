# Paquete para el agente corporativo — cómo usarlo

Este paquete permite que un agente de IA con **acceso limitado** (máximo **20 archivos de
entrada** y **5 archivos devueltos** por pedido) revise VIRF, diagnostique fallos y proponga
cambios seguros.

## Qué contiene

| Archivo | Para qué | ¿Subirlo siempre? |
|---|---|---|
| `GUIA_VIRF_AGENTE_CORPORATIVO.pdf` | Guía completa (+200 páginas): qué hace cada parte, capturas, catálogo de módulos, endpoints, tests, variables y grafo. | Solo para estudio o dudas generales: ocupa 1 archivo pero es largo. |
| `CONOCIMIENTO_VIRF.md` | Resumen operativo: arquitectura, reglas, mapa síntoma → skill. | **Sí** (1 archivo). |
| `skills/Sxx_*.md` | Una «skill» por tipo de cambio: qué archivos subir, qué puede devolver, pasos, tests. | **Sí, solo la que aplique** (1 archivo). |
| `docs/agente_corporativo/capturas/` | Pantallas de la web (también están dentro del PDF). | No. |

## Cómo armar cada pedido (regla 20 / 5)

1. Sube **`CONOCIMIENTO_VIRF.md`** + **la skill** del tipo de cambio → 2 archivos.
2. Sube los archivos de código que la skill indica en «Qué subir» → **máximo 18**.
   Si el problema no está claro, empieza por **S01 — Diagnosticar un fallo**: te dice qué skill
   usar y qué archivos pedir.
3. El agente devuelve **como máximo 5 archivos modificados**. Cada skill indica cuáles son los
   que normalmente cambian («Qué puede devolver»). Si un cambio necesita tocar más de 5, la skill
   explica cómo **partirlo en varios pedidos** que dejen el sistema funcionando en cada paso.
4. Después de aplicar los archivos devueltos, ejecuta los tests que indica la skill
   (siempre con el entorno `.venv`, sin llamadas reales a la IA).

## Plantilla de pedido

```
Contexto: te adjunto CONOCIMIENTO_VIRF.md y la skill <Sxx>.
Problema: <qué falla o qué quieres cambiar; pega el error o la captura si la hay>.
Archivos adjuntos: <lista>.
Pide: diagnóstico + archivos modificados completos (máx. 5) + tests que debo correr.
Respeta las «Reglas que no se pueden romper» de la skill.
```

## Regenerar este paquete

El PDF se genera desde el código (no se escribe a mano):

```
.venv\Scripts\python.exe scripts\generar_guia_agente.py --pdf
```

Ver la skill **S12 — Regenerar la guía y el grafo**.
