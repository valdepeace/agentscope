---
id: investigacion-agentes-harness
title: Arquitecturas de harness
---

# 05. Arquitecturas y ejecución

Patrón auditable: construir mensajes/tools → llamada al modelo → parsear/validar → aprobación/política → dispatch → resultado como mensaje → continuar/parar. Debe probarse en código; README acredita descripción, no ruta de ejecución.

| Agente | Rutas/artefactos de evidencia primaria |
|---|---|
| Codex | repo Rust; `models-manager/prompt.md`, `core/src/agents_md.rs` |
| Gemini CLI | `packages/core/src/core/prompts.ts`, `PromptProvider`; docs de contexto, override y compactación |
| OpenCode | `packages/opencode/src/session/system.ts`; primary/subagent y permisos |
| Crush | `internal/agent/agent.go`, `internal/agent/prompts.go`, `templates/*.md.tpl`, mapa `AGENTS.md` |
| Aider | `aider/coders/architect_prompts.py`, `aider/models.py` |
| Cline | `apps/cli/src/runtime/prompt.ts`; superficie CLI |
| Roo | `src/core/prompts/system.ts` |
| OpenHands | SDK `agent.py`, `llm_convertible/system.py` |
| SWE-agent | `sweagent/agent/agents.py`, `TemplateConfig` |
| Continue | `core/llm/defaultSystemMessages.ts` |
| Qwen | `packages/core/src/core/prompts.ts`, docs de subagents/config |
| Kimi | repo oficial TS; rutas exactas del dispatcher/prompt pendientes |
| MiniMax Code, Trae Agent, ZCode, MiMo Code | repos oficiales identificados; rutas puntuales pendientes |
| Claude/Cursor/Windsurf/Devin/Comate | docs de proveedor; runtime interno cerrado/no localizado |

Auditoría completa debe fijar commit y recorrer entrypoint, mensaje builder, tool registry, dispatcher, permission/sandbox, errores/retries, límites, sesión, compactación, hooks/MCP, telemetría y pruebas.

**Seguridad:** la instrucción “pide permiso” no es garantía. La aplicación debe estar en dispatcher/sandbox/ACL; registrar aprobaciones, red, y permisos. No inferir comportamiento del producto hospedado desde SDK parcial.
