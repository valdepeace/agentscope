# DeerFlow (bytedance/deer-flow) - eje ARQUITECTURA

Lectura dirigida, solo lectura, de references/deer-flow/ (clon de 1 commit). Rutas relativas a la raiz del repo. Nada se ejecuto. Lo no leido se marca "no encontrado en lo leido".

## 1. Resumen y veredicto
- Harness de agentes en Python (LangGraph/LangChain): un lead agent con cadena de middlewares, subagentes (en memoria y en lotes durables), sandboxes enchufables, memoria, MCP y canales IM; el runtime del agente corre dentro del Gateway FastAPI (backend/AGENTS.md:3-7, app/gateway/app.py:629).
- Separacion en dos paquetes publicables mas el Gateway: harness (deerflow.*), extension-api (sin importar deerflow) y app.*; la regla "harness no importa app" la hace cumplir un test AST (backend/tests/test_harness_boundary.py:1-8, 37-43). Es la frontera mas limpia del repo.
- Dentro del harness no hay capas: a nivel de import de modulo hay 23 pares con dependencia mutua entre subpaquetes (p. ej. agents<->runtime, agents<->tools, config<->runtime) segun mi recuento de imports (seccion 3). Ademas hay ficheros "dios" de 3000-4400 lineas.
- Robustez ante fallos bien pensada para runs (leases, heartbeat, reconciliacion de huerfanos, drenaje en shutdown) pero los subagentes "task" viven en memoria de proceso y no se reanudan; los durables son los batch.
- Veredicto: arquitectura ambiciosa y bien disciplinada en fronteras externas y en contratos de fallo, con coste alto de complejidad interna (acoplamiento circular, ficheros enormes, 50 clases Middleware). Adecuada como referencia de patrones (leases, aceptacion por codigo, extension API), no como base ligera.

## 2. Mapa de modulos y capas
Medido con git ls-files + wc -l sobre .py (comando en seccion 9 de mantenibilidad). Tamano = ficheros / lineas.

| Modulo | Responsabilidad | Tamano | Depende de (imports de modulo, mi recuento) |
|---|---|---|---|
| backend/packages/harness/deerflow (total) | Framework del agente | 829 / 204.336 | extension-api (pin exacto ==0.2.13, backend/packages/harness/pyproject.toml) |
| deerflow/agents | lead agent, middlewares (50 clases Middleware en 45 ficheros), memoria, prompt | 123 / 43.417 | 24 subpaquetes (sandbox, tools, runtime, subagents, mcp, models...) |
| deerflow/community | proveedores de busqueda/crawl/sandbox (e2b, aio, boxlite, tenki...) | 97 / 29.369 | tools, sandbox, config |
| deerflow/runtime | runs, worker, manager, stream bridge, checkpointer, eventos | 62 / 21.915 | config, persistence, agents, sandbox... |
| deerflow/persistence | repos SQLAlchemy + Alembic (61 ficheros en versions/) | 164 / 21.098 | config, runtime, subagents, mcp |
| deerflow/skills | carga, revision, mutaciones de skills | 49 / 11.006 | |
| deerflow/sandbox | contrato SandboxProvider, tools de ficheros/bash, leases | 26 / 10.266 | agents, tools, runtime, authz |
| deerflow/config | modelos pydantic de config.yaml | 68 / 9.201 | runtime, extensions |
| deerflow/subagents | executor, registry, batch durable, aceptacion | 22 / 8.265 | 15 subpaquetes |
| deerflow/mcp | cliente/cache MCP, OAuth | 25 / 7.941 | |
| deerflow/tools | get_available_tools y tools builtin | 26 / 6.830 | 20 subpaquetes |
| deerflow/extensions | carga de plugins, ordenacion de middleware | 23 / 5.808 | persistence, runtime, tools |
| backend/packages/extension-api | contrato publico de extensiones | 19 / 2.177 | nada (declara no importar deerflow: deerflow_extension_api/__init__.py:3-6) |
| backend/app/gateway | FastAPI: routers, auth, authz, servicios de run | 124 / 40.942 | deerflow.*, extension-api |
| backend/app/channels | adaptadores IM (feishu, discord, slack, telegram...) | 36 / 22.948 | deerflow.*, langgraph_sdk |
| backend/app/scheduler, mcp_tasks, subagent_batches | tareas programadas, tareas MCP, batches | 4+3+4 / 2.304+1.795+203 | |
| frontend/ (Next.js) | UI | 645 ts/tsx fuente / 109.133 | LangGraph SDK (frontend/src/core/api/api-client.ts:4-6) |

Fan-in/fan-out de imports de modulo: mayor fan-out agents 24, tools 20, subagents 15; mayor fan-in config 23, runtime 19, utils 17.
Topologia de servicios: nginx 2026, Gateway 8001, frontend 3000, provisioner 8002 opcional (AGENTS.md raiz, tabla Service topology); nginx reescribe /api/langgraph/ a /api/ (docker/nginx/nginx.conf:106-108).

## 3. Fronteras y acoplamiento
- Frontera app/harness: unidireccional y verificada por test (backend/tests/test_harness_boundary.py). Comprobado con rg que no hay imports "app." dentro del harness (cero coincidencias).
- Frontera extension-api: paquete sin dependencia de deerflow (deerflow_extension_api/__init__.py:3-6); el harness fija version exacta (harness pyproject: deerflow-extension-api==0.2.13).
- Ciclos: script propio sobre 1.336 imports "from/import deerflow.X" del harness (461 dentro de funciones, 875 a nivel de modulo; incluye bloques TYPE_CHECKING porque no los distingui). Pares con import mutuo a nivel de modulo: agents<->sandbox, agents<->tools, agents<->runtime, agents<->subagents, agents<->projects, agents<->community, authz<->guardrails, authz<->sandbox, community<->tools, config<->runtime, config<->extensions, config<->utils, extensions<->tools, mcp<->persistence, mcp<->tools, models<->utils, persistence<->runtime, persistence<->subagents, runtime<->utils, runtime<->sandbox, runtime<->workspace_changes, sandbox<->tools, uploads<->utils (23). Ejemplos leidos: config/agents_config.py:20 importa deerflow.runtime.user_context; runtime/checkpoint_state.py:22 importa deerflow.agents.thread_state; agents/lead_agent/agent.py:71 importa deerflow.runtime.checkpoint_mode. Que sean ciclos reales de ejecucion depende de imports diferidos; no lo verifique pieza a pieza.
- Contradiccion con la guia: agents/middlewares/AGENTS.md y extensions/AGENTS.md dicen que extensions "sits below agents in the import graph"; en lo medido extensions<->tools y tools->agents existen, aunque extensions no importa agents directamente.
- Ficheros "dios" (no test): runtime/runs/worker.py 4.332 lineas (run_agent empieza en :1045 y llega hasta ~:2650, unas 1.600 lineas), app/gateway/services.py 3.716 (start_run :2478-3285, ~800 lineas), runtime/runs/manager.py 3.706, sandbox/tools.py 3.773, app/channels/manager.py 4.418, subagents/executor.py 2.731, lead_agent/agent.py 1.493.
- Donde se cruzan capas: app/gateway/deps.py:830 (langgraph_runtime) ensambla stream bridge, motor, checkpointer, store, repos, RunManager y extensiones en un AsyncExitStack; es el punto de composicion unico.
- Los canales IM no tocan el runtime directamente: usan un cliente langgraph_sdk contra el Gateway (backend/app/channels/manager.py:340-354).

## 4. Estado y persistencia
- Base SQL (SQLite por defecto, PostgreSQL opcional): repos en deerflow/persistence/* (run, feedback, agents, projects, scheduled_tasks, coordination, subagent_batches, mcp_tasks, thread_meta, tokens PAT, etc.), motor en persistence/engine.py.
- Migraciones: Alembic, 61 ficheros en persistence/migrations/versions (0001..0057 mas duplicados de numero 0019, 0023, 0049 y un 0051_merge_run_inputs_fencing). Se aplican al arrancar con bootstrap_schema; BD vacia = create_all + stamp a head; revision desconocida detiene el arranque (persistence/migrations/AGENTS.md, secciones Bootstrap). Convencion: columnas nuevas nullable, helpers idempotentes.
- Checkpoints LangGraph: make_checkpointer(config) (app/gateway/deps.py, dentro de langgraph_runtime); modo de canal "full"/"delta" congelado por proceso (runtime/AGENTS.md seccion Checkpoints; agent.py:875-915).
- Stream de eventos: bridge memory o redis (runtime/stream_bridge/async_provider.py:58-90); multi-worker exige Postgres (deps.py:160 _enforce_postgres_for_multi_worker).
- Ficheros: DEER_FLOW_HOME por usuario/hilo (uploads, workspace, outputs), memoria DeerMem un .md por hecho (agents/memory/AGENTS.md), config.yaml y extensions_config.json escribibles por el Gateway (AGENTS.md raiz, Repository map). config.yaml se relee en caliente, pero motores y stores son "restart-required" (deps.py:830-850 docstring).
- Quien escribe el estado de un run: solo el worker que posee la lease commitea el terminal (runtime/AGENTS.md seccion Admission and terminal commit). Transacciones: no encontre una transaccion que abarque checkpoint + fila de run; se usan escrituras condicionales con reintento (manager.py:555 _call_store_with_retry, claim_for_takeover en :2850 aprox.).

## 5. Flujo de control de una tarea
1. nginx recibe /api/langgraph/* y reescribe a /api/* hacia el Gateway (docker/nginx/nginx.conf:106-108). El frontend usa LangGraph SDK (frontend/src/core/api/api-client.ts:4-6).
2. create_app() registra ~45 routers (app/gateway/app.py:1102-1458); el de runs llama a start_run (app/gateway/services.py:2478).
3. start_run comprueba acceso al hilo (services.py:2597-2598), resuelve agent_factory (:2616), valida/normaliza input (rechaza mensajes system/developer, :639-642), construye config y contexto (build_run_config :1658).
4. Admision: run_mgr.create_or_reject (services.py:3119; manager.py:2306) con idempotency key y un solo run activo por hilo (BusyThreadConflict, services.py:3260).
5. Se lanza el worker: record.task = asyncio.create_task(worker) (services.py:3224) que ejecuta run_agent (worker.py:1045; llamada en services.py:3048).
6. run_agent ensambla el agente con assemble_lead_agent (agents/lead_agent/agent.py:875), que construye middlewares (build_middlewares :486, base en agents/middlewares/tool_error_handling_middleware.py:399/595) y llama create_agent (agent.py:1294/1452).
7. Bucle del modelo: agent.astream(..., stream_mode=...) (worker.py:1783 y 1828); cada item se publica por el stream bridge (worker.py:4295 _publish_stream_item) y el cliente lo consume por SSE (services.py:3481 sse_consumer).
8. Herramientas: get_available_tools (tools/tools.py:163) mezcla tools de config.yaml (resolve_variable), builtins, MCP, subagentes. Una tool de sandbox adquiere sandbox via provider (sandbox/tools.py:2213 ensure_sandbox_initialized).
9. Delegacion: la tool task crea un SubagentExecutor y llama execute_async (subagents/executor.py:2401) en un loop aislado; la tool sondea el resultado (tools/builtins/task_tool.py:259-306). Estados en executor.py:102.
10. Cierre: commit terminal por el worker propietario, publicacion de END y limpieza; hooks de finalizacion y goal (worker.py:2658+). Heartbeat renueva leases cada lease_seconds/3 (manager.py:3226).

## 6. Puntos de extension (ficheros a tocar)
- Proveedor de modelo: clase BaseChatModel referenciada por "use" en config.yaml, resuelta por reflexion (deerflow/models/factory.py:576, reflection/resolvers.py:103). Minimo: 1 fichero nuevo + config.yaml (0 cambios en core).
- Sandbox/harness de ejecucion: sandbox.use -> clase SandboxProvider (sandbox/sandbox_provider.py:242; config.example.yaml:1764). Existen 8+ proveedores en community/. Minimo: paquete nuevo + config.
- Herramienta: entrada "use: modulo:variable" en config.yaml (tools/tools.py:191-193) -> 1 fichero + config; las tools builtin exigen editar BUILTIN_TOOLS y la lista de asignacion (tools/AGENTS.md "Subagent, bootstrap and embedded assembly get a narrower list").
- Rol (subagente): 3 vias: builtin en subagents/builtins/ y registro en BUILTIN_SUBAGENTS, custom_agents en config.yaml, o definicion gestionada por admin (subagents/registry.py:110-130). Via config = 0 ficheros de codigo.
- Canal IM: anadir clase en app/channels/<x>.py y entradas en _CHANNEL_REGISTRY y _CHANNEL_CREDENTIAL_KEYS (app/channels/service.py:33-55) = 2 ficheros minimo; el test de registro no lo verifique.
- Extension de producto: deerflow_extension_api con middleware, servicios, routers (extensions/AGENTS.md); la regla del repo es evaluar primero como extension (AGENTS.md raiz, "Evaluate an extension first").
- Middleware nuevo: contribuido con colocacion semantica, compuesto por extensions/ordering.py (extensions/AGENTS.md seccion Composition); un middleware del core requiere editar build_middlewares (agent.py:486-800).

## 7. Comportamiento ante fallos a mitad
- Crash del Gateway con runs en vuelo: al arrancar, reconcile_orphaned_inflight_runs marca como error los runs con lease vencida (o sin lease en modo single-worker) mediante claim_for_takeover; NO los reanuda (manager.py:2798-2900; llamada en app/gateway/deps.py:1140-1160 aprox., "Startup recovery"). Se publica marcador terminal en el stream (deps.py:549-700).
- Multi-worker: heartbeat renueva leases y cada 3 ciclos reconcilia huerfanos de pares muertos (manager.py:3226-3262).
- Shutdown: _drain_inflight_runs protege el cierre del checkpointer con shield (deps.py:510-548, issue #3373).
- Timeout de subagente: asyncio.wait con timeout_seconds, publica TIMED_OUT y cancela, supervisando la limpieza (executor.py:2401-2480). El timeout efectivo de builtins es el global subagents.timeout_seconds=1800, no el 900 del dataclass (subagents/config.py:55; config/subagents_config.py:185-188; registry.py:110-150).
- Crash con subagentes "task": viven en _background_tasks de proceso (executor.py:2401-2480); no encontre persistencia ni reanudacion de ellos. Doc admite que observacion de workers usa handle local y reconciliacion llega "en la siguiente etapa" del RFC #8345 (backend/docs/task-coordination-state.md:98).
- Batch durable: items con lease y heartbeat; al apagar no se finaliza, la lease expira y otro worker reclama con la misma clave estable (subagents/batch_service.py:666-667); escrituras comprobadas contra el propietario de la lease (:590-652).
- Aislamiento de extensiones: un middleware contribuido falla sin romper el run (IsolatedMiddleware, extensions/AGENTS.md).
- Config invalida: ValueError al arrancar para combinaciones peligrosas (SQLite con varios workers, deps.py:160).

## 8. Decisiones de diseno notables y su coste
- Gateway con runtime embebido (sin servidor LangGraph aparte): simplifica despliegue; coste: un proceso concentra API, runs y canales, y obliga a mucha logica de recuperacion en el Gateway (deps.py ~1.600 lineas).
- Comportamiento en middleware (50 clases) y orden-es-comportamiento (agents/middlewares/AGENTS.md): flexible; coste: orden fragil y dificil de auditar.
- Un unico lead agent LLM decide delegar (sin router determinista); limites duros por codigo: max total por run 6 (subagents_config.py:11, 195), concurrencia 3 por defecto (config.example.yaml:2070; subagents_config.py:28-37) con tope 64.
- Aceptacion de subagentes decidida en codigo (subagents/acceptance_checks.py, no abierto entero).
- Extension API con version exacta fijada: contrato estable; coste: acoplamiento de versiones.
- Ficheros monoliticos y comentarios de diseno largos dentro de funciones: la logica de concurrencia es muy defensiva (executor.py:2401-2540) a cambio de lectura dificil.
- Sandbox por defecto local, con allow_host_bash false (config.example.yaml:1764-1768; backend/docs/CONFIGURATION.md:1890, 2401-2408): seguro por omision para bash pero no es frontera de aislamiento.

## 9. Cobertura de esta lectura
- Leidos con detalle (aprox.): app/gateway/app.py (cabecera y estructura), deps.py 830-1205, services.py (indice + firmas, start_run por muestreo de llamadas), runs/manager.py (indice + 2798-2910 + 3180-3262), worker.py (indice), executor.py 100-112 y 2401-2600, registry.py 110-200, subagents_config.py, tools.py 163-330, channels/service.py 30-60, sandbox_provider.py, AGENTS.md raiz/backend/middlewares/tools/extensions/runtime/persistence-migrations/tests, security.md, config.example.yaml (secciones sueltas), nginx.conf (grep).
- Aproximado: ~3.500 lineas de codigo fuente leidas de ~275.000 (backend .py fuente) = 1-2 %; ~20 ficheros de guia/docs de 180 (~10 %). Leido por busqueda sistematica (no por lectura): imports de todo el harness (grep), recuento de lineas, TODOs, links de docs.
- No leido: frontend (salvo package.json y api-client cabecera), memoria DeerMem/mem0/honcho, canales IM (salvo registro y cliente SDK), providers de sandbox de community, routers del Gateway, skills, guardrails y authz internos, acceptance_checks, scheduler, mcp, TUI, deploy/helm, examples.
- Correcciones a la nota previa (study/deer-flow.md): (1) timeout de builtins: 900 es el valor del dataclass, pero el efectivo es 1800 (subagents_config.py:185); (2) "max 3 concurrentes" es solo el valor por defecto de subagent_runtime.max_running, configurable hasta 64; (3) "~40+ middlewares": hay 50 clases Middleware en 45 ficheros; (4) skills publicas son 23, no 22; (5) SECURITY.md:5 y no :3; ademas GitHub si muestra releases v2.0.0, v2.1.0, v2.2.0 (ver eje mantenibilidad); (6) workflows en el repo: 21 ficheros (no ~20); (7) la nota omitia que allow_host_bash es false por defecto (config.example.yaml:1768). Verificadas como correctas: licencia MIT, make_lead_agent en agent.py:870, estados de executor.py:102, MAX_CONCURRENT_SUBAGENTS en executor.py:2586, ~1020 .py en harness+app+extension-api (829+172+19), 1.378 ficheros test_*.py.
