---
name: S06-estandar-validacion
description: El Excel de estándar (Ámbito, Control, Actividad, Tier I–IV) no se lee, faltan actividades, el filtro por Tier falla o cambia el formato del Excel.
---

# S06 — Estándar de validación (Excel)

## Cuándo usarla
- «Excel inválido: faltan columnas», actividades que no aparecen, ámbitos mal agrupados.
- El filtro por Tier deja fuera o dentro actividades que no debería.
- La gerencia cambia el formato del estándar (columnas nuevas, otra hoja).

## Hay tres lectores (por razones distintas)
| Lector | Uso | Archivo |
|---|---|---|
| `leer_estandares_virf(tier)` | Pipeline CLI (ruta global `RUTAS["estandares_virf"]`) | `motor/Mx_01_LecturaEvidencias.py` |
| `estandares_service.parsear` | Paso 02 de la web (por sesión, exige cabecera en fila 1 y TIER I–IV) | `web/services/estandares_service.py` |
| `solo_informe_service.parsear_estandar` | Solo Informe (tolerante: cabecera por palabra clave en las 30 primeras filas, Control General/Específico, TIER opcional) | `web/services/solo_informe_service.py` |

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill, el lector afectado, su test
(`tests/test_estandares_sesion.py` o `tests/test_solo_informe.py`), `config/dominio.py`
(`normalizar_tier`), `motor/Mx_00_CargaRutas.py` (`redirigir_estandar`, flag `--estandares`) y,
si se puede, **el Excel** que falla (o una captura de su cabecera y 5 filas).

## Qué puede devolver (máx. 5)
El lector + su test (+ `config/dominio.py` si cambian los tiers).

## Pasos
1. Abrir el Excel: ¿en qué fila está la cabecera? ¿cómo se llaman las columnas? ¿hay celdas
   combinadas (ámbito/control)? openpyxl solo da valor a la primera celda combinada: el código
   propaga el último valor hacia abajo (forward-fill).
2. Comparar con el lector: palabras clave reconocidas (`ambito`/`dimension`, `actividad`/`tarea`,
   `control`, `especific`, `tier i..iv` o `tier 1..4`).
3. Ajustar el reconocimiento SIN adivinar el modo: si el formato nuevo es otro, añadir la
   palabra clave; si es incompatible, error explícito con el nombre de la columna que falta.
4. Añadir un test con un Excel construido en memoria (openpyxl) que reproduzca el formato.

## Reglas que no se pueden romper
- Sin detección «mágica»: el estándar alternativo se pasa con `--estandares <ruta>`.
- Una actividad huérfana (sin ámbito) se descarta, no se inventa ámbito.
- Filas de totales/notas (`Total…`, `Nota:`) no son actividades.
- La marca de tier cuenta como «aplica» si no está vacía ni es `no/n/false/0`.

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_estandares_sesion.py tests/test_solo_informe.py -q
```
`test_estandar_real_del_repo` lee `inputs/Estandares_VIRF.xlsx` real.

## Errores frecuentes
| Síntoma | Causa | Arreglo |
|---|---|---|
| Paso 02 rechaza el Excel real | Cabecera en fila 2 (fila 1 vacía) | Usar el lector tolerante o mover la cabecera |
| Actividades con ámbito equivocado | Celda combinada no propagada | Revisar forward-fill de la columna |
| Tier IV muestra todas las actividades | Sin columnas TIER | El estándar no distingue tiers: es lo esperado |
