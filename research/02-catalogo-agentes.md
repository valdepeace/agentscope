---
id: investigacion-agentes-catalogo
title: Catálogo de agentes
---

# 02. Catálogo de agentes

Consulta: 25-10-2026. Grados de prompts: A=código oficial, B=documentación, C=extracción comunitaria verificable, D=inferencia, E=no disponible. Las URLs de ramas activas necesitan SHA antes de publicación.

| Agente | Atribución/estado | Evidencia y hallazgo verificable | Prompt |
|---|---|---|---|
| Claude Code | Anthropic, propietario | [Docs](https://docs.anthropic.com/en/docs/claude-code/overview); repo incluye [prompt para crear agentes](https://github.com/anthropics/claude-code/blob/main/plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md), que no es el prompt principal de ejecución | E (principal); A parcial |
| Codex CLI | OpenAI, open source | [Repo](https://github.com/openai/codex); `codex-rs/models-manager/prompt.md`; `codex-rs/core/src/agents_md.rs` recoge instrucciones AGENTS.md jerárquicas | A parcial; no único prompt |
| OpenCode | SST, open source | [Docs de agentes](https://docs.opencode.ai/docs/agents/); [constructor system.ts](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/session/system.ts) contempla componentes dinámicos para skills/MCP y prompts por proveedor/modelo | A/B |
| Crush | Charmbracelet, open source | [Repo](https://github.com/charmbracelet/crush); [mapa](https://github.com/charmbracelet/crush/blob/main/AGENTS.md), `internal/agent/agent.go`, prompts/plantillas Go | A, falta anclar comportamiento al SHA |
| Mastra Code | Mastra | [mastra-code-ui](https://github.com/mastra-ai/mastra-code-ui) parece producto desktop; separar de framework Mastra | A/B limitado; precisar alcance |
| Gemini CLI | Google, open source | [prompt docs](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/system-prompt.md); [prompts.ts](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/core/prompts.ts) | A/B |
| Aider | Aider, open source | [architect_prompts.py](https://github.com/Aider-AI/aider/blob/main/aider/coders/architect_prompts.py), [models.py](https://github.com/Aider-AI/aider/blob/main/aider/models.py): architect/editor y formato según modelo | A |
| Cline | Cline, open source | [CLI prompt](https://github.com/cline/cline/blob/main/apps/cli/src/runtime/prompt.ts); compone workspace/mode/rules; CLI docs describen plan/compaction/retry/`--system` | A, ámbito CLI |
| Roo Code | Roo Code, repo archivado en 2026 | [system.ts](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/prompts/system.ts); dinámica por modo, tools, skills, reglas y entorno | A/B, snapshot archivado |
| OpenHands | All-Hands, open source | [SDK agent](https://github.com/OpenHands/software-agent-sdk/blob/main/openhands-sdk/openhands/sdk/agent/agent.py), [system event](https://github.com/OpenHands/software-agent-sdk/blob/main/openhands-sdk/openhands/sdk/event/llm_convertible/system.py): prompt estático separado de herramientas/contexto | A, SDK; no todo el producto |
| SWE-agent | Princeton y comunidad | [agents.py](https://github.com/SWE-agent/SWE-agent/blob/main/sweagent/agent/agents.py): plantillas system/instance/next-step/observación | A |
| Continue | Continue, open source | [defaultSystemMessages.ts](https://github.com/continuedev/continue/blob/main/core/llm/defaultSystemMessages.ts): chat/agent/plan; plan restringido a lectura | A |
| Goose | Block, open source | [Repo](https://github.com/block/goose) | pendiente ruta/commit |
| Cursor | Anysphere, propietario | [Docs tools](https://docs.cursor.com/en/agent/tools): búsqueda, edición, terminal, MCP/guardrails | B; prompt E |
| Windsurf | Cognition, propietario | [Docs](https://docs.windsurf.com/) | prompt/runtime no verificados |
| Devin | Cognition, propietario | [Docs](https://docs.devin.ai/) | prompt/runtime no verificados |
| Zed Agent | Zed Industries; cliente abierto | [agent.rs](https://github.com/zed-industries/zed/blob/main/crates/agent/src/agent.rs); ACP integra agentes externos sin hacer suyo su prompt | A |
| Qwen Code | Alibaba/Qwen, open source | [Repo](https://github.com/QwenLM/qwen-code); prompt en `packages/core/src/core/prompts.ts`; `QWEN.md`, overrides y subagentes configurables | A |
| Kimi Code CLI | Moonshot AI, oficial, MIT según README | [Repo](https://github.com/MoonshotAI/kimi-code); coder/explore/plan, aislados según README; permisos/herramientas en docs | A/B; prompt exacto pendiente |
| MiniMax Code / MCode | MiniMax, repo oficial | [Repo](https://github.com/MiniMax-AI/minimax-code); CLI/TUI/headless/ACP; funciones anunciadas requieren auditoría de ruta | A/B |
| Trae Agent | ByteDance, herramienta de investigación | [Repo](https://github.com/bytedance/trae-agent); no confundir con el IDE Trae; config YAML, modelos/herramientas/trazas | A/B |
| DeepSeek Harness | DeepSeek lo anuncia como harness preview | [Página](https://www.deepseek.com/en/harness/); repositorio/ruta aún sin verificar | E parcial |
| ZCode | Z.ai/Zhipu, repo oficial | [Repo](https://github.com/zai-org/ZCode), CLI en `apps/zcode-cli`, licencia Apache-2.0 declarada | A |
| Baidu Comate | Baidu, producto oficial | [Sitio](https://comate.baidu.com/), [Cloud](https://cloud.baidu.com/product/comate-public.html): capacidades anunciadas; código/prompt no localizado | B; runtime E |
| Doubao Seed | ByteDance, familia/API | [Seed cookbook](https://github.com/ByteDance-Seed/Seed2.0) trae ejemplos agentic; no se confirmó CLI oficial de programación | Modelo/API ≠ agente |
| Tencent Hunyuan / CodeBuddy | Sin harness Hunyuan verificado en esta pasada | No atribuir CodeBuddy a Hunyuan sin prueba directa | E |
| Xiaomi MiMo Code | XiaomiMiMo, repo oficial | [MiMo-Code](https://github.com/XiaomiMiMo/MiMo-Code); separar de `mimoagent` y herramientas comunitarias como Mio | A/B |

Se consideran relevantes además los frameworks de harness OpenHands, SWE-agent y Trae Agent. Las comunidades de prompts [Piebald](https://github.com/Piebald-AI/claude-code-system-prompts), [x1xhlol](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools) y [tallesborges](https://github.com/tallesborges/agentic-system-prompts) son C, nunca fuente oficial del proveedor. No reproducir prompts propietarios sin licencia y procedencia.
