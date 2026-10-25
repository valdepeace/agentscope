---
id: investigacion-agentes-prompts
title: System prompts: procedencia y comparación
---

# 04. System prompts

Para leer extractos cortos de cada agente y comprobar qué prompts no están publicados, consulta el [muestrario de prompts](../docs/14-muestras-prompts.md).

## Procedencia

- **A:** código oficial; **B:** documentación oficial; **C:** captura comunitaria verificable; **D:** inferencia/reconstrucción; **E:** no disponible.
- **A** no implica que el archivo sea el prompt único o efectivo en todos los modos. Registrar API/modelo, configuración, tools habilitadas y hash del prompt serializado.
- Codex: `codex-rs/models-manager/prompt.md` es texto de prompt publicado, pero la ruta no prueba que sea el único prompt runtime.
- Gemini CLI: docs permiten exportar/reemplazar prompt (`GEMINI_SYSTEM_MD`, reemplazo entero); variables `${AgentSkills}`, `${SubAgents}`, `${AvailableTools}` y nombres de tools se resuelven dinámicamente. `getCoreSystemPrompt` y `getCompressionPrompt` delegan en `PromptProvider`.
- OpenCode: `packages/opencode/src/session/system.ts` construye instrucciones con skills/MCP, configuración de proveedor/modelo. Docs permiten primary/subagents.
- Qwen Code: `packages/core/src/core/prompts.ts`; override/append, reglas `QWEN.md` y subagentes configurable por prompt/model/tools.
- Cline CLI: `apps/cli/src/runtime/prompt.ts` agrega metadata workspace/reglas/modo; no extrapolar a todas sus superficies.
- OpenHands SDK separa prompt estático, herramientas y contexto dinámico; no atribuir SDK a producto hospedado.
- Claude Code: captura de prompt de creación de agentes en repo no demuestra prompt principal. Colecciones comunitarias quedan C solo al verificar método/hash/licencia.
- Cursor/Windsurf/Devin: prompt principal completo no verificado públicamente (E).

## Matriz de evidencia (no de calidad)

| Instrucción/capacidad | Codex | Gemini | OpenCode | Qwen | Cline CLI | Roo | OpenHands SDK | cerrado |
|---|---|---|---|---|---|---|---|---|
| Rol/agente | A parcial | A/B | A | A | A | A | A | B/E |
| Tools dinámicas | A | A/B | A | A | A | A | A | ? |
| Reglas proyecto | AGENTS.md A | GEMINI.md B | config | QWEN.md | workspace | rules | config | ? |
| Modos de plan | verificar | docs/código | B | B | B | A/B | configurable | ? |
| Delegación | ? | remoto documentado | A/B | A/B | ? | ? | ? | ? |
| Compactación | ? | A | ? | ? | B CLI | ? | ? | ? |
| Seguridad/permisos | A | B | B | ? | B | A/B | configurable | B |

La comparación textual normaliza saltos de línea sin perder bloques; cuenta caracteres, tokens con tokenizer declarado, secciones, imperativos, reglas de tools y restricciones. Para prompt dinámico se guarda prompt base y recibo de bloques inyectados. Longitud/similitud no miden calidad. No reconstruir texto ausente como “oficial”.
