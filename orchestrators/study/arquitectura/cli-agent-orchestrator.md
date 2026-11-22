# Arquitectura: awslabs/cli-agent-orchestrator (CAO)

Estudio de solo lectura. Rutas relativas a `references/cli-agent-orchestrator/` (abrev. `pkg/` = `src/cli_agent_orchestrator/`). Clon superficial de 1 commit, sin historial. No se ejecutó nada del repo. Esta nota corrige y amplía `study/cli-agent-orchestrator.md` (ver sección 9, "Correcciones").

## 1. Resumen y veredicto

- CAO es un servidor FastAPI local (`cao-server`) que lanza CLIs de código reales dentro de tmux (o herdr, experimental) y expone herramientas MCP (`handoff`, `assign`, `send_message`...) para que un agente supervisor delegue. Detalle de superficies: `CODEBASE.md:9-19`.
- Arquitectura en capas declarada (CLI/Web/MCP -> ruta FastAPI -> servicio -> backend -> proveedor -> proceso CLI, `CODEBASE.md:62-71`), pero el código real la cruza mucho: `api/main.py` tiene 12.562 líneas y ~123 rutas, y hace 24 imports de módulo hacia `services/` y 128 imports diferidos (dentro de funciones).
- El estado es SQLite (SQLAlchemy, 22 tablas) más ficheros bajo `CAO_HOME_DIR`; las migraciones son funciones `_migrate_*` escritas a mano y encadenadas en `init_db` (`pkg/clients/database.py:1354-1401`), sin Alembic.
- La robustez ante fallos es buena en el camino de un paso (`run_agent_step`: timeout, error, cancelación, error de proveedor, persistencia previa al teardown) y en el arranque (barridos de recuperación). Es débil en lo estructural: el "completado" se infiere leyendo la pantalla del TUI, y hay un fallo reconocido en el código (reutilizar un terminal puede cerrar con la finalización del turno anterior, `pkg/services/agent_step.py:826-831`).
- Veredicto: diseño con ideas buenas (callback garantizado, idempotencia de creación, fail-closed) pero con módulos "dios" y acoplamiento circular entre paquetes salvado con imports diferidos. Mantenible solo con mucha disciplina de tests (ver eje mantenibilidad).

## 2. Mapa de módulos y capas

Medido con `git ls-files 'src/cli_agent_orchestrator/<dir>/*.py' | xargs cat | wc -l`. Dependencias: calculadas con un script que lee todas las líneas `from cli_agent_orchestrator... import` de los 317 ficheros .py de `src/` y separa import de módulo (M) de import diferido (D, con sangría). Es análisis por paquete, no por módulo.

| Paquete | Responsabilidad | Ficheros .py / líneas | Depende de (M/D) |
|---|---|---|---|
| `api/` | HTTP, SSE, AG-UI, WebSocket PTY; lifespan y demonios | 2 / 12.563 | services 24/128, models 8/16, utils 9/9, decisions 5/7, backends 4/4, providers 4/0, graph 3/0, clients 1/5, agent_plugins 0/7, cli 2/1 |
| `services/` | sesiones, terminales, inbox, workflow, memoria, eventos, config | 109 / 67.651 | utils 58/9, models 57/7, providers 36/5, clients 34/53, constants 35/4, backends 14/8, plugins 7/0, decisions 2/7, agent_plugins 3/3 |
| `providers/` | adaptadores por CLI (estado, extracción, arranque) | 26 / 27.568 | utils 61/8, models 37/2, backends 14/2, services 12/31, agent_plugins 12/0, constants 10/3, clients 1/4 |
| `utils/` | perfiles, rutas, orquestación, mapeo de herramientas | 37 / 12.526 | constants 14/2, models 7/1, services 3/11, providers 0/4, backends 0/3 |
| `cli/` | comandos `cao` (Click) | 26 / 8.277 | utils 22/9, services 12/28, agent_plugins 10/1, constants 12/1 |
| `clients/` | SQLite (`database.py`), tmux (`tmux.py`), RPC herdr | 3 / 7.904 | constants 2/25, utils 5/0, models 4/0, backends 2/0, services 0/1 |
| `agent_plugins/` | Agent Plugins 1.0.0 (manifiestos, contención, instalación) | 12 / 6.316 | services 0/4, utils 1/4 |
| `mcp_server/` | herramientas MCP en sesión | 5 / 4.497 | services 7/15, utils 7/1, security 3/0, models 3/0 |
| `decisions/` | plataforma de decisiones de lanzamiento | 18 / 4.092 | models 6/0, services 6/2, utils 3/0, api 0/1 |
| `backends/` | abstracción de terminal: tmux, herdr, herdr-app | 8 / 3.781 | clients 2/6, models 3/0, services 1/0 |
| `models/` | modelos Pydantic | 18 / 2.627 | solo constants 3/0 |
| `plugins/` | API de plugins de eventos + integrados | 10 / 1.549 | services 10/21, backends 0/6, clients 1/3, mcp_server 0/1 |
| `graph/`, `ops_mcp_server/`, `security/`, `ext_apps/`, `telemetry/`, `skills/` | grafo, MCP operativo, auth, apps MCP, OTel, skills empaquetadas | 12/1.427, 5/1.299, 4/882, 4/584, 6/454, 2/626 | hacia services/utils/constants |
| `constants.py` (raíz) | constantes y rutas | 1 / 1.306 | models 1/0 |
| `src/cao_workflow/` | paquete aparte de runtime de workflows | 7 ficheros | no analizado |

Total `src/`: 317 ficheros .py, 166.505 líneas (`git ls-files 'src/*.py' | xargs cat | wc -l`). Fuera de `src/`: `web/` (React, 122 ficheros .ts/.tsx), `tui/` (Rust, 18 .rs, 27.454 líneas), `cao_mcp_apps/`, `examples/` (721 ficheros), `docs/`.

## 3. Fronteras y acoplamiento

- **Capas declaradas vs reales.** `CODEBASE.md:62-71` dice que el backend posee el ciclo de vida del terminal y el proveedor solo la lógica específica del CLI, "sin asumir un backend concreto". En código `services/terminal_service.py` llama a `get_backend()` (p. ej. líneas 3243, 3434, 6508, 7220), de acuerdo con el diseño. Pero ocho ficheros mencionan `tmux_client`/`clients.tmux` directamente, entre ellos `services/workflow_service.py` y `services/workflow_spec_service.py` además de `cli/commands/info.py` (`grep -rln 'clients.tmux import\|clients import tmux\|tmux_client' src`). No leí esos usos: no sé si cruzan la abstracción o solo consultan.
- **Ciclos entre paquetes (a nivel de paquete, medidos).** Hay aristas en ambos sentidos entre `services` y: `plugins` (services->plugins 7 M; plugins->services 10 M + 21 D), `providers` (36 M / 31 D), `utils` (58 M / 11 D), `clients` (34 M / 1 D), `decisions` (2 M / 0 vs 6 M / 2 D), `backends`. También `decisions`->`api` diferido y `plugins`->`mcp_server` diferido. No comprobé si son ciclos reales a nivel de módulo (los imports diferidos suelen evitarlos). Evidencia de que es una preocupación consciente: `backends/registry.py:1-6` dice que existe "para romper la cadena de imports circulares".
- **Imports diferidos como técnica de ruptura de ciclos:** `api/main.py` -> services: 24 de módulo y 128 diferidos; `services` -> `clients`: 34 de módulo y 53 diferidos; `clients` -> `constants`: 2 de módulo y 25 diferidos.
- **Módulos "dios"** (líneas): `api/main.py` 12.562 (289 `def`/`class` a nivel de módulo; ~123 decoradores de ruta; `grep -n '^def \|^async def \|^class '`), `services/terminal_service.py` 7.370, `services/memory_service.py` 5.251, `clients/database.py` 4.661 (22 tablas y 24 migraciones en un fichero), `providers/kimi_cli.py` 3.742, `mcp_server/server.py` 3.686, `utils/orchestration.py` 3.448 (un helper de "utils" que contiene `_handoff_impl`, `orchestration.py:2221`, y `_assign_impl`, `:2915`: lógica de dominio en un paquete utils). `lifespan` de `api/main.py` ocupa 1743-~2100.
- **Fuga de nombres de proveedor fuera de `providers/`:** `kimi_cli` aparece en 15 ficheros de `src`, `grok_cli` en 11, `hermes` en 8 (grep por nombre), incluidos `api/main.py`, `cli/commands/launch.py`, `utils/enforcement.py`, `agent_plugins/mcp_mapping.py`, `services/terminal_service.py`. Los proveedores no son plug-ins aislados.
- **Dentro del servidor, bus de eventos:** `status_monitor` consume `terminal.{id}.output` y publica `terminal.{id}.status` (`pkg/services/status_monitor.py:1-5`); inbox y callback de delegación consumen esos eventos (`api/main.py:1895-1898`).

## 4. Estado y persistencia

| Estado | Dónde | Quién escribe | Cita |
|---|---|---|---|
| Terminales, inbox, aristas de delegación, kept, decisiones, lanzamientos, efímeros, incarnaciones, memoria, vault, claves de idempotencia, resultados de handoff | SQLite, 22 `__tablename__` | `clients/database.py` (única capa de acceso declarada, `CODEBASE.md`), más SQL directo en otros módulos | `database.py:108-1035`; `grep -c __tablename__` = 22 |
| Conexión | singleton `create_engine(DATABASE_URL, connect_args={"check_same_thread": False})` | module-level | `database.py:1350` |
| Diario de workflows | SQLite vía `sqlite3` directo, con `busy_timeout` por conexión | `services/workflow_journal.py` | `workflow_journal.py:297-308` |
| Directorio y fichero DB | 0700 / 0600 | `_ensure_db_dir`, `_restrict_db_file_permissions` | `database.py:1332-1345, 1404-1420` |
| Estado de status de terminal | memoria del proceso + filas de evento, escritas por un único hilo | `status_monitor` | `status_monitor.py:36-43` |
| Estado "diferido" de init | memoria del proceso (marcas `_mark_deferred_init_external_owner_active`...) | `terminal_service` | `terminal_service.py:538-661` |

- **Migraciones.** `init_db` hace `create_all` y luego ejecuta en orden 24 funciones `_migrate_*` (`grep -c '^def _migrate_'` = 24), aditivas (columnas nulas, índices, reconstrucción de tabla para `source_kind`); un comentario exige "nunca reordenar las entradas de arriba" (`database.py:1374-1376`). No encontré tabla de versión de esquema ni Alembic en lo leído (`pyproject.toml` no lo declara). Consecuencia: el orden y la idempotencia de cada migración es una convención, no un mecanismo.
- **Transacciones/concurrencia.** El motor SQLAlchemy no fija WAL ni `busy_timeout` (`services/session_service.py:847`, comentario; `constants.py:991` dice que WAL queda deliberadamente fuera de alcance). El diario de workflows sí fija `busy_timeout` (`workflow_journal.py:297-308`). Una escritura de estado "puede esperar hasta 5 s por el bloqueo del driver" (`status_monitor.py:37-39`) y por eso se hace en un hilo propio.
- **Idempotencia de creación.** Tabla `idempotency_keys` con `key` como clave primaria y huella SHA-256 de 14 campos hasheados (`database.py:1307-1329`; `terminal_service.py:2090-2123`). El INSERT duplicado falla en commit (`database.py:1299-1304`).
- **Instancia única.** `cao-server` toma un bloqueo exclusivo por `CAO_HOME_DIR` antes de tocar DB o puerto y se niega a arrancar si otro lo posee (`api/main.py:1759-1774`).

## 5. Flujo de control de una tarea (handoff síncrono), de punta a punta

1. El supervisor (un CLI LLM) llama la herramienta MCP `handoff` (`pkg/mcp_server/server.py:339, 465`).
2. Se delega en `_handoff_impl` (`pkg/utils/orchestration.py:2221`), que hace un único `POST /terminals/run-step` (`orchestration.py:1760, 1808`).
3. La ruta `run_step` (`pkg/api/main.py:6216`) comprueba ámbito de escritura, fija recolectores de pasos de script, gestiona la rama de reproducción y finalmente llama `run_agent_step(...)` (`api/main.py:6590`). Si hay `job_id` registra `running` al inicio (docstring `api/main.py:6257-6266`).
4. `run_agent_step` (`pkg/services/agent_step.py:436`) llama `terminal_service.create_terminal(...)` (`agent_step.py:717`). Ahí: ID y ventana, sesión/ventana tmux, fila BD, `provider.initialize()`, pipe-pane de logs (`terminal_service.py:2004-2013`; `get_backend().pipe_pane` en 3434-3442; `provider_manager.create_provider` en 3481).
5. Se notifica el terminal para el barrido de huérfanos y se escribe la fila `running` ANTES de la espera de readiness (`agent_step.py:743-751`).
6. Espera de readiness con `wait_until_status`; si no llega, lanza `StepExecutionError(kind="timeout")` y no borra el terminal (`agent_step.py:753-765`).
7. Se envía el prompt con `terminal_service.send_input` en un hilo, usando la memoria congelada del run si existe (`agent_step.py:780-814`).
8. `_wait_for_completion` sondea `status_monitor.get_status`: COMPLETED sale ya; IDLE solo cuenta tras haber visto trabajo y `_IDLE_STABLE_POLLS` lecturas seguidas; ERROR lanza `kind="error"`; vencido el plazo, `kind="timeout"` (`agent_step.py:171-286`). Si no hay evidencia de recogida en `_PROMPT_PICKUP_GRACE`, reenvía hasta `_PROMPT_REDELIVER_MAX` veces (`agent_step.py:196-211`).
9. Extrae el último mensaje con `get_output(..., OutputMode.LAST)` (provider.extract_last_message_from_script) (`agent_step.py:849-859`) y lo clasifica contra errores de proveedor en banda (`:875-897`).
10. Persiste el resultado (`upsert_handoff_result`) ANTES de destruir el terminal (`agent_step.py:899-918`) y hace teardown (salida graciosa + `delete_terminal`), salvo "keep" (`:920-935, 945-995`).
11. La ruta devuelve 200 o error estructurado: `kind` timeout -> 504; error/provider_error -> 502 (`api/main.py:6237-6248`).

`assign` (asíncrono) y `send_message`: el servicio de aristas de delegación abre una arista y reenvía la última respuesta si el worker queda IDLE sin llamar `send_message` (`pkg/services/delegation_callback_service.py:1-8, 60-73`). La cadena `assign`/inbox a nivel de código (`orchestration.py:2915`, `inbox_service.py`) la leo solo de la nota previa: no reverificada aquí.

## 6. Puntos de extensión (cuántos ficheros se tocan)

- **Proveedor (harness).** La guía oficial (`pkg/skills/cao-provider/SKILL.md`, pasos 1-9) pide: (1) valor en `ProviderType` (`models/provider.py:4-21`); (2) clase en `providers/<x>.py` implementando 5 métodos abstractos de `BaseProvider`: `initialize`, `get_status`, `extract_last_message_from_script`, `exit_cli`, `cleanup` (`providers/base.py:198, 207, 463, 475, 484`); (3) registro en `providers/manager.py`; (4) `PROVIDERS_REQUIRING_WORKSPACE_ACCESS` en `cli/commands/launch.py`; (5) fila `PROVIDER_ENFORCEMENT` en `utils/enforcement.py` y, según el mecanismo, `TOOL_MAPPING` en `utils/tool_mapping.py`; (7-9) tests unitarios, fixtures, e2e y fichero de documentación (hay un test que falla si las tablas de docs no coinciden, SKILL.md paso 5). Mínimo realista: 5-6 ficheros de código + 2-3 de test + docs; los proveedores existentes aparecen en 8-15 ficheros de `src` (sección 3).
  - Observación (no desfase probado): el registro en `manager.py` existe dos veces, el diccionario `provider_class` (`manager.py:44-59`) y la cadena `if provider_type == ...` de `create_provider` (`manager.py:114`...). SKILL.md paso 3 solo menciona la cadena `elif`. Hay que mantener ambos sincronizados.
- **Backend de terminal.** `TerminalBackend` en `backends/base.py` + rama `elif` por nombre en `backends/factory.py:46-108` (sin registro dinámico). Además, el lifespan tiene ramas `isinstance(backend, HerdrBackend/HerdrAppBackend)` para el servicio de inbox (`api/main.py:1943-1965`). Un backend nuevo toca como mínimo base, factory y lifespan. Herdr y herdr-app se declaran "EXPERIMENTAL" (`factory.py:84, 100`).
- **Canal/hook de eventos.** Entry points de grupo `cao.plugins` (`pyproject.toml:137-142`) y `cao.deciders` (`:144-145`); contrato en `plugins/base.py`, `plugins/events.py` (`CODEBASE.md`, sección Plugins). Es la extensión más limpia: un paquete externo sin tocar el núcleo.
- **Rol / perfil.** Un fichero Markdown con frontmatter en `agent_store/` o instalado por usuario; no requiere código (nota previa, `models/agent_profile.py:328-371`, no reverificado en esta pasada).
- **Herramienta MCP.** Añadir función en `mcp_server/server.py` (3.686 líneas) y, si lleva autorización, tocar `_tool_denied_reason` (prior note: `server.py:1607-1665`, no reverificado).

## 7. Comportamiento ante fallos a mitad

| Situación | Comportamiento | Cita |
|---|---|---|
| Reinicio del servidor con terminales en init diferido | filas pendientes pasan a ERROR duradero antes de servir peticiones; reintento en tarea si el barrido fue incompleto | `api/main.py:1810-1832` |
| Reinicio con entrega de mensaje inicial pendiente | se marca fallida/interrumpida | `api/main.py:1815-1824` |
| Residuos efímeros | barrido al arrancar y demonio periódico | `api/main.py:1870-1875, 1933` |
| Segundo servidor sobre el mismo home | se niega a arrancar, mensaje con el proceso dueño | `api/main.py:1759-1774` |
| Terminal no llega a ready | `kind="timeout"`, NO lo borra, devuelve `terminal_id` | `agent_step.py:753-765` |
| Terminal en ERROR / plazo agotado | `kind="error"` / `kind="timeout"`; en timeout el terminal sigue vivo | `agent_step.py:241-286` |
| Cancelación | `StepCancelledError` rápido; teardown de lo creado por la llamada | `agent_step.py:234-236, 834-840` |
| Error del proveedor impreso en el pane | clasifica y borra el terminal creado; el paso nunca queda como completado reutilizable | `agent_step.py:875-897` |
| Crash durante teardown | resultado ya persistido antes | `agent_step.py:899-918` |
| Pérdida de la respuesta MCP | se recupera con `GET /handoff-results/{job_id}` | docstring `api/main.py:6257-6266` |
| Mensaje perdido al teclear | reentrega hasta `_PROMPT_REDELIVER_MAX`; nunca si el terminal no lee IDLE | `agent_step.py:196-211, 288-300` |
| Worker mudo | auto-forward de la última respuesta al caller | `delegation_callback_service.py:60-73` |
| Procesos de script muertos | barrido de huérfanos desmonta el terminal registrado | `api/main.py:6270-6273` |

Huecos comprobados: (a) reutilizar un terminal puede cerrar con la finalización del turno previo, reconocido en el código como "LIVE SIBLING of #735, not fixed here" (`agent_step.py:826-831`); (b) en timeout, un worker que "corre largo" queda vivo y huérfano si el llamador no lo limpia; (c) no encontré en lo leído transacción que cubra a la vez tmux y BD en `create_terminal`: el orden es ventana -> fila -> init (`terminal_service.py:2004-2013`), y la compensación (fila fantasma o ventana sin fila) la gestionan barridos de limpieza (`api/main.py:1878`, `cleanup_service.py` de la nota previa), no verificada por mí.

## 8. Decisiones de diseño notables y su coste

- **Estado por pantalla.** El estado se deduce con regex/pyte sobre el TUI (`providers/base.py:207`, `status_monitor.py`). Coste: fragilidad ante cada versión de CLI; el volumen de código de proveedores lo refleja (27.568 líneas, tests 57.961).
- **Un solo asiento para pasos (`run_agent_step`).** El motor de workflows lo llama en proceso y los clientes MCP por HTTP (`api/main.py:6223-6229`). Ganancia: una sola semántica; coste: `run_step` es una ruta de cientos de líneas con ramas de replay, huellas y fences de generación (`api/main.py:6216-6760`).
- **Fail-closed y ceiling de delegación** (nota previa; `security/`, 882 líneas): coste en complejidad de configuración.
- **Singleton de módulo** (`engine`, `provider_manager`, `get_backend()`, `bus`): simple, pero impide varias instancias en proceso y favorece el parcheo en tests (45 ficheros de test usan el mock provider).
- **Todo en un proceso con hilos y SQLite**: coste explícito: escribir en hilo dedicado para no bloquear el bucle (`status_monitor.py:36-43`), `asyncio.to_thread` por todas partes (`agent_step.py:780-850`).
- **Documentación y comentarios como contrato:** docstrings de varios centenares de líneas (p. ej. fingerprint de idempotencia, `terminal_service.py:2090-2200`), 19.305 líneas de comentario `#` en `src`. Ayuda a entender; coste de ruido y de desfase.
- **Prompt como control** en lugar de política determinista (nota previa).

## 9. Cobertura de esta lectura y correcciones

Lectura directa (abrí y leí con número de línea): `api/main.py` 1743-2000 y 6216-6330 (más greps), `services/agent_step.py` 171-300 y 700-995, `services/terminal_service.py` 1973-2200 (más greps de 142 definiciones), `clients/database.py` 1290-1420 (más greps), `providers/manager.py` 1-120, `providers/base.py` (índice de firmas), `backends/registry.py` y `factory.py` completos, `models/provider.py` completo, `utils/sqlite_connection.py` completo, `services/delegation_callback_service.py` 1-75, `services/status_monitor.py` 1-50, `CODEBASE.md`, `skills/cao-provider/SKILL.md` (1-140), `pyproject.toml` 1-330.
Recorrido automático (script, no lectura): imports de los 317 ficheros de `src/`; recuentos de ficheros y líneas.
Cobertura aproximada: ~14 ficheros de `src/` abiertos parcialmente de 317 (~4 %); ~1.900 líneas leídas de 166.505 (~1 %), más estructura de todos por grep. Es una lectura dirigida a ruta crítica, no una auditoría.
Sin ver: `inbox_service`, `tmux.py` (2.965 líneas), `orchestration.py` (leído solo vía la nota previa), `providers/*` salvo base y manager, `memory_*`, `vault/`, `workflow_*`, `decisions/`, `security/auth.py`, `web/`, `tui/`, `cao_workflow/`, `examples/cao-clusters/kubernetes/eks/` (broker K8s). Las afirmaciones sobre esos puntos vienen de la nota previa y están marcadas "no reverificado".

Correcciones a la nota previa (`study/cli-agent-orchestrator.md`):
- Dice "3033 ficheros; 477 .py fuera de `test/`... el paquete solo suma ~115k líneas". Medido: 3.004 ficheros versionados (`git ls-files | wc -l`); `src/` tiene 317 .py y 166.505 líneas; `test/` 907 .py y 399.459 líneas (el 907 sí era correcto). El 477 y los ~115k eran erróneos.
- Dice "CI con ~16 workflows": confirmado, 16 ficheros en `.github/workflows/`.
- Dice "plataforma de decisiones solo leída a nivel de índice" y "Diario de workflows no leído en detalle": sigue igual en esta pasada.
- Añadido que no estaba: tabla de aristas entre paquetes, el fallo reconocido de reutilización de terminal (`agent_step.py:826-831`), las 24 migraciones manuales sin versión de esquema.
- Confirmado en código: 14 valores de `ProviderType` (13 reales + `mock_cli`; el de MiniMax tiene valor `mcode`, `models/provider.py:19`), registro de proveedores y timeouts de `kind`.
