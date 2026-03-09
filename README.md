Correciones 

[
  {
    "dimension": "Calidad de Datos",
    "actividad": "Identificar y tratar outliers con metodología justificada",
    "el_agente_dijo": "No Aceptado",
    "correcto_es": "Revisado y Aceptado",
    "razon": "El documento sí menciona IQR en la sección 4.2 — el agente no lo detectó porque buscaba 'outlier' y el doc dice 'valor extremo'",
    "leccion": "Buscar también los términos: valor extremo, dato anómalo, dato atípico, extreme value"
  }
]



# Cargar feedback previo para calibrar el motor
ruta_feedback = Path("feedback/correcciones.json")
if ruta_feedback.exists():
    import json
    correcciones = json.loads(ruta_feedback.read_text(encoding="utf-8"))
    log(f"Feedback cargado: {len(correcciones)} corrección(es) previas", "info")
    # Inyectar en las instrucciones del motor
    for dim in INSTRUCCIONES_DIMENSION:
        casos = [c for c in correcciones if c["dimension"] == dim]
        if casos:
            ejemplos = "\n".join(
                f'- Si ves "{c["leccion"]}", esto equivale a: {c["actividad"]} → {c["correcto_es"]}'
                for c in casos
            )
            INSTRUCCIONES_DIMENSION[dim] += f"\n\nCORRECCIONES APRENDIDAS:\n{ejemplos}"
            
