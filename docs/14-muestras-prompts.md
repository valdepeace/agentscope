---
id: muestras-prompts-agentes
title: Muestras públicas de prompts
---

# Muestras públicas de prompts

Consulta: 25-10-2026. Son **extractos breves** para ver estilo e instrucciones; no son los prompts completos ni equivalen necesariamente al prompt efectivo de una sesión. Los fragmentos reproducidos se mantienen cortos y se enlaza la fuente. “No localizado” significa que no encontré una muestra pública primaria del prompt principal en esta pasada; no implica que el agente no tenga uno.

## Agentes occidentales

| Agente | Muestra verificable | Procedencia / precisión |
|---|---|---|
| Claude Code | “You are an elite AI agent architect specializing in crafting high-performance agent configurations.” | [Prompt oficial de generación de agentes](https://github.com/anthropics/claude-code/blob/main/plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md), A. **No es** el prompt principal de Claude Code. |
| Codex CLI | “You are a coding agent running in the Codex CLI, a terminal-based coding assistant.” | [prompt.md](https://github.com/openai/codex/blob/main/codex-rs/models-manager/prompt.md), A. Archivo oficial, pero no prueba que sea el único prompt de todos los modos. |
| OpenCode | “You are opencode, an interactive CLI tool that helps users with software engineering tasks.” | [prompt/default.txt](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/session/prompt/default.txt), A. La plantilla cambia según modelo y configuración. |
| Crush | “You are an agent for Crush. Given the user's prompt, you should use the tools available to you…” | [task.md.tpl](https://github.com/charmbracelet/crush/blob/main/internal/agent/templates/task.md.tpl), A. Plantilla de tarea/subagente, no el prompt de coder completo. |
| Mastra Code | No localizada muestra primaria del prompt. | Repo/producto identificado; ruta exacta de prompt pendiente. |
| Gemini CLI | No reproduzco el prompt base: la documentación permite exportarlo desde la instalación concreta. | [System prompt docs](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/system-prompt.md), B. El prompt se compone dinámicamente; la exportación local es la muestra adecuada para una versión fijada. |
| Aider | “Act as an expert architect engineer and provide direction to your editor engineer.” | [architect_prompts.py](https://github.com/Aider-AI/aider/blob/main/aider/coders/architect_prompts.py), A. Es el prompt del modo Architect, no del modo editor. |
| Cline | No localizada cita directa del cuerpo del prompt en el builder consultado. | [resolveSystemPrompt](https://github.com/cline/cline/blob/main/apps/cli/src/runtime/prompt.ts) llama a `buildClineSystemPrompt`; la CLI admite override. |
| Roo Code | No localizada muestra directa del prompt principal en esta pasada. | [system.ts](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/prompts/system.ts), A; repo archivado. |
| OpenHands | “You are OpenHands agent, a helpful AI assistant that can interact with a computer to solve tasks.” | [agent/base.py](https://github.com/OpenHands/software-agent-sdk/blob/main/openhands-sdk/openhands/sdk/agent/base.py), A. Identidad base del SDK; el prompt final añade secciones dinámicas. |
| SWE-agent | No hay un texto fijo único: la configuración recibe `system_template` y `instance_template`. | [agents.py](https://github.com/SWE-agent/SWE-agent/blob/main/sweagent/agent/agents.py), A. Muestra de la arquitectura de plantillas, no prompt ya renderizado. |
| Continue | “You are in agent mode. If you need to use multiple tools, you can call multiple read-only tools simultaneously.” | [defaultSystemMessages.ts](https://github.com/continuedev/continue/blob/main/core/llm/defaultSystemMessages.ts), A. Fragmento del mensaje de agente; chat y plan tienen variantes. |
| Goose | No localizada muestra directa del prompt en esta pasada. | [Repositorio](https://github.com/block/goose); falta elegir/fijar la plantilla activa. |
| Cursor | No publicado/verificado. | La documentación de tools no expone el prompt interno. |
| Windsurf | No publicado/verificado. | No atribuir capturas comunitarias como prompt oficial. |
| Devin | No publicado/verificado. | Documentación de producto, harness interno cerrado. |
| Zed Agent | No localizada muestra directa del prompt principal. | El agente nativo y ACP no son lo mismo; el prompt externo depende del servidor conectado. |

## Agentes y herramientas del ecosistema chino

| Agente | Muestra verificable | Procedencia / precisión |
|---|---|---|
| Qwen Code | “You are Qwen Code, ${role} developed by Alibaba Group, ${focus}.” | [prompts.ts](https://github.com/QwenLM/qwen-code/blob/main/packages/core/src/core/prompts.ts), A. Plantilla con variables, el rol y foco se resuelven dinámicamente. |
| Kimi Code CLI | No se muestra aquí un fragmento del prompt built-in; sí se documenta que `SYSTEM.md` puede reemplazarlo y que los agentes Markdown aportan su propio body. | [Docs de agentes](https://github.com/MoonshotAI/kimi-code/blob/main/docs/en/customization/agents.md), B. No confundir formato de override con prompt predeterminado. |
| MiniMax Code / MCode | “You are a coding agent running in the MiniMax Code terminal, developed by MiniMax.” | [Captura comunitaria](https://gist.github.com/inikishev/03c1f70cc683c9617fff4cb40ee52421), C, extracción no oficial. No usar como transcripción autenticada hasta verificar versión/hash contra el runtime oficial. |
| Trae Agent | No localizada muestra directa del prompt en el README/config consultado. | [Repo oficial](https://github.com/bytedance/trae-agent) configura modelo, herramientas y pasos; obtener plantilla del commit para mostrar prompt efectivo. |
| DeepSeek Harness | No localizado prompt/repo de runtime verificable. | [Página del harness](https://www.deepseek.com/en/harness/), B/E. |
| ZCode (Z.ai / GLM) | No confirmado por fuente oficial. | Una issue comunitaria cita una posible línea de prompt de un binario, pero no equivale a archivo oficial ni a extracción reproducible; se omite. |
| Baidu Comate | No publicado/verificado. | Documentación comercial de capacidad no revela el prompt. |
| Doubao / Seed | No es un coding agent por sí mismo; por tanto no hay prompt de agente de Doubao que atribuir. | Trae Agent puede usar Doubao como modelo, pero su prompt pertenece al harness Trae. |
| Tencent Hunyuan / CodeBuddy | No localizado prompt oficial verificable. | No atribuir prompt de herramienta de terceros a Hunyuan. |
| Xiaomi MiMo Code | No localizada muestra del prompt built-in en esta pasada. | El repo es un harness de agente; no copiar el prompt de OpenCode por su parentesco/fork sin comparar el commit. |

## Cómo obtener el prompt efectivo completo

Para los proyectos que permiten exportarlo, guardar el artefacto de la versión instalada y anotar versión del CLI, modelo/API ID, modo, configuración, herramientas activas, archivos de instrucciones, hash SHA-256 y fecha. Un prompt estático del repo no incluye automáticamente los bloques añadidos en runtime. Para sistemas cerrados, conservar solo documentación, capturas con procedencia y hash cuando sea lícito; no reconstruir texto ausente como si fuera oficial.

## Nota de derechos y reproducibilidad

Los extractos se limitan a frases breves y se enlaza la fuente. El contenido completo permanece en el repositorio/origen; las plantillas mutables deben fijarse a un commit antes de comparar longitud, tokens o similitud.
