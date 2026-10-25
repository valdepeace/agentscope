# Oleada 1: cinco harnesses de programación

**Corte:** 2026-10-25. **Método:** inspección estática del código oficial en cinco clones shallow; sin ejecutar los agentes, sin inferencia de modelos y sin benchmarks. Los SHA hacen reproducible el snapshot, pero no equivalen a una versión publicada.

## Hallazgo resumido

En estos cinco casos, el texto visible del prompt es solo una parte del comportamiento. El harness decide qué prompt ensamblar, qué herramientas exponer, qué contexto entregar, cómo ejecutar llamadas, qué permisos aplicar y cómo devolver el resultado. Los snapshots muestran además que “subagente” no designa un protocolo común: Codex ofrece control y mensajes entre hilos; Gemini separa ejecución local y remota; OpenCode crea sesiones hijas con notificación; Crush usa una herramienta paralela con sesión; Qwen añade fork de conversación y colaboración A2A experimental. La existencia del mecanismo no demuestra que mejore resultados ni que dos productos interoperen.

## Muestras verificadas de prompts

Son extractos literales breves de los archivos enlazados, no transcripciones del prompt efectivo de una sesión. Las plantillas dinámicas pueden cambiar con el modo, herramientas, modelo, configuración, memoria o archivos del usuario.

| Agente | Extracto localizado | Qué evidencia es |
|---|---|---|
| Codex CLI | “You are a coding agent running in the Codex CLI, a terminal-based coding assistant.” | Apertura del prompt de sistema publicado en el repositorio. El runtime añade contexto y herramientas. |
| Gemini CLI | “The COMPLETE query to send the subagent. MUST be comprehensive and detailed.” | Descripción del parámetro `prompt` de la herramienta Agent; es instrucción de delegación, no el system prompt principal. |
| OpenCode | “You are an AI agent running in OpenCode, a coding agent harness.” | Apertura de la plantilla común `system.txt`; el marcador de guía de herramientas se sustituye dinámicamente. Hay plantillas adicionales por proveedor. |
| Crush | “You are Crush, a powerful AI Assistant that runs in the CLI.” | Apertura de `coder.md.tpl`, plantilla del rol coder; opciones de runtime rellenan variables y contexto. |
| Qwen Code | `You are Qwen Code, ${role} developed by Alibaba Group...` | Plantilla de identidad en código. `${role}` y el resto del prompt dependen de modo, modelo, estilo, tools y overrides. |

## Fichas de código

### Codex CLI

- **Snapshot:** `openai/codex`, `970111f6cb62cc02fe36a82ea7d71b37a2889c97` (`main`); implementación principalmente Rust en el árbol inspeccionado. El manifiesto local da `0.0.0`, por lo que no se atribuye un número de versión comercial.
- **Prompt/contexto:** [`prompt.md`](https://github.com/openai/codex/blob/970111f6cb62cc02fe36a82ea7d71b37a2889c97/codex-rs/models-manager/prompt.md) documenta rol, estilo, AGENTS.md y uso de herramientas. [`agents_md.rs`](https://github.com/openai/codex/blob/970111f6cb62cc02fe36a82ea7d71b37a2889c97/codex-rs/core/src/agents_md.rs) implementa la carga de instrucciones de proyecto y su alcance desde raíz detectada hasta el directorio actual; no es texto fijo del prompt.
- **Delegación/comunicación:** [`spawn.rs`](https://github.com/openai/codex/blob/970111f6cb62cc02fe36a82ea7d71b37a2889c97/codex-rs/core/src/agent/control/spawn.rs) prepara agentes/historial fork; [`control.rs`](https://github.com/openai/codex/blob/970111f6cb62cc02fe36a82ea7d71b37a2889c97/codex-rs/core/src/agent/control.rs) incluye `send_inter_agent_communication`, listado e interrupción. El código de fork filtra elementos y transforma instrucciones para el hijo; no se debe describir como un brief independiente sin revisar cada modalidad.
- **Contexto:** [`compact.rs`](https://github.com/openai/codex/blob/970111f6cb62cc02fe36a82ea7d71b37a2889c97/codex-rs/core/src/compact.rs) trata compactación; el historial heredado no implica heredar sesiones vivas de herramientas/procesos.
- **Protocolo:** evidencia de coordinación interna del runtime; en las rutas inspeccionadas no se verificó un endpoint A2A estándar.

### Gemini CLI

- **Snapshot:** `google-gemini/gemini-cli`, `cd1b45c2db0a5ea7b7fc3b67c1ba5879601d6005` (`main`); `@google/gemini-cli-core` declara `0.67.0-nightly.20261020.gb041de017`; TypeScript.
- **Prompt/contexto:** [`promptProvider.ts`](https://github.com/google-gemini/gemini-cli/blob/cd1b45c2db0a5ea7b7fc3b67c1ba5879601d6005/packages/core/src/prompts/promptProvider.ts), llamado por [`core/prompts.ts`](https://github.com/google-gemini/gemini-cli/blob/cd1b45c2db0a5ea7b7fc3b67c1ba5879601d6005/packages/core/src/core/prompts.ts), compone el prompt a partir de modo interactivo, aprobación/plan, modelo, herramientas registradas, skills, memoria y posible override `GEMINI_SYSTEM_MD`. Hay un prompt de compresión separado. El resultado efectivo no es un único literal estable.
- **Delegación:** [`agent-tool.ts`](https://github.com/google-gemini/gemini-cli/blob/cd1b45c2db0a5ea7b7fc3b67c1ba5879601d6005/packages/core/src/agents/agent-tool.ts) solicita un prompt completo y detallado para el subagente y resuelve agentes registrados; hay invocaciones local y remota en clases separadas.
- **A2A:** [`remote-session-invocation.ts`](https://github.com/google-gemini/gemini-cli/blob/cd1b45c2db0a5ea7b7fc3b67c1ba5879601d6005/packages/core/src/agents/remote-session-invocation.ts) delega al protocolo remoto A2A y conserva estado de sesión remota `contextId`/`taskId`. Esto verifica el lado cliente observado; la presencia de un paquete servidor en el repo no basta para afirmar conformidad de servidor.

### OpenCode

- **Snapshot:** `anomalyco/opencode`, `450e7b92412c780a7b5a75226bb8d49aaca24e22` (`dev`); `@opencode/core` declara `2.1.4`; TypeScript. La estructura vigente aquí está en `packages/core`, no en la ruta histórica `packages/opencode`.
- **Prompt/contexto:** [`system.txt`](https://github.com/anomalyco/opencode/blob/450e7b92412c780a7b5a75226bb8d49aaca24e22/packages/core/src/session/runner/prompt/system.txt) aporta instrucciones comunes y `${OPENCODE_TOOL_GUIDANCE}`. [`session/prompt.ts`](https://github.com/anomalyco/opencode/blob/450e7b92412c780a7b5a75226bb8d49aaca24e22/packages/core/src/session/prompt.ts) deja pasar la entrada por hooks antes de preparar attachments/prompt. También existen variantes por proveedor bajo `plugin/system-prompt/`; por tanto no hay una única plantilla efectiva universal.
- **Delegación/comunicación:** [`subagent.ts`](https://github.com/anomalyco/opencode/blob/450e7b92412c780a7b5a75226bb8d49aaca24e22/packages/core/src/tool/plugin/subagent.ts) crea sesión hija; permite reanudar con `sessionID` y modo background. El padre recibe notificación cuando acaba; [`subagent-completion.ts`](https://github.com/anomalyco/opencode/blob/450e7b92412c780a7b5a75226bb8d49aaca24e22/packages/core/src/session/subagent-completion.ts) empaqueta texto/estado como resultado. La descripción instruye no sondear ni duplicar el trabajo de un subagente en background.
- **A2A:** en las rutas estudiadas no se verificó implementación A2A. MCP/tools o sesiones hijas no son evidencia de A2A.

### Crush

- **Snapshot:** `charmbracelet/crush`, `9ab41e330eb47057a328ba2aeba3c702522a43eb` (`main`); módulo Go, `go.mod` declara Go `1.27.0` y no fija versión de aplicación.
- **Prompt/contexto:** [`prompts.go`](https://github.com/charmbracelet/crush/blob/9ab41e330eb47057a328ba2aeba3c702522a43eb/internal/agent/prompts.go) carga plantillas embebidas para coder, task, plan e initialize. [`prompt.go`](https://github.com/charmbracelet/crush/blob/9ab41e330eb47057a328ba2aeba3c702522a43eb/internal/agent/prompt/prompt.go) construye los datos y renderiza la plantilla; [`coder.md.tpl`](https://github.com/charmbracelet/crush/blob/9ab41e330eb47057a328ba2aeba3c702522a43eb/internal/agent/templates/coder.md.tpl), `task.md.tpl`, `summary.md` y otras plantillas separan rol, tarea y resumen.
- **Delegación:** [`agent_tool.go`](https://github.com/charmbracelet/crush/blob/9ab41e330eb47057a328ba2aeba3c702522a43eb/internal/agent/agent_tool.go) configura el rol `AgentTask` y `fantasy.NewParallelAgentTool`; exige `sessionID` y `agentMessageID` del contexto, y el coordinador ejecuta `runSubAgent`. Es paralelismo local mediante el framework, no protocolo entre productos.
- **Contexto/recuperación:** [`agent.go`](https://github.com/charmbracelet/crush/blob/9ab41e330eb47057a328ba2aeba3c702522a43eb/internal/agent/agent.go) contiene estado de sesión y ruta de autosumarización; se detectaron umbrales y prompt de resumen. No se midieron disparadores ni pérdida de información en ejecución.
- **A2A:** no se localizó implementación A2A en las rutas inspeccionadas; no es una auditoría completa del repositorio.

### Qwen Code

- **Snapshot:** `QwenLM/qwen-code`, `41ce46af24c0bda9441216191a4b119fef846fab` (`main`); `@qwen-code/qwen-code-core` declara `0.26.1`; TypeScript.
- **Prompt/contexto:** [`prompts.ts`](https://github.com/QwenLM/qwen-code/blob/41ce46af24c0bda9441216191a4b119fef846fab/packages/core/src/core/prompts.ts) monta un prompt dependiente de modo (interactive/headless/ACP), modelo, output style, herramientas y otros parámetros. `QWEN_SYSTEM_IDENTITY_MD` sustituye la identidad; `QWEN_SYSTEM_MD` puede reemplazar la base completa. Por eso cualquier muestra estática es parcial.
- **Subagentes:** [`agent.ts`](https://github.com/QwenLM/qwen-code/blob/41ce46af24c0bda9441216191a4b119fef846fab/packages/core/src/tools/agent/agent.ts), [`fork-subagent.ts`](https://github.com/QwenLM/qwen-code/blob/41ce46af24c0bda9441216191a4b119fef846fab/packages/core/src/tools/agent/fork-subagent.ts) y `subagents/` implementan subagentes de tarea y forks explícitos. El fork declara heredar conversación del padre; las opciones de ejecución incluyen perfiles/selección de herramientas y otras políticas. Hay límites que requieren auditar cada modo antes de llamar “sandbox” a un worktree.
- **A2A externo:** [`a2a-server.ts`](https://github.com/QwenLM/qwen-code/blob/41ce46af24c0bda9441216191a4b119fef846fab/packages/core/src/agents/workspace-agents/a2a-server.ts) y [`a2a-contract.ts`](https://github.com/QwenLM/qwen-code/blob/41ce46af24c0bda9441216191a4b119fef846fab/packages/core/src/agents/workspace-agents/a2a-contract.ts) implementan el contrato del servidor observado: A2A `1.0`, binding JSON-RPC, Agent Card, tareas ligadas a sesiones Qwen. El endpoint de colaboración está protegido por la opción experimental `agentCollaboration`, cuyo default en configuración es `false`. Quien llama externamente no puede resolver una aprobación local de herramienta; el run puede quedar esperando intervención del propietario del workspace.
- **Límite:** el código demuestra implementación y opt-in en este commit, no interoperabilidad efectiva con pares independientes ni activación por defecto.

## Comparación de la delegación

| CLI | Contexto del hijo | Ejecución y retorno observados | Comunicación entre agentes | A2A comprobado |
|---|---|---|---|---|
| Codex | Fork con filtrado/preparación de historial e instrucciones del hijo | Hilos gestionados; actividad, interrupción y mensajes | Sí, interna del runtime | No localizado en este snapshot |
| Gemini CLI | Herramienta pide al padre un brief completo; ejecución local o remota | Invocación por Agent Tool; remoto conserva estado de sesión A2A | Herramienta unificada, backends distintos | Cliente remoto verificado; servidor no evaluado aquí |
| OpenCode | Prompt de tarea y sesión hija; puede continuar por ID | Background y notificación de completion al padre | Sesión/mensajes internos | No localizado |
| Crush | Prompt de tarea con working dir y contexto mínimo requerido por tool | `fantasy` parallel agent y coordinador con IDs de sesión/mensaje | Retorno como resultado de herramienta | No localizado |
| Qwen Code | Subagente normal o fork explícito de conversación | Herramientas de agente; colaboración entre sesiones opt-in | Coordinación local y endpoint externo son capas diferentes | Servidor externo A2A 1.0 implementado, opt-in |

**Lectura:** cada producto define por su cuenta qué significa heredar contexto, qué herramientas ofrecer al hijo, cómo se informa el progreso y qué estado retorna. Los casos de cliente/servidor A2A tampoco implican que la delegación local use A2A. Esta muestra respalda la hipótesis de fragmentación por defecto, pero cinco repos no prueban que “todos” sean incompatibles.

## Qué queda pendiente

- Recorrer los tool loops completos, permisos/sandbox, reintentos y reanudación de cada runtime; esta pasada seleccionó rutas de prompt, delegación y contexto.
- Ejecutar pruebas de interoperabilidad A2A (descubrimiento, auth, tarea, seguimiento, cancelación, approvals y artifacts) en pares independientes.
- Congelar release tags además de commits, calcular hashes de los archivos de prompt y comparar versiones para estimar cambios.
- Capturar prompts efectivos sin datos privados mediante modo de depuración de cada producto; los extractos anteriores no representan una serialización runtime.
- Hacer el benchmark controlado solo después de fijar modelos, tareas, coste máximo y condiciones, con autorización para cualquier inferencia facturable.

## Fuentes primarias fijadas

La lista con commit, ruta y URL está en [`oleada-01-fuentes.csv`](oleada-01-fuentes.csv). Los clones de consulta se mantienen fuera del repositorio para no añadir cinco árboles de código al sitio.
