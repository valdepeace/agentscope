# agent-swarm: estudio de arquitectura

Repo: `desplega-ai/agent-swarm`, clon superficial en `references/agent-swarm/`, versión 2.14.2 (`package.json`). Solo lectura; no se ejecutó nada. Las rutas son relativas a la raíz del repo. Segunda lectura dirigida: corrige la nota previa `study/agent-swarm.md` donde se indica en la sección 9.

## 1. Resumen y veredicto

- Arquitectura de dos mitades: un servidor API/MCP (Bun) que es único dueño de un SQLite en WAL, y procesos worker/lead que hablan con él solo por HTTP (`CLAUDE.md:9`), con 12 adaptadores de harness intercambiables (`src/types.ts:519-532`, `src/providers/index.ts:19-71`).
- El control de tareas es "poll + claim atómico + reconciliación": el worker pregunta a `/api/poll` (`src/commands/runner.ts:4041-4108`), el servidor reclama con un `UPDATE ... WHERE status='unassigned'` (`src/be/db.ts:3217-3221`) y un barrido de heartbeat cada 90 s repara lo que quede colgado (`src/heartbeat/heartbeat.ts:124-126, 2085-2092`).
- Los límites entre capas están vigilados por CI (script grep + dependency-cruiser), pero el propio dependency-cruiser deja la regla transitiva en `warn` porque hay violaciones conocidas (`.dependency-cruiser.cjs:29-34`). Hay ciclos de directorio abundantes (sección 3).
- Hay módulos "dios": `src/be/db.ts` (14.753 líneas, 455 líneas `export`) y `src/commands/runner.ts` (8.334 líneas, con `runAgent` desde la 6374 hasta el final).
- Veredicto: diseño operativo y bien instrumentado contra regresiones (gates, e2e de contrato), con deuda estructural concentrada en el núcleo (db.ts, runner.ts) y en el acoplamiento `be` con casi todo. Fiabilidad ante crash mejor documentada y comprobada que la media; sin tope de duración por tarea encontrado en lo leído.

## 2. Mapa de módulos y capas

Medición: ficheros `.ts/.tsx` bajo `src/` excluyendo `*.test.*` y `src/tests/`, y líneas con `git ls-files ... | xargs cat | wc -l`. Total: 1.252 ficheros `.ts`, 362.665 líneas (ts+tsx). "Depende de" sale de un escaneo automático de imports relativos de los 1.252 ficheros (ver limitación abajo).

| Módulo | Responsabilidad | Ficheros / líneas | Depende de (nº de imports, mayores) |
|---|---|---|---|
| `src/be` (+`be/db/`, `be/db-queries/`, `be/migrations`) | Persistencia, db-client, migraciones, seeds, runtime pools, scripts, memoria, modelos | 308 / 99.433 | utils 150, raíz 106, workflows 22, scripts-runtime 15, runtime-providers 14 |
| `src/http` | Rutas REST (`route()`), poll, webhooks, OpenAPI | 116 / 54.383 | be 336, raíz 154, utils 147, workflows 23 |
| `src/tools` | Herramientas MCP (registradas por capability en `src/server.ts`) | 184 / 26.220 | raíz 11, apps 6, be 5 (infravalorado, ver nota) |
| `src/commands` | Worker/lead runner, CLI, e2b | 63 / 23.803 | utils 70, providers 31, raíz 15 |
| `src/providers` | 12 adaptadores de harness + utilidades | 57 / 22.503 | utils 100, hooks 14, prompts 7 |
| `src/slack` | Integración Slack | 54 / 15.630 | be 48, utils 41, raíz 21 |
| `src/workflows` | Motor de workflows, event-bus | 62 / 15.602 | be 49, raíz 46, utils 20 |
| `src/utils` | Utilidades compartidas (worker y API) | 86 / 14.764 | raíz 7 |
| `src/scripts-runtime` | Sandbox de scripts TS | 48 / 12.558 | utils 12 |
| `src/apps`, `integrations` (chat-sdk, kapso), `github`, `tasks`, `hooks` | Apps/pages; canales Chat SDK; webhook GitHub; follow-ups de tareas; hooks de Claude | 13/7.833; 39/7.505; 9/3.294; 14/3.632; 7/3.662 | be, utils, prompts, tasks |
| `src/heartbeat` | Barrido de salud/crash | 4 / 2.410 | be 14, utils 5 |
| `src/extensions`, `prompts`, `realtime`, `oauth`, `rbac`, `runtime-providers`, `scheduler`, `script-workflows`, trackers (`jira`, `linear`), VCS (`gitlab`, `bitbucket`, `azure-devops`), otros | 2.749 / 2.670 / 2.636 / 2.194 / 1.112 / 1.795 / 1.418 / 1.713 / 2.101 / 1.888 / 1.053 / 808 / 2.065 líneas | resto, cada uno < 15 ficheros | be, utils, prompts, tasks |
| Fuera de `src/` | `apps/ui` (1.616 ficheros), `apps/evals` (358), `apps/templates-ui` (45), `packages/{model-catalog,model-routing,ui-e2e}` (17/22/66), `templates/` (353), `docs-site/` (344), `thoughts/` (872), `scripts/` (413) | no leído salvo lo citado | `packages/model-routing` no puede importar `src/` (`.dependency-cruiser.cjs:35-42`) |

Entradas: `src/http.ts` solo importa `./http/index` (más dos efectos secundarios; `src/http.ts:1-4`); `src/server.ts` (821 líneas) registra herramientas por capability (`src/server.ts:517-545`); `src/cli.tsx` enruta comandos (`CLAUDE.md`, bloque CLI).

Limitación del escaneo: 257 ficheros de `src/` importan con el alias `@/` (`tsconfig.json`, `paths`), que mi escaneo no resolvió. Por eso las cifras de acoplamiento son cotas inferiores; p. ej. `tools -> be` sale 5 pero 130 ficheros de `src/tools` mencionan `be/db` (grep).

## 3. Fronteras y acoplamiento

**Frontera declarada API/worker.** `CLAUDE.md:9` dice que el código worker (`commands`, `hooks`, `providers`, `prompts`, `cli.tsx`, `claude.ts`) nunca importa `src/be/db` ni `bun:sqlite`. Comprobado, con matices:
- El script `scripts/check-db-boundary.sh` vigila más rutas que el doc: añade `runtime-providers`, `scripts-runtime` y `utils` (script líneas 12-25 aprox.; `.dependency-cruiser.cjs:14-15` igual). Desfase doc vs código: `CLAUDE.md:9` no los menciona.
- Solo se prohíbe `be/db`; el worker sí importa otras piezas de `be/`: `src/hooks/hook.ts:18`, `src/providers/codex-adapter.ts:40`, `src/providers/pi-mono-extension.ts:10` y `src/utils/internal-ai/summarize-session.ts:23` importan `be/memory/raters/llm`; `src/commands/claude-managed-setup.ts:36` importa `be/seed-skills/render`; `src/utils/runtime-model-catalog.ts:21` importa `be/modelsdev-cache.json`. No leí `raters/llm` para saber si toca DB.
- La regla transitiva "el worker no alcanza `be/db` por cadena de imports" está en `warn`: el comentario admite tres ficheros híbridos (`commands/{worker,lead}.ts -> server.ts -> be/db*`; `cli.tsx -> be/scripts/maintenance.ts`) que pueden arrancar la API en proceso (`.dependency-cruiser.cjs:29-34`).
- Hay solo 3 reglas en dependency-cruiser (`.dependency-cruiser.cjs:20-42`): no hay reglas entre el resto de módulos.

**Ciclos (a nivel de directorio, imports relativos; formato A->B / B->A).** be<->workflows 22/49; be<->slack 2/48; be<->tasks 7/13; be<->heartbeat 1/14; be<->extensions 7/12; be<->realtime 5/6; be<->oauth 3/4; be<->tools 1/5; be<->ui-spec 1/7; be<->commands 1/1; http<->slack 13/1; http<->realtime 10/1; extensions<->http 1/9; prompts<->providers 1/7; slack<->workflows 5/6. El caso más claro: `src/be/db.ts:3164-3166` hace `import("../workflows/event-bus")` dinámico dentro de `afterCommit`, es decir, la capa de datos emite eventos a workflows con un import perezoso que esquiva el ciclo estático.

**Módulos dios.**
- `src/be/db.ts`: 14.753 líneas, 455 líneas `^export` (grep); es fachada y a la vez implementación (`createTaskExtended` en 2550, `claimTask` 3188, `claimInboxMessages` 5802, `claimKv` 14008). Más 14.706 líneas en `be/db/` y `be/db-queries/`.
- `src/commands/runner.ts`: 8.334 líneas; `spawnProviderProcess` 4611-5690 (~1.080 líneas) y `runAgent` 6374-fin (~1.960 líneas).
- `src/types.ts` 4.849 líneas (schemas zod de todo el dominio, también usado para OpenAPI según `CLAUDE.md:120`), `src/heartbeat/heartbeat.ts` 2.229.

**Cruces de capa.** `http` depende de `be` con 336 imports y de casi 30 módulos más; `be` depende de `slack`, `extensions`, `runtime-providers`, `scripts-runtime`, `commands` (1) y `tools` (1). El detalle de esos dos últimos no lo abrí.

## 4. Estado y persistencia

| Estado | Dónde vive | Quién escribe | Cita |
|---|---|---|---|
| Dominio (tareas, agentes, sesiones activas, inbox, memoria, workflows, KV, config) | SQLite único, WAL, `busy_timeout=5000`, `foreign_keys=ON`, `synchronous=NORMAL` | Solo la API | `src/be/db/runtime.ts:153-159`; `CLAUDE.md:9` |
| Esquema | `src/be/migrations/NNN_*.sql` (261 .sql), tabla `_migrations` con checksum | Arranque de la API | `src/be/migrations/runner.ts:183-322` |
| Config operativa | tabla `swarm_config`, hidratada a `process.env` al arrancar | API (UI de configuración) | `src/heartbeat/heartbeat.ts:129-135`; `CLAUDE.md` (bloque env var) |
| Identidad/ficheros del agente, repos clonados | Volúmenes del contenedor (`/workspace/personal`, `/workspace/shared`) | Worker | nota previa, compose; no releído |
| Procesos de provider y cola local de tareas | Memoria del runner (`state.activeTasks`) | Worker | `src/commands/runner.ts:2725-2733` |

**Acceso y transacciones.** Todo el acceso en runtime pasa por `getDbClient()` (`CLAUDE.md:50`). El cliente serializa transacciones con un lock FIFO; abre `BEGIN IMMEDIATE` salvo `readOnly`, anida con SAVEPOINT, y ejecuta `afterCommit` tras el COMMIT (`src/be/db-client.ts:509-548, 572-576`). Reintenta BUSY con `busy_timeout` corto y un plazo global (`db-client.ts:44-50, 198-236`). Un poll inactivo corre en transacción de solo lectura y solo reabre en escritura si necesita escribir (`src/http/poll.ts:217-218, 804-808`).

**Migraciones.** Solo hacia delante; cada una se aplica en su propia transacción con `.immediate()` y FK desactivadas durante toda la pasada (`runner.ts:275, 296-303`). Un fallo lanza error con el nombre de la migración y el arranque cierra la DB (`runner.ts:304-310`; `runtime.ts:169-180`). Un checksum distinto en una migración ya aplicada **solo avisa** (`runner.ts:282-288`); la protección real es la política y `scripts/check-migration-conflicts.sh` en CI (`.github/workflows/merge-gate.yml:1726`). Existe un parche de renumeración puntual (`repairRenumberedModelTiers`, `runner.ts:51-83, 224`), señal de que alguna vez se renumeró una migración ya desplegada. Si no se encuentra el directorio de migraciones con DB vacía, se niega a arrancar (`runner.ts:106-125, 197-204`).

## 5. Flujo de control de una tarea, de punta a punta

Lo que sigue es lo que abrí; los pasos de origen (Slack, GitHub, workflow, lead delegando) no se leyeron en detalle.

1. Creación: `createTaskExtended` inserta la fila y registra `afterCommit` para emitir `task.created` en el event-bus de workflows (`src/be/db.ts:2550, 3164-3183`).
2. Arranque del worker: registro con reintentos (`runner.ts:3896-3936`), `cleanupActiveSessions` y `recoverOrphanedInProgressTasks` contra la API (`runner.ts:7097-7099, 3441-3481`).
3. Bucle principal: `while (shouldRunDeliveryLoop(...))` (`runner.ts:7664`) llama a `checkCompletedProcesses`, `reconcileActiveTasks` y `pollForTrigger` (`7677, 7771, 7836`); la ventana de poll es 60 s, o 5 s si hay tareas activas (`6615, 7832-7833`).
4. El poll llega a `/api/poll`; el servidor busca tarea pendiente del agente (`getPendingTaskForAgent`, `src/http/poll.ts:493`) y la marca `in_progress` (`poll.ts:543`), o intenta reclamar una de pool con `claimTask` (`poll.ts:762`).
5. `claimTask` comprueba elegibilidad por rol/afinidad (`isAgentEligibleForTask`) y reclama con un único UPDATE guardado por `status='unassigned'` (`src/be/db.ts:3188-3221`); registra `task_claimed` y emite `emitTaskStarted` (`3225-3239`).
6. El worker construye el prompt (`buildPromptForTrigger`, `runner.ts:7907`), prepara el repo (`ensureRepoForTask`, `8054`) y lanza el proceso (`spawnProviderProcess`, `8222`); el adaptador sale de `createProviderAdapter` (`runner.ts:6452`; `src/providers/index.ts:19`).
7. Mantiene viva la sesión: `registerActiveSession` (`runner.ts:3381`) y heartbeats por actividad (`src/http/active-sessions.ts:211-217`).
8. Termina: `checkCompletedProcesses` (`runner.ts:5923`) y `ensureTaskFinished` envía `completed`/`failed` con reintentos exponenciales hasta 300 s (`runner.ts:1914, 1877-1912`).
9. En paralelo, el servidor reconcilia: `reconcileActiveTasks` aborta sesiones de tareas ya terminales o canceladas (`runner.ts:5807`, intervalo 30 s en `5680`, gracia 45 s en `5687`).

## 6. Puntos de extensión

| Extender | Qué hay que tocar | Ficheros (orden de magnitud) |
|---|---|---|
| Harness/proveedor | Adaptador en `src/providers/`; `ProviderNameSchema` (`src/types.ts:519`); `createProviderAdapter` (`src/providers/index.ts:19`); `PROVIDER_STEER_CAPABILITIES` (`types.ts:770`); `docker-entrypoint.sh`; `runtime-dependencies.ts`; pricing (`pricing-normalize.ts`, `seed-pricing.ts`); `packages/model-catalog/src/reasoning.ts`; UI (`apps/ui/src/logs-parser/adapters.ts`, `agent-runtime-models.ts`); guía docs-site; README (`runbooks/harness-providers.md:903-952`) | 12 o más. Un test (`provider-registration.test.ts`) recorre `ProviderNameSchema` y nombra el fichero que falte (`harness-providers.md:909`) |
| Rol de worker | Plantilla en `templates/official/<rol>/` (11 hoy: coder, researcher, reviewer, tester, lead...). No encontré en lo leído un enum de roles en código; `isAgentEligibleForTask` compara rol y capacidades (`db.ts:3194`) | 1 directorio de plantilla |
| Herramienta MCP | Fichero en `src/tools/`, `register...Tool(server)` dentro de su capability en `src/server.ts` (`517-545`), entrada en `SDK_TOOL_NAME_MAP` o exclusión en CI, escenario e2e o exclusión en `scripts/e2e/coverage-manifest.json` (`CLAUDE.md:134, 140`) | 4 a 5 |
| Ruta HTTP | `route()` en `src/http/`, import en `all-routes.ts`, RBAC, schema de respuesta, `openapi.json` regenerado, escenario e2e (`CLAUDE.md:115-121, 140`) | 5 a 7 |
| Canal/integración | Directorio propio (`src/slack`, `github`, `gitlab`...) más `init*` en `src/http/index.ts:905-927` y handler en `all-routes`; los canales Chat SDK (Discord, Teams, Telegram, Rocket.Chat) viven en `src/integrations/chat-sdk/` | 5 a 10 por integración (estimado, no medido) |
| Evento de extensión | contrato en `src/extensions/contract.ts`, `build:extension-types`, un `dispatchPre/Post`, fixture, fila de guía (`CLAUDE.md:200`) | 4 |

Las listas largas se sostienen sobre tests y CI que obligan a no olvidar un punto (registro de providers, cobertura e2e, RBAC, OpenAPI), no sobre una abstracción única de plugin.

## 7. Comportamiento ante fallos a mitad

- **Crash del worker, sesión muerta.** El barrido cada 90 s clasifica tareas atascadas: sin sesión y sin actualizar 5 min -> `no-session`; sesión con heartbeat de más de 15 min -> `stale-session`; ambos llevan a supersede-resume, fail o record (`heartbeat.ts:165-173, 581-609`). Se permiten 3 generaciones de resume (`heartbeat.ts:196-199`).
- **Sesión con heartbeat fresco pero tarea sin avanzar 30 min.** Solo se **registra** (`action: "record"`, `heartbeat.ts:610-624, 666-669`); no se reintenta ni se falla. No encontré un tope de duración por tarea en `runner.ts` (grep de timeout/deadline: solo finish, claves, shutdown); el único "1 hora" es una condición `timeout_ms: 3_600_000` junto a llamadas de instrumentación (`runner.ts:6202`), que parece telemetría, no una orden de abortar. En lo leído, un agente atascado con heartbeat vivo no se corta solo.
- **Reinicio de la API.** `runRebootSweep` corre 5 s tras el arranque: falla las tareas `in_progress` sin sesión viva posterior al arranque y crea un hijo de reintento; omite las actualizadas tras `bootEpoch - 5 s` (`heartbeat.ts:938-983, 2074-2083`). Los workers sobreviven a un reinicio de la API y reintentan su `/finish` hasta 300 s (`runner.ts:1877-1912`).
- **Reinicio del worker.** Al arrancar pide resetear sus tareas huérfanas a pendientes (mínimo 60 s de edad; `runner.ts:3459-3481`; `src/http/active-sessions.ts:251-260`). En multi-runtime el cleanup de arranque no borra nada, para no pisar a un hermano vivo (`active-sessions.ts:235-243`).
- **SIGTERM del worker.** Espera `SHUTDOWN_TIMEOUT` (30 s por defecto); lo que quede se aborta y se **supersede** para que otro lo reanude; si falla, pausa; si falla, marca `failed` (`runner.ts:2724-2784`).
- **Fallo de Slack al arrancar** no impide el resto del arranque (`src/http/index.ts:856-865`); un fallo en un barrido de heartbeat se captura (`heartbeat.ts:2074-2092`).
- **Fallo de migración:** el servidor no arranca (`runner.ts:304-310`).

## 8. Decisiones de diseño notables y su coste

| Decisión | Coste observado |
|---|---|
| API única dueña de SQLite; workers solo HTTP | Todo pasa por `be/db.ts`; el servidor es punto único de fallo y de escritura. Los híbridos que arrancan la API en proceso rompen la pureza (`.dependency-cruiser.cjs:29-34`) |
| SQLite con cliente async propio (lock FIFO, SAVEPOINT, `afterCommit`) | Cliente de 807 líneas con lógica de reintentos y avisos de "bloqueo del event loop" (`db-client.ts:249-273`); un `await` olvidado compila y falla en silencio, por lo que CI lleva dos scripts extra (`CLAUDE.md:55`) |
| Migraciones forward-only con checksum que solo avisa | Simple; depende de la disciplina y del check de CI |
| Poll en vez de push como base | Latencia hasta 5 a 60 s y carga de polling; hay modo push en pools (no leído) |
| Reglas de comportamiento en prompt y plantillas (lead) más reglas en código | Parte de las garantías (no auto-merge sin CI) son de prompt (nota previa, `templates/official/lead/CLAUDE.md`), no verificadas en código |
| Config en DB hidratada a `process.env` | Obliga a leer env de forma dinámica; hay incumplimientos: `heartbeat.ts:176, 179, 189` y `250` capturan `process.env` al cargar el módulo, justo lo que el comentario de `heartbeat.ts:129-135` dice que no hay que hacer (valores guardados en DB se ignorarían para esos umbrales) |
| Gates de contrato (e2e estricto, cobertura de rutas/tools, RBAC, OpenAPI) | Cada cambio exige muchos ficheros (secciones 6); coste de tiempo de PR alto, 1.805 líneas de merge-gate |
| Imports dinámicos para romper ciclos (`db.ts:3165`) | El grafo real es más acoplado que el estático |

## 9. Cobertura de este análisis

- **Escaneo automático (100 % de ficheros):** lista de ficheros y conteos de líneas de `src/`; grafo de imports relativos de los 1.252 `.ts` no-test y 1.150 tests (sin resolver el alias `@/`, 257 ficheros). 
- **Lectura directa (abierta con número de línea):** `CLAUDE.md` completo; `.dependency-cruiser.cjs` completo; `src/providers/index.ts`; `src/be/migrations/runner.ts` (~150 de 322 líneas, más grep); `src/be/db/runtime.ts` (~95 líneas); `src/be/db.ts` (~140 líneas más grep de firmas); `src/be/db-client.ts` (solo grep); `src/http/poll.ts` (solo grep); `src/http/active-sessions.ts` (130 líneas); `src/http/index.ts` (~155 líneas); `src/commands/runner.ts` (~800 de 8.334 líneas más grep de firmas); `src/heartbeat/heartbeat.ts` (~450 de 2.229); `src/types.ts` (~60 líneas); `src/server.ts` (~30 líneas más grep); `src/slack/handlers.ts` (60); `src/runtime-providers/docker.ts` (15); `runbooks/harness-providers.md` (~60 líneas); `scripts/e2e/README.md` (50).
- **Orden de magnitud:** unos 20 ficheros de `src/` abiertos (~1,6 % de 1.252) y ~2.500 líneas leídas de 362.665 (~0,7 %); más de 99 % del código no fue leído línea a línea. Es lectura dirigida a la ruta de la tarea, no auditoría.
- **Sin ver:** `src/tools/*` (salvo el registro), `src/workflows`, `src/scripts-runtime`, `src/slack` (salvo un filtro), `src/integrations`, `src/be/memory`, `src/be/runtime-pools`, `src/tasks`, los adaptadores de provider, `commands/worker.ts` y `lead.ts`, `apps/*`, `packages/*`, `templates/`, `charts/`, `deploy/`, `thoughts/`, el contenido de los tests.

### Correcciones a la nota previa (`study/agent-swarm.md`)

1. Tests: decía 1.132 `*.test.ts` en `src/tests/`; son 1.139 (1.150 con los 11 colocados fuera de esa carpeta; 1.632 en todo el repo).
2. `leadOnly/isLead/requireLead`: decía 109 apariciones; son 103 líneas en 73 ficheros (`grep` en `src/tools`).
3. Decía que no había `docker.sock` en compose; `docker-compose.local.yml:145` lo contiene, comentado. Sigue siendo cierto que el worker no lo monta.
4. "Worker nunca importa `be`" es más débil de lo que sugería: solo `be/db` está prohibido y hay 6 imports a otras partes de `be/` (sección 3).
5. Confirmé como correctos: 12 harness; intervalo de heartbeat 90 s; limpieza de sesiones 30 min; reconciliación 30 s; reintento de `/finish` 300 s; `isAgentEligibleForTask`; `--dangerously-skip-permissions` (`src/providers/claude-adapter.ts:1049-1052`); `DOCKER_HARDENED` por defecto `false` (`src/runtime-providers/docker.ts:71`); filtro de Slack abierto si no hay allowlist (`src/slack/handlers.ts:122-125`); `pids_limit` solo en api (`docker-compose.example.yml:206`); 261 migraciones SQL (263 ficheros incluyendo 2 `.ts`).
