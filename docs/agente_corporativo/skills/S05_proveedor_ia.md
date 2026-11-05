---
name: S05-proveedor-ia
description: Errores de conexión con la IA (401, 403, 503, 429, timeouts), cambio de proveedor (Azure OpenAI, OpenAI, Anthropic), de modelo o deployment, y VIRF PLUS.
---

# S05 — Proveedor de IA, credenciales y modelos

## Cuándo usarla
- «rechazó la clave (401)», «faltan en el .env», 403 de política de contenido, 429, timeouts.
- Pasar de Claude a Azure GPT (o al revés), cambiar de deployment/modelo, usar uno más barato.
- Activar o corregir VIRF PLUS (GPT + Claude con árbitro).

## Qué subir (máx. 20)
`CONOCIMIENTO_VIRF.md`, esta skill, `config/ia.py`, `config/parametros.py`, `config/red.py`,
`config/virf_plus.py`, `.env.example`, `web/routers/config.py`, `web/services/env_service.py`,
`tests/test_estructura_config.py`, `tests/test_azure_mocks.py`, `tests/test_virf_plus.py`.
Si el fallo es de Solo Informe: `web/services/solo_informe_ia.py`, `tests/test_solo_informe_ia.py`.
**Nunca subir `.env`** (contiene claves). Describir qué variables hay (sin valores).

## Qué puede devolver (máx. 5)
`config/ia.py`, `config/parametros.py`, `.env.example`, y como mucho un servicio y un test.

## Configuración por proveedor (en `.env`, lo edita la persona, no el agente)
| Proveedor | Variables |
|---|---|
| Azure OpenAI (principal) | `IA_PROVEEDOR=azure_openai`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_DEPLOYMENT`, opcional `AZURE_OPENAI_API_VERSION` |
| OpenAI directo | `IA_PROVEEDOR=openai`, `OPENAI_API_KEY`, `OPENAI_MODELO` |
| Anthropic | `IA_PROVEEDOR=anthropic`, `ANTHROPIC_API_KEY` (empieza por `sk-ant-api03-`), `ANTHROPIC_MODELO`, opcional `ANTHROPIC_WORKSPACE_ID` |
| Solo Informe aparte | `SOLO_INFORME_IA_PROVEEDOR`, `SOLO_INFORME_DEPLOYMENT`, `SOLO_INFORME_MODELO` |

Prueba de humo (una llamada mínima al proveedor activo; funciona con cualquier proveedor pese al
nombre del flag): `.venv\Scripts\python.exe agente.py --verificar-conexion-azure`, o el botón
«Verificar conexión» de Configuración (endpoint en `web/routers/config.py`, que llama a
`config.ia.prueba_de_humo`).

## Pasos
1. Leer el error: 401 = clave inválida; 503 en la web = falta una variable (el mensaje la nombra);
   403 = filtro de contenido de Azure (`_diagnostico_filtro` explica la categoría); 429 = cuota.
2. Si es configuración: indicar a la persona qué variable corregir en `.env`; no tocar código.
3. Si es código: los clientes están en `config/ia.py` (`ClienteAzureOpenAIIA`,
   `ClienteAnthropicIA`, `ClienteOpenAIIA`), el único punto de instanciación es `_instanciar`,
   `get_cliente_ia()` usa la caché y `cliente_dedicado(proveedor, modelo)` crea uno aparte.
4. Reintentos/timeouts de red: `config/red.py` (`con_reintento`, `VIRF_HTTP_TIMEOUT`).

## Reglas que no se pueden romper
- La selección de proveedor vive SOLO en `config/` (test `test_la_seleccion_de_proveedor_vive_solo_en_config`).
- Sin cambio automático a otro proveedor si uno falla; y la palabra «fallback» no puede aparecer
  en el código vivo de `config/ia.py` (test `test_no_existe_fallback_automatico_en_el_codigo`).
- `FABRICAS[proveedor]()` aparece una sola vez (en `_instanciar`).
- Tests sin llamadas reales (`monkeypatch` de `FABRICAS` o `_llm_de_prueba`).
- El default de `IA_PROVEEDOR` en el código es `azure_openai`.

## Cómo verificar
```
.venv\Scripts\python.exe -m pytest tests/test_estructura_config.py tests/test_azure_mocks.py tests/test_virf_plus.py tests/test_solo_informe_ia.py -q
```

## Errores frecuentes
| Síntoma | Causa | Arreglo |
|---|---|---|
| 401 «invalid x-api-key» | Clave incompleta o con comillas/espacios | Corregir en `.env` y reiniciar la web |
| 400 «workspace» con Anthropic | Clave «identity-linked» | Añadir `ANTHROPIC_WORKSPACE_ID` |
| Azure «deployment not found» | Nombre de deployment ≠ nombre del modelo | Usar el nombre del deployment del portal |
| 403 content_filter | Política de contenido de Azure | Ver categoría en el mensaje; ajustar prompt |
| La web sigue usando la clave vieja | `.env` se lee al arrancar | Reiniciar uvicorn |
