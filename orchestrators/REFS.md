# Referencias consultadas

Registro de todo lo que se ha consultado para el estudio de "Super-agent". Fecha de las consultas: **2026-11-21**.

## Cómo leer los niveles de confianza

Cada fuente lleva una etiqueta según lo que realmente se hizo con ella:

| Etiqueta | Significa |
|---|---|
| **CLONADO** | Repo descargado en `references/` y leído (con muestreo en los grandes). Es la fuente más fiable. |
| **LEÍDO** | Página abierta y leída con WebFetch (el contenido pasó por un modelo pequeño que lo resumió según una pregunta mía). |
| **API** | Dato sacado con `gh api` de GitHub. Fiable y fechado. |
| **SNIPPET** | Solo vi el título y un resumen en un resultado de búsqueda. **No abrí la página.** Cualquier cosa que diga es de oídas. |

Regla del estudio: lo que dice un README o un blog es promesa hasta verlo en código.

---

## 1. Repos clonados (`references/`)

Clones superficiales (`--depth 1`). Notas de cada uno en `study/<carpeta>.md`.

| Carpeta | Repo | Commit | Ficheros | Estado de la nota |
|---|---|---|---:|---|
| `dsh-superagent-prototype` | https://github.com/deepseek-ai/dsh-superagent-prototype | `8c32a9d` | 216 | Hecha |
| `cc-super-agent` | https://github.com/yingpengma/cc-super-agent | `1538e25` | 35 | Hecha |
| `deepseek-super-agent` | https://github.com/LofiSu/dsh-super-agent (antes `LofiSu/deepseek-super-agent`) | `ca9823e` | 437 | Hecha (muestreo) |
| `super-super-harness` | https://github.com/ketchupz1999/super-super-harness | `df78e64` | 118 | Hecha |
| `deer-flow` | https://github.com/bytedance/deer-flow | `900d1b6` | 4.770 | Hecha (muestreo) |
| `agent-orchestrator` | https://github.com/OrchestratorInc/agent-orchestrator | `080a1a2` | 7.950 | Hecha, **cobertura ~3-4 %** |
| `agent-swarm` | https://github.com/desplega-ai/agent-swarm | `48fa684` | 7.498 | Hecha, cobertura ~15 % (watchdog desde el runbook, no desde el código) |
| `cli-agent-orchestrator` | https://github.com/awslabs/cli-agent-orchestrator | `ba53d49` | 3.004 | Hecha, cobertura ~12 % |
| `gastown` | https://github.com/gastownhall/gastown | `94cc822` | 1.565 | Hecha, cobertura ~15-20 % |

## 2. Fuentes primarias abiertas (LEÍDO)

| URL | Para qué se usó |
|---|---|
| https://github.com/deepseek-ai/dsh-superagent-prototype | README del prototipo oficial: propósito, estado "research prototype", estructura de docs, licencia MIT |
| https://agentspulse.github.io/tutorials/deepseek-super-agent-architecture/ | Resumen de la arquitectura Router/Worker, admission control, benchmark (p50/p95 12,4/15,7 min), límites. **Es un tutorial de terceros, no el artículo original de DeepSeek.** |
| https://github.com/yingpengma/cc-super-agent | README: cómo implementa Router/Worker sobre Claude Code, stack, limitaciones |

> **Pendiente:** el artículo original de DeepSeek ("Exploring Super-agent Design") **no se ha localizado ni leído**. El README del prototipo lo cita pero no está dentro del repo. Todo lo que sabemos del artículo viene de resúmenes de terceros.

## 3. Datos de GitHub (API)

Consultados con `gh api repos/<owner>/<repo>` y `gh api search/repositories`.

### 3.1 Los cinco repos del estudio inicial

| Repo | ⭐ | Forks | Creado | Último push |
|---|---:|---:|---|---|
| bytedance/deer-flow | 85.571 | 12.070 | 2025-05-07 | 2026-11-21 |
| deepseek-ai/dsh-superagent-prototype | 24.127 | 2.997 | 2026-10-31 | 2026-11-03 |
| LofiSu/dsh-super-agent | 146 | 18 | 2026-10-31 | 2026-11-21 |
| yingpengma/cc-super-agent | 7 | 1 | 2026-11-01 | 2026-11-02 |
| ketchupz1999/super-super-harness | 1 | 0 | 2026-11-03 | 2026-11-04 |

Datos completos (contribuyentes, commits, releases): `TABLAS.md` y el dashboard publicado.

### 3.2 Ampliación: orquestadores de agentes de código

| Repo | ⭐ | Lenguaje | Licencia | Último push | Notas |
|---|---:|---|---|---|---|
| ruvnet/ruflo | 82.196 | TypeScript | MIT | 2026-11-21 | Antes Claude Flow. No clonado |
| stablyai/orca | 131.546 | TypeScript | MIT | 2026-11-21 | Creado 2026-03; cifra de estrellas enorme, **orgánica o no, sin verificar**. No clonado |
| OpenHands/OpenHands | 93.948 | TypeScript | MIT | 2026-11-21 | Plataforma general. No clonado |
| MetaGPT (FoundationAgents) | 71.374 | Python | MIT | 2026-01-21 | **Parado desde enero**. No clonado |
| Yeachan-Heo/oh-my-claudecode | 40.988 | TypeScript | MIT | 2026-11-21 | No clonado |
| BloopAI/vibe-kanban | 28.767 | Rust | Apache-2.0 | 2026-11-15 | Los blogs hablan de cierre de Bloop; GitHub muestra actividad. No clonado |
| gastownhall/gastown | 18.859 | Go | MIT | 2026-11-14 | **Clonado** |
| Paradev (superset-sh/superset) | 16.444 | TypeScript | MIT | 2026-11-21 | Licencia contradictoria según blogs. No clonado |
| OrchestratorInc/agent-orchestrator | 14.985 | TypeScript según la API; **Go** según el código | Apache-2.0 | 2026-11-21 | **Clonado**. Es una reescritura en Go con frontend TypeScript. `ComposioHQ/agent-orchestrator` redirige aquí |
| smtg-ai/claude-squad | 8.846 | Go | AGPL-3.0 | 2026-11-13 | No clonado |
| generalaction/emdash | 6.289 | TypeScript | Apache-2.0 | 2026-11-20 | Multiplataforma. No clonado |
| awslabs/cli-agent-orchestrator | 1.615 | Python | Apache-2.0 | 2026-11-21 | **Clonado** |
| desplega-ai/agent-swarm | 1.094 | TypeScript | MIT | 2026-11-21 | **Clonado** |
| johannesjo/parallel-code | 1.101 | TypeScript | MIT | 2026-11-20 | No clonado |
| dlorenc/multiclaude | 565 | Go | — | 2026-01-28 | Sin actividad desde enero |
| kingbootoshi/codex-orchestrator | 361 | TypeScript | MIT | 2026-05-28 | No clonado |
| nwiizo/ccswarm | 157 | Rust | MIT | 2026-11-07 | No clonado |
| hyw007726/multi-agent-orchestrator-skill | 6 | JavaScript | MIT | 2026-05-24 | No clonado |
| RobBrautigam/ai-orchestrator | 1 | Python | MIT | 2026-11-20 | No clonado |
| andyrewlee/awesome-agent-orchestrators | 2.467 | — | CC0-1.0 | 2026-11-20 | Lista curada, no código |

### 3.3 Routers de modelos (otra categoría)

| Repo | ⭐ | Licencia | Último push |
|---|---:|---|---|
| musistudio/claude-code-router | 38.283 | MIT | 2026-11-21 |

(`mehdihadeli/agent-router` y `FlorianBruniaux/agent-router` solo se vieron como SNIPPET.)

## 4. Resultados de búsqueda no abiertos (SNIPPET)

Solo se vio título y resumen. Se listan por si alguna merece lectura posterior. **Ninguna se ha usado como evidencia en el estudio** salvo para orientarme.

### 4.1 Linaje DeepSeek Super-agent

- https://github.com/topics/super-agent (lista de ~34 repos)
- https://github.com/LofiSu/deepseek-super-agent
- https://github.com/columbia11/deepseek-super-agent (copia o fork del anterior)
- https://github.com/ketchupz1999/super-super-harness
- https://dev.to/neuraitool/deepseeks-open-source-super-agent-codes-without-being-asked-4m5
- https://startupfortune.com/deepseek-open-sources-the-architecture-behind-its-super-agent/
- https://shattered.io/deepseek-open-sources-super-agent-oct-31-2026/
- https://x.com/AGTPinsights/status/2116631973519372495
- https://www.mindstudio.ai/blog/deepseek-harness-plugin-architecture
- https://animaapp.com/blog/agentic/deepseek-super-agent/
- https://en.spaziocrypto.com/ai/deepseek-super-agent-open-source-github-prototype/
- https://ai-tldr.dev/tools/dsh-superagent/
- https://pypi.org/project/superagentx (proyecto distinto: SuperAgentX)

### 4.2 DeerFlow

- https://medium.com/@2315610426/deerflow-by-bytedance-the-complete-guide-to-the-open-source-super-agent-that-researches-codes-99192db52a5f
- https://tosea.ai/blog/deerflow-bytedance-open-source-research-agent-guide
- https://andrew.ooo/posts/deer-flow-bytedance-superagent-harness-review/
- https://andrew.ooo/posts/deerflow-2-bytedance-superagent-harness-review/
- https://github.com/bytedance/deer-flow/blob/main/README_zh.md (y variantes `_ru`, `_fr`)

### 4.3 Otros "super agent" y marcos generales

- https://stackshare.io/stackups/agentops-vs-super-agi
- https://superagenticai.github.io/superoptix-ai/guides/superspec/
- https://github.com/aloth/awesome-ai-agents
- https://github.com/caramaschiHG/awesome-ai-agents-2026
- https://github.com/ARUNAGIRINATHAN-K/awesome-ai-agents-2026

### 4.4 Orquestadores de agentes de código (ampliación)

Listas y directorios:
- https://github.com/andyrewlee/awesome-agent-orchestrators
- https://github.com/Agent-Analytics/awesome-multi-agent-orchestrators
- https://openorchestrators.org/
- https://agent-orchestrators.com/

Comparativas y blogs (**varios son de empresas con producto competidor**, tratar con cautela):
- https://www.cloudroom.dev/blog/open-source-agent-orchestrators
- https://nimbalyst.com/blog/best-conductor-alternatives-2026/ (Nimbalyst vende un producto competidor)
- https://nimbalyst.com/blog/best-multi-agent-coding-tools-2026/
- https://nimbalyst.com/blog/best-ai-agent-orchestration-platforms-2026/
- https://rustman.org/wiki/conductor-parallel-agents/
- https://www.mulmoterminal.com/guide/en/alternatives.html
- https://www.codeagentswarm.com/en/guides/best-tools-to-run-multiple-ai-coding-agents
- https://aq.dev/alternatives/vibe-kanban/

Productos y repos:
- https://orchestrator.inc/
- https://superset.sh/
- https://github.com/hyw007726/multi-agent-orchestrator-skill
- https://github.com/RobBrautigam/ai-orchestrator
- https://github.com/johannesjo/parallel-code

Papers (aparecieron en resultados, **no leídos ni relevantes confirmados**):
- https://arxiv.org/pdf/2602.21351
- https://arxiv.org/pdf/2610.22195

### 4.5 Routers de modelos

- https://github.com/topics/model-router
- https://github.com/topics/claude-code-router
- https://github.com/musistudio/claude-code-router
- https://github.com/hishamkaram/claude-code-router
- https://github.com/mehdihadeli/agent-router
- https://github.com/FlorianBruniaux/agent-router
- https://www.morphllm.com/claude-code-router (fuente de la cifra "ahorro de la mitad de tokens", **no es un benchmark**)
- https://www.datacamp.com/tutorial/claude-code-router
- https://munderdiffl.in/blog/claude-code-router/
- https://opper.ai/blog/claude-code-router

## 5. Afirmaciones de fuentes secundarias SIN verificar

Anotadas para que nadie las cite como hechos:

| Afirmación | De dónde | Estado |
|---|---|---|
| "Agent Swarm fue #1 en GitHub Trending con v2.0 en febrero de 2026" | Resumen de búsqueda | No verificada |
| Existe un proyecto "firstmate" (un jefe que lanza workers en worktrees) | Resumen de búsqueda | **No se encontró el repo** |
| "Vibe Kanban está en sunset formal; Bloop cerró en abril de 2026" | Blogs | Contradicho por GitHub (push 2026-11-15) |
| Licencia de Paradev: Elastic License 2.0 o MIT | Blogs, contradictorios | API dice MIT; sin leer el LICENSE |
| Licencia de DeerFlow: Apache 2.0 o MIT | Blogs, contradictorios | **Resuelto:** MIT, leído en `LICENSE:1` |
| "Claude Code, ~155k estrellas; Codex, ~135k" | Resumen de búsqueda | No verificada |
| Benchmark DeepSeek p95 15,7 frente a 29,5 min | Tutorial de terceros | Cifras sintéticas, 5 corridas; no reproducidas |
| Anthropic no admite modelos no-Claude vía gateway en Claude Code | Resumen de búsqueda | Atribuido a docs de Anthropic, **no leídas** |
| 24k estrellas del repo oficial de DeepSeek "con un solo commit" | API | Verificada |

## 6. Consultas de búsqueda realizadas

Búsqueda web (herramienta WebSearch):
1. `"super agent" open source GitHub repo multi-agent harness`
2. `DeerFlow ByteDance super agent harness GitHub`
3. `superagent github "super-agent" agent framework comparison`
4. `open source orchestrator multiple coding agents parallel Claude Code Codex git worktrees GitHub 2026`
5. `multi-agent coding orchestration framework GitHub trending 2026 supervisor workers`
6. `agent orchestrator "coding agents" kanban parallel sessions open source Conductor Vibe Kanban Claude Squad alternatives`
7. `LLM router coding agent model routing open source GitHub claude-code-router`

Lectura de páginas (WebFetch): las tres de la sección 2.

GitHub (`gh api`): metadatos de repos, contribuyentes, commits y releases; búsquedas por nombre para los candidatos de la ampliación.

## 7. Material generado en este estudio

| Fichero | Contenido |
|---|---|
| `STUDY.md` | Comparativa narrativa de los cinco repos iniciales |
| `TABLAS.md` | Tablas con métricas de GitHub y matrices de capacidades |
| `REFS.md` | Este fichero |
| `study/<repo>.md` | Notas por repo con evidencias `ruta:línea` (9 repos, 4 en curso) |

## 8. Herramientas instaladas

| Herramienta | Origen | Cómo se instaló | Dónde |
|---|---|---|---|
| **archify** (skill de agente: diagramas de arquitectura, flujo, secuencia, ciclo de vida, árbol, etc. como HTML autocontenido; acepta Mermaid como entrada) | https://github.com/tt-a1i/archify, commit `8509a3d`, 98.486 ⭐ (API, 2026-11-21) | Copia manual de la carpeta `archify/` del commit revisado. **No** se usó `npx skills add` (descarga y ejecuta otro paquete) y **no** es instalación global | `.claude/skills/archify/` dentro de este proyecto |

Revisión previa a la instalación (lectura, sin ejecutar nada de él): licencia Apache-2.0 con `LICENSE-MIT` adjunto (los blogs discrepaban); sin dependencias npm (`engines.node >= 18`; hay Node 24.8.0); los scripts usan `child_process` para lanzar sus propios comprobadores y abrir el navegador; **sin llamadas de red** en el código propio (la búsqueda de `fetch`, `curl` y `XMLHttpRequest` solo dio una librería minificada de terceros). Es una revisión por muestreo de patrones, no una auditoría completa de sus ~200 ficheros. Tras copiarlo solo se ejecutó `node bin/archify.mjs --help`.

Fuentes sobre archify vistas solo como SNIPPET: https://www.kucoin.com/news/flash/github-trending-tool-archify-automates-architecture-diagrams-with-ai, https://highlightsx.github.io/archify, https://www.coddykit.com/pages/blog-detail?id=513044, https://www.lejnel.com/blog/github-trending-tt-a1i-archify/, https://www.mrtechking.com/archify-github/, https://www.besthub.dev/articles/archify-ai-skill-that-reads-code-to-generate-interactive-architecture-diagrams-34ca272b17b7, https://dev.to/river_jiang_670cccce7d97d/archify-online-interactive-diagrams-in-your-browser-no-agent-needed-44nd. Forks mencionados: `All-The-Vibes/ATV-archify`, `GongYuanCaiJi/dsh-archify` (port para DeepSeek Harness), `Harrison-Yuan/archify-webui`.

## 9. Huecos de documentación

- Artículo original de DeepSeek: sin localizar (ver sección 2).
- `STUDY.md` (§10 y §11) y `TABLAS.md` cubren ya los nueve repos. El dashboard publicado (privado) se actualizó el 2026-11-21 a los nueve repos con datos generados desde `TABLAS.md`.
- Cobertura de lectura baja en los cuatro repos de la ampliación (3-20 %). Sus conclusiones sobre watchdogs, sandbox y rendimiento no están contrastadas con ejecución.
- Ningún test, benchmark ni comando de los repos se ha ejecutado.
- Citas `ruta:línea` de las notas: 807 comprobadas mecánicamente (fichero y rango) y 72 contrastadas con el código (ver `STUDY.md` §0). Se corrigieron 5 citas desplazadas 1-3 líneas: 4 salieron en la muestra y una quinta, vecina de la misma afirmación, en `deepseek-super-agent` (`agent-orchestrator` 1, `deepseek-super-agent` 3, `deer-flow` 1).

## Enlaces a los nueve repositorios

- [bytedance/deer-flow](https://github.com/bytedance/deer-flow)
- [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype)
- [gastownhall/gastown](https://github.com/gastownhall/gastown)
- [OrchestratorInc/agent-orchestrator](https://github.com/OrchestratorInc/agent-orchestrator)
- [awslabs/cli-agent-orchestrator](https://github.com/awslabs/cli-agent-orchestrator)
- [desplega-ai/agent-swarm](https://github.com/desplega-ai/agent-swarm)
- [LofiSu/dsh-super-agent](https://github.com/LofiSu/dsh-super-agent)
- [yingpengma/cc-super-agent](https://github.com/yingpengma/cc-super-agent)
- [ketchupz1999/super-super-harness](https://github.com/ketchupz1999/super-super-harness)

## Segunda pasada: arquitectura y mantenibilidad (2026-11-21)

- [`ARQUITECTURA.md`](ARQUITECTURA.md) y [`MANTENIBILIDAD.md`](MANTENIBILIDAD.md): comparativas de siete repos, separadas por eje. Notas por repo en `study/arquitectura/` y `study/mantenibilidad/`.
- Las correcciones a las notas anteriores (tabla de `ARQUITECTURA.md`) aún no se han propagado a `STUDY.md`, `TABLAS.md` ni al dashboard.
- [`PUNTUACION.md`](PUNTUACION.md): rúbrica y notas de arquitectura y mantenibilidad (0 a 10) de los siete repos; incluye análisis de sensibilidad.
