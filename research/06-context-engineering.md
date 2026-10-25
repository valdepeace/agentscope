---
id: investigacion-agentes-contexto
title: Context engineering y memoria
---

# 06. Context engineering

Contexto efectivo = system/developer + instrucciones proyecto + historial + archivos recuperados + resultados tools + plan/memoria + schemas + petición. Ventana máxima no equivale a tokens enviados, ni a recuperación pertinente.

Gemini CLI documenta búsqueda jerárquica y concatenación de `GEMINI.md`; `GEMINI_SYSTEM_MD` reemplaza totalmente el prompt (no merge), y variables insertan tools/skills/subagentes. `getCompressionPrompt` permite identificar un prompt de compactación. Codex publica lectura de `AGENTS.md` jerárquico; Qwen emplea `QWEN.md`. Evidencia enlazada en docs 4 y 13.

## Telemetría mínima

| Dimensión | Registrar |
|---|---|
| Contexto | bytes/tokens por bloque, origen, orden, prioridad |
| Selección | archivos descubiertos, recuperados, insertados, descartados |
| Compacción | historial antes/después, resumen, truncamiento |
| Coste | input/output/cache tokens, moneda, región, tarifa fechada |
| Memoria | persistencia, borrado, consulta y contenido recuperado |
| Límite | contexto solicitado/usado, rechazos y errores |

Presupuesto igual no equivale a información igual; comparar estrategia nativa y condición de tokens igual por separado. Distinguir resumen automático de memoria persistente y ventana larga.
