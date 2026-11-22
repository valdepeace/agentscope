# DeerFlow 2.0 (ByteDance) - notas de estudio (solo lectura)

Raiz estudiada: `references/deer-flow/`. Nada se ejecuto; AGENTS.md/CLAUDE.md se leyeron como objeto de estudio. Muestreo, no lectura exhaustiva: lo no mencionado es "no leido", no "inexistente".

## 1. Resumen
- Que es: "super agent harness" (README.md:13) que orquesta subagentes, memoria y sandboxes con skills extensibles. Reescritura total de v1 (Deep Research), que no comparte codigo (README.md:17).
- Stack: backend Python 3.12+ (LangGraph/LangChain, FastAPI Gateway con runtime embebido), frontend Next.js (solo: UI de chat web + paneles), nginx en :2026, Docker/Helm (AGENTS.md raiz, backend/docs/ARCHITECTURE.md).
- Licencia VERIFICADA: **MIT** (`LICENSE:1` "MIT License", copyright ByteDance 2025 + DeerFlow Authors 2025-2026; badge README.md:5 coincide). No es Apache 2.0.
- Tamano: ~4770 ficheros en el repo; backend con ~1020 .py (packages+app) y ~1377 ficheros `test_*.py`; backend/docs con ~19.000 lineas de docs; README en 7 idiomas; CHANGELOG de 1 MB.
- Madurez: alta en volumen de ingenieria (CI con ~20 workflows, migraciones SQL, multiusuario), pero SECURITY.md:3 dice que "no hay release oficial todavia"; diseno pensado para entorno local de confianza (docs/guide/security.md:3).

## 2. Modelo de dominio
- **Thread**: conversacion con estado checkpointeado LangGraph; directorio propio `users/{user_id}/threads/{thread_id}/user-data` (workspace/uploads/outputs) (backend/docs/AUTH_DESIGN.md, sandbox/AGENTS.md).
- **Run**: una ejecucion del agente sobre un thread, API compatible con LangGraph (`/api/threads/{id}/runs`, ARCHITECTURE.md); streaming SSE, eventos (docs/RUN_EVENT_STREAM.md).
- **Lead agent**: unico agente principal (`make_lead_agent`, agents/lead_agent/agent.py:870) con cadena de middlewares.
- **Subagente**: `task` (en memoria, espera) o `batch_task` (filas SQL durables con lease) (subagents/AGENTS.md:1-8).
- **Task de coordinacion**: unidad de trabajo con inputs ordenados, hijos, dependencias, epoch de ejecucion; IDs independientes de Thread/Run (docs/task-coordination-state.md).
- **Skill**: paquete con `SKILL.md` (frontmatter) en `skills/public|custom`; cargado y proyectado a `/mnt/skills` (skills/AGENTS.md).
- **Memoria**: hechos por usuario (un .md por hecho) + resumenes en `memory.json` (memory/AGENTS.md).
- **Project**: contexto fijado + estante de documentos, con papelera (ARCHITECTURE.md).
- **Channel**: adaptador de chat IM (Feishu, Discord, DingTalk, Telegram...) al MessageBus (app/channels/AGENTS.md).
- Relaciones: User -> Projects -> Threads -> Runs; Run -> lead -> N subagentes (max 3 concurrentes en memoria) o batch durable; Thread <-> sandbox por (user_id, thread_id).

## 3. Orquestador (equivalente al Router)
- Un unico "lead agent" LLM decide TODO: responder, usar tools, delegar. No hay router determinista separado. Entrada: `make_lead_agent` (agent.py:870); langgraph.json lo registra.
- Modelo: configurable en `config.yaml` (cualquier proveedor LangChain; README recomienda Doubao/DeepSeek v3.2/Kimi); el subagente hereda modelo (`model="inherit"`).
- Herramientas: sandbox (bash, ls, glob, grep, read_file, write_file, str_replace), `task`/`batch_task`, `ask_clarification`, `present_files`, MCP, busqueda/crawl (community/), memoria, skills.
- Politica de delegacion en prompt: "Delegation is not the default"; solo si latencia paralela, skill especialista o aislamiento de contexto compensan (subagents/AGENTS.md; general_purpose.py descripcion).
- Cadena de ~40+ middlewares (loop detection, guardrails, summarization, token budget, PII, skills, delegation ledger...) en agents/middlewares/. Mucho comportamiento vive en middleware y prompt (prompt.py 63 KB, agent.py 73 KB).

## 4. Subagentes (equivalente al Worker)
- Config: `SubagentConfig` (subagents/config.py:54-55): max_turns 50 por defecto, timeout 900 s; general-purpose max_turns=150 (builtins/general_purpose.py), `disallowed_tools=["task","ask_clarification","present_files"]` (sin anidamiento). Hay tambien `bash_agent`.
- Aislamiento: hereda alcance (identidad, skills del usuario, scope de conocimiento, copia de lista de uploads) pero NO el historial de mensajes (subagents/AGENTS.md "Isolation"). Comparte el MISMO sandbox del padre, con lease propio por tarea para matar solo sus procesos.
- Allowlist/denylist de tools auditada justo antes de `create_agent`; denylist gana.
- Reporte: contrato de informe + `acceptance_checks.py`: la aceptacion se decide en codigo contra resultados de tools registrados, el informe del subagente no puede marcar criterios cumplidos; "unverified es el resultado seguro".
- Ciclo de vida: estados PENDING/RUNNING/COMPLETED/FAILED/CANCELLED/TIMED_OUT (executor.py:102); "el primer write terminal gana"; limpiar antes de notificar; `stop_reason` aditivo para caps. Limite `MAX_CONCURRENT_SUBAGENTS = 3` (executor.py:2586). `execute_async` (executor.py:2401).
- Batch durable: lease por item con heartbeat, escrituras con comprobacion SQL del propietario del lease, `depends_on` con deteccion de ciclos (docs/BATCH_CONTINUATION.md), ejecutable en un worker separado.

## 5. Estado y memoria
- Checkpoints LangGraph + persistencia SQL (SQLite o PostgreSQL), migraciones Alembic (`make migrate-rev`), repositorios con leases (persistence/AGENTS.md). Retencion de checkpoints: docs/checkpoint-retention-contract.md.
- Memoria: backend por defecto "DeerMem" local; hechos = un fichero Markdown cada uno; `memory.json` solo resumenes y revision; adaptadores opcionales mem0, openviking, honcho (memory/AGENTS.md). Escritura diferida con cola/debounce, redaccion PII antes de extraer, confirmacion solo con evidencia humana, una unica ruta de evicion. Inyeccion en prompt con tope `max_injection_tokens` (docs/MEMORY_COGNITIVE_STYLE.md:40).
- Ficheros por usuario/hilo en `DEER_FLOW_HOME`; blobs (docs/blob-storage.md); artefactos con SHA-256 (task-coordination-state.md).

## 6. Supervision y manejo de fallos (Watchdog?)
No hay un proceso "Watchdog" unico; lo equivalente esta repartido:
- LoopDetectionMiddleware: hash de llamadas identicas sin progreso -> aviso/hard stop; frecuencia solo observa (docs/LOOP_DETECTION.md).
- Timeouts, max_turns, token budget (token_budget_middleware), SubagentLimitMiddleware, LLM error handling, dangling tool call repair.
- Leases + heartbeat + recuperacion de items batch tras reinicio; scheduler con aislamiento de fallos (app/scheduler/guide/failure-isolation.md).
- Cancelacion con fence por epoch y confirmacion asincrona (HTTP 202 `cancelling`).
- Limite declarado: la observacion de workers usa handle de proceso local; reautorizacion y reconciliacion tras reinicio estan "en la siguiente etapa" del RFC #8345 (task-coordination-state.md, final) = PROMETIDO.

## 7. Seguridad y autorizacion
- Sandbox: `local` (por defecto; ejecuta en el propio contenedor gateway, "no es frontera fuerte", docs/CONFIGURATION.md ~2395-2410), `aio` (Docker via socket del host = equivalente a root del host) y `provisioner`/Kubernetes (el mas fuerte). Providers comunitarios: e2b, boxlite, opensandbox...
- `build_sandbox_env()` limpia credenciales heredadas; autorizacion `sandbox:execute` antes de reutilizar un sandbox; traduccion de rutas host->virtuales; aprobacion humana para red (sandbox/AGENTS.md).
- Guardrails: middleware que evalua cada tool call contra un provider enchufable (AllowlistProvider, openai_decisions) (docs/GUARDRAILS.md; guardrails/).
- Auth: sesion/JWT con cookie + CSRF, aislamiento por usuario en repos/ficheros/memoria, roles solo admin/user, sin RBAC fino; limite de login en memoria de proceso (AUTH_DESIGN.md "no objetivos"). Admin del Gateway = ejecucion de codigo en el host (docs/guide/security.md).
- Prompt injection: rechazo 400 de mensajes `system`/`developer` del cliente (security.md); el contenido recordado se inyecta como HumanMessage, no SystemMessage; PoC de inyeccion incluido (tests/poc_external_system_message_injection.py citado). No hay defensa general contra injection en contenido web/skills salvo scanner offline de skills (skills/skillscan) y PII redaction; el propio doc admite que injection + socket Docker pivotaria al host.
- SECURITY.md (raiz): solo 398 bytes, remite a GitHub Security; sin politica detallada.
- Diseno por defecto: loopback 127.0.0.1; exponerlo exige medidas propias.

## 8. Madurez real
- IMPLEMENTADO (verificado en codigo/estructura): lead agent, middlewares, `task`/executor (2731 lineas), sandbox local + providers, memoria DeerMem, guardrails, gateway multiusuario, canales IM (feishu.py 111 KB, discord 69 KB...), batch durable, migraciones, ~1377 ficheros de test, CI amplio (unit, blocking-io, e2e, replay, nightly, sandbox smoke, Windows live policy).
- Tests live opt-in con APIs reales (backend/AGENTS.md); suites offline sin red ni API key.
- PROMETIDO / en curso: reconciliacion tras reinicio y reautorizacion automatica (RFC #8345); sin RBAC fino; sin release oficial (SECURITY.md:3); docs/plans y docs/experiments contienen trabajo no consolidado (no leidos). Skill "allowed-tools" es solo metadato (skills/AGENTS.md).
- Muchos docs backend estan comprimidos/ruidosos (ej. ARCHITECTURE.md seccion Projects ilegible parcialmente); no verifique cada afirmacion contra codigo.

## 9. Ideas originales / lo distintivo
- Skills como paquetes Markdown (22 publicas: deep-research, ppt, podcast, video...) con scanner offline, revision y "skill evolution host" que muta skills (docs/skill-evolution-host.md).
- Message gateway: canales IM como clientes del Gateway via langgraph-sdk, lider por lease en multi-worker, dedupe, allowlist de texto de stream para no filtrar contexto oculto (app/channels/AGENTS.md).
- Aceptacion de subagentes decidida por codigo sobre evidencia de tools (no por autoinforme).
- Sandbox con identidad (user_id, thread_id) y leases dobles (inter-instancia y local).
- Extension API publica y regla "evalua primero como extension" (AGENTS.md raiz).
- Memoria pluggable y "goal"/continuacion de runs, scheduled tasks, projects con pinned context.

## 10. Carencias y huecos
- Orquestador unico LLM sin separacion Router/Worker por modelo; costes y decisiones dependen de un prompt enorme.
- Subagentes comparten sandbox del padre (aislamiento de filesystem parcial); max 3 concurrentes en memoria.
- Sandbox por defecto debil; el camino "facil" (aio) monta socket Docker.
- Sin Watchdog explicito, sin reconciliacion completa tras reinicio, rate limit no global.
- Sin RBAC; politica de seguridad minima; pensado para localhost de confianza.
- Complejidad enorme (cientos de middlewares/modulos): dificil de auditar; docs desiguales.

## 11. Diferencias de enfoque respecto a Router/Worker estilo DeepSeek
- DeerFlow: un lead agent generalista que delega opcionalmente; DeepSeek-style: Router barato decide/enruta y Workers ejecutan acotados. Aqui el "router" es el propio LLM + prompt.
- Subagentes del mismo modelo (`inherit`), con tools heredadas menos denylist; Worker estilo DeepSeek tendria contrato mas estrecho.
- Supervision: middlewares y leases SQL frente a un Watchdog dedicado.
- Verificacion: acceptance_checks en codigo es una idea a copiar (el Worker no se autocertifica).
- Infra: servidor multiusuario con Postgres, auth y canales IM, frente a un harness ligero local.
- Seguridad: confia en sandbox + guardrails opcionales; el linaje DeepSeek debe decidir si exige sandbox fuerte por defecto.

## Evidencias
| Afirmacion | Ruta:linea |
|---|---|
| Licencia MIT | deer-flow/LICENSE:1 ; README.md:5 |
| Super agent harness; v2 sin codigo de v1 | README.md:13,17 |
| Sin release oficial | SECURITY.md:5 |
| Lead agent factory | backend/packages/harness/deerflow/agents/lead_agent/agent.py:870 |
| Estados de subagente | backend/packages/harness/deerflow/subagents/executor.py:102 |
| Max 3 subagentes concurrentes | .../subagents/executor.py:2586 |
| max_turns 50 / timeout 900 por defecto | .../subagents/config.py:54-55 |
| general-purpose sin `task`, max_turns 150 | .../subagents/builtins/general_purpose.py (final) |
| Aceptacion decidida en codigo | backend/packages/harness/deerflow/subagents/AGENTS.md (seccion Evidence and acceptance) |
| Batch durable con lease | backend/docs/BATCH_CONTINUATION.md ; subagents/AGENTS.md (Durable batches) |
| Reconciliacion tras reinicio pendiente (RFC #8345) | backend/docs/task-coordination-state.md (final) |
| Loop detection | backend/docs/LOOP_DETECTION.md |
| Sandbox local no es frontera fuerte; socket Docker = root | backend/docs/CONFIGURATION.md ~2395-2420 |
| Gateway admin = ejecucion de codigo; rechazo de system msgs | docs/guide/security.md:10-25 |
| Guardrails pre-tool-call | backend/docs/GUARDRAILS.md:1-10 |
| Memoria DeerMem un .md por hecho | backend/packages/harness/deerflow/agents/memory/AGENTS.md (DeerMem storage) |
| Auth: sin RBAC, rate limit en proceso | backend/docs/AUTH_DESIGN.md (No objetivos) |
| Canales IM como clientes del Gateway | backend/app/channels/AGENTS.md:1-8 |
| ~1377 test files, ~1020 .py | recuento con find sobre backend/tests, packages, app |
