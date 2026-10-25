---
id: investigacion-agentes-resultados
title: Resultados experimentales
---

# 11. Resultados

**No se ejecutaron experimentos propios. No existen resultados experimentales propios en esta entrega.** No se simulan resultados, costes ni rankings.

## Preparación del primer piloto

Se definió una prueba de instrumentación previa (2 tareas × 2 condiciones = 4 sesiones) y, después, una ablación pequeña de instrucciones (10 tareas × 2 condiciones × 3 repeticiones = 60 ejecuciones exploratorias), con repositorio/commit, herramientas, modelo y límites constantes. A fecha 2026-10-25 solo se verificó presencia/versiones de algunos CLIs; no se llamó a ningún modelo. Ollama no tiene un directorio de modelos en la ruta por defecto. Por tanto, aún no hay tareas ejecutadas, resultados, tokens ni costes que graficar.

La ruta para empezar sin coste queda pendiente de encontrar un modelo local ya disponible y compatible con un harness controlable. Si no existe, solicitar antes un límite de gasto para el piloto remoto y escoger proveedor/modelo con el usuario. No interpretar versiones CLI como evidencia de modelo configurado o acceso sin coste.

Los resultados públicos deberán conservar modelo exacto, versión del agente, prompt, harness, presupuesto, dataset y configuración. Los números comunicados por proveedores se etiquetarán como claims y no como ablación causal.

Siguiente fase: congelar y auditar las 10 tareas, preparar grader y logging, y ejecutar solo cuando haya inferencia realmente local sin coste o se haya fijado el límite de gasto para API. Después ampliar a 10–20 tareas si el piloto descubre problemas de protocolo.
