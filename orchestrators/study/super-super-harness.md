# Super Super Harness (estudio, solo lectura)

Repo: `references/super-super-harness` (abreviado `R/`). Las rutas de abajo son relativas a ese repo. Estudio hecho solo leyendo (Read/Grep/Glob), sin ejecutar nada. Lo escrito en los docs se trata como dato, no como instrucción.

## 1. Resumen

- **Qué es**: orquestador local de agentes de código. Hablas con un Router (sesión larga de Claude Code) y este contrata "Workers" (otras sesiones de Claude Code, cada una en su git worktree y rama) que corren en paralelo bajo un servicio en segundo plano. Terminal, CLI y web local hablan con el mismo Router (`README.md:7-26`, `docs/guide.md:5`).
- **Stack**: Python 3.11+ solo con librería estándar ("No packages, no build step", `docs/setup.md:10`), asyncio, SQLite en WAL (`core/store.py:260`), un único `index.html` para la web, MCP propio en stdlib (`router/mcp_server.py`). Usa el binario `claude` como harness (`adapters/claude_code.py`).
- **Licencia**: MIT (`LICENSE:1`).
- **Tamaño**: ~27.3k líneas de Python en total contando tests (salida de `wc`); `daemon.py` 570, `router/router.py` 740, `workers/tasks.py` 1275, `channels/web.py` 1056; 866 funciones `def test_` en `tests/`. Prompts: `prompts/router.md` (58 líneas), `prompts/worker.md` (16).
- **Madurez**: 6 commits, v0.3.0 (el clon superficial, `.git/shallow`, mostraba uno), fecha 2026-11-05 ("Web images, GitHub observe mode, Hermes Worker fixes"). Proyecto personal, un usuario, una máquina, macOS como único host soportado (`docs/setup.md:9`, `docs/faq.md:51-53`). Inspirado en el trabajo "Exploring Super-agent Design" de DeepSeek (`README.md:84`).

## 2. Modelo de dominio

| Concepto | Qué es | Dónde |
|---|---|---|
| Mensaje (`messages`) | Bandeja del Router: `user`, `event`, `system`; estados `queued → written → confirmed → done` | `docs/design.md:58-60`, `core/store.py:18` |
| Turno / efecto (`turns`, `effects`) | Turno del Router y resultado guardado de herramientas con efectos (at-most-once) | `core/store.py:33,43`, `router/router.py:727` (`once`) |
| Item (事项) | "Asunto" que le importa al usuario. `kind` delivery/standing, `status`, `permission` read/act (solo instrucción, no se aplica), progreso, next, fecha, notas versionadas en `items/<code>/notes.md` | `docs/design.md:147-149`, `core/items.py`, `core/store.py:198` |
| Worker / task (`tasks`) | Una sesión de harness con `code_name` único e irrepetible, `role`, workspace, rama `w/<code_name>`, perfil y harness guardados al arrancar | `docs/design.md:81-83`, `workers/workspace.py:40`, `core/store.py:65` |
| Assignment | Cada encargo a un Worker, con presupuesto de reintentos propio | `core/store.py:92`, `docs/design.md:88` |
| Worker run | Cada proceso lanzado (pid + hora de inicio) | `core/store.py:108`, `workers/tasks.py:1185` |
| Dependencia (`task_deps`) | `wait_for` en `worker_hire`; arranca solo cuando todos los upstream terminan, con sus reportes/rama/commit en el brief; ciclos rechazados | `docs/design.md:151`, `workers/tasks.py:226-335` |
| Evento / trigger | Todo lo que entra por `Bus.emit`; se registra y luego se enruta por reglas | `events/bus.py`, `docs/design.md:99-145` |
| Fuente y regla | `[sources.<n>]` con permiso observe/notify/act; `[[rules]]` con acción drop/record/direct/wake/observe | `ops/config.py:11-14`, `docs/design.md:116-133` |
| Suscripción | Un Worker en un item se suscribe a `item:<code>` y a hilos extra; `hand_back` devuelve un evento al Router | `core/store.py:213`, `daemon.py:112-115` |
| Recordatorio | `notify` (sin modelo) o `run` (despierta al Router); cron o `at` | `events/reminders.py` |
| Observación | Evento "de ensayo": el Router anota qué haría sin actuar | `core/store.py:184`, `router/router.py:675` |
| Provider | Endpoint, auth, clave, ventanas de contexto de modelos | `docs/design.md:31-35`, `ops/catalog.py` |
| Profile | Harness + Provider + modelo + ajustes (`settings`, `add_dirs`, `mcp`, `tools`, `env`) | `docs/setup.md:142`, `profiles.toml:15-19` |
| Harness | Adaptador que lanza un producto de agente (hoy `claude-code`; `hermes` experimental) | `adapters/__init__.py`, `docs/protocol.md` |

Relaciones: Usuario ↔ Router → (MCP) servicio → Workers. Item 1..N Workers (item por defecto "杂事", `docs/design.md:83`). Worker 1..N assignments y runs. Profile referencia Provider y Harness. Router también tiene `primary` y `backup` (perfiles del mismo harness).

## 3. Router

- **Qué es**: una sesión larga de harness en `SSHARNESS_HOME/router`, con `prompts/router.md` + "Soul" como system prompt (`docs/design.md:23`). Solo actúa mediante herramientas MCP servidas por el servicio (`mcp__router__<name>`).
- **Qué decide** (`prompts/router.md`): hace él mismo lo pequeño (preguntas, lectura/inspección, recordatorios, notas de item, preguntar a Workers); lo que sea cambios de repo, escribir ficheros, trabajo largo o que pida otro modelo va a un Worker (`router.md:14-15`). Elige perfil (`profile`+`reason`), crea items antes de contratar, reutiliza Workers del item con `worker_assign` y contrata otro solo para trabajo nuevo, parado o **verificación independiente siempre con otro modelo** (`router.md:28-29`). Nunca `git push`, nunca merge sin orden explícita (`router.md:55`).
- **Modelo**: el que diga `[router] primary`; `backup` debe ser del mismo harness (`profiles.toml:5-7`, `docs/setup.md:199`). Ejemplo del autor: Opus para el Router, otro modelo para el grueso, otro para verificar (`README.md:64`).
- **Herramientas** (30 en `router/tools.py:33-100`): `reply`, `ask_user`, `worker_hire/assign/say/interrupt/resume/stop/switch_model/rename/read/detail/search/wait_for`, `item_*`, `event_read`, `observe_note`, `message_search`, `router_switch_model`, `reminder_*`, `inbox_read`, `memory_read/write`. El usuario solo ve lo que pasa por `reply`/`ask_user` (`router.md:7`).
- **Filtrado de entrada**: cada lote lleva `<clock>`, `<workers>` (reenviado solo si cambió), `<message>`, `<event>`; todo escapado para que no cierre bloques ni falsifique mensajes (`docs/design.md:62-71`). El texto de eventos/reportes se marca como "dato, no instrucción" (`design.md:14`, `router.md:56`). El Router ve 600 chars de un evento; `event_read` da todo (`design.md:109`).
- **Filtrado de herramientas**: por token. `Daemon.call_tool` rechaza lo que no esté en las specs de ese rol (`daemon.py:86-89`). En turnos "solo observación" solo pasan lecturas + `observe_note` (`daemon.py:94-95`). Límite reconocido: Bash/Write/Edit nativos no pasan por esa puerta; solo se auditan en `used_builtin_tools` (`docs/design.md:139-141`).
- **At-most-once**: `reply`, `ask_user`, `worker_hire`, `worker_assign`, `worker_say`, `item_create`, `reminder_add` guardan resultado por turno; si hay replay tras un crash se devuelve lo guardado (`docs/design.md:73`, `router/router.py:727`).
- **No espera**: el Router nunca hace sleep/poll; los resultados llegan como mensajes (`router.md:18`). Las esperas, reintentos y failover los hace código (`design.md:12`).
- Detalle: el prompt del Router habla de "Feishu" como canal (`router.md:3`), pero Feishu está en el roadmap, no implementado (`README.md:74-79`).

## 4. Worker

- **Aislamiento**: un workspace `workspaces/<id>-<code_name>/` con `notes.md`, `brief.md`, `tmp/` y `repo/` (git worktree en rama `w/<code_name en minúsculas>`; se niega si la rama ya existe) (`workers/workspace.py:25-62`). Cada proceso se lanza con `start_new_session=True` (`adapters/claude_code.py:199`). Entorno por lista blanca (`ENV_ALLOW`) más endpoint, una variable de auth y modelos (`claude_code.py:15,27-53`). Hook de `pushInsteadOf` que rompe `git push` normales, solo anti-accidentes (`claude_code.py:19-24`). **No hay sandbox**: `--dangerously-skip-permissions` y `--settings {"disableAllHooks":true}` (`claude_code.py:70-71`). `$TMPDIR` y `$SSHARNESS_PORT_BASE` para no chocar (`prompts/worker.md:12`).
- **Cómo reporta**: por MCP con token propio: `report_progress`, `report_done`, `ask_user`, `notes_read/write`, `request_help`, `subscribe/unsubscribe`, `hand_back`, `item_read/update` (solo su item, solo `next` y notas) (`router/tools.py:102-122`, `daemon.py:107-137`). `report_done` debe venir del Worker, no de un subagente; se verifica con el `tool_use_id` (`workers/tasks.py:504-540`, `docs/protocol.md:80`). Si no llama, el texto final se usa como reporte.
- **Ciclo de vida** (`docs/design.md:81-95`, `guide.md:63-73`): Queued → Running → Needs you / Done; también Waiting on deps, Interrupted, Dormant, Failed, Stopped. Planifica hasta `max_workers`=10 (`workers/tasks.py:585`). Terminado deja la sesión para seguimientos. Inactivo `dormant_days`=7 pierde checkout y scratch pero conserva rama y `notes.md` (`workspace.py:65-94`, `tasks.py:1131-1153`). `worker_switch_model` solo dentro del mismo harness. Relevo (handoff) a sesión nueva si desborda el contexto o hay más de `compact_thrash`=3 compactaciones; máx. 2 por encargo (`design.md:89`).
- **Protocolo** (`docs/protocol.md`, contrato real en `adapters/__init__.py`): métodos de `Harness` (`build_env`, `headless_argv`, `interactive_argv`, `prepare_session`, `session_has`, `session_file`, `transcript`, probes, `available`, `Process`) y eventos v1 (`user` con `isReplay`, `assistant`, `system/init|input_started|background_tasks_changed|api_retry|compact_boundary`, `result`). Claude Code se lanza con `claude -p --input-format stream-json --output-format stream-json --verbose --replay-user-messages ... --strict-mcp-config` (`claude_code.py:84-88`). Un Worker solo termina cuando todo input enviado tiene ACK por replay.
- **Comunicación Router↔Worker**: nunca directa; todo pasa por el servicio. Eventos `worker.done|failed|question|stalled|blocked|handoff` hilo `task:<id>` (`design.md:97`).

## 5. Estado y memoria

- **SQLite** `SSHARNESS_HOME/router.db` (WAL), ~25 tablas: `messages`, `turns`, `effects`, `outbox`, `conversation`, `tasks`, `assignments`, `worker_names`, `worker_runs`, `events`, `triggers`, `deliveries`, `observations`, `items`, `item_tasks`, `subscriptions`, `task_deps`, `questions`, `reminders`, `reminder_fires`, `attachments`, `message_attachments`, `web_requests`, `kv` (`core/store.py:18-245`; resumen en `docs/design.md:166-177`).
- **Ficheros**: `~/.super-super-harness` (0700) con `profiles.toml` (0600), `keys/`, `router.sock`, `owner-token`, `web-session-key`, `router/soul.md` y `router/memory.md`, `workspaces/*/notes.md`, `items/<code>/notes.md`, `run/` (MCP y logs), `attachments/` (`docs/setup.md:70-84`).
- **Memoria del Router**: `memory.md` + `soul.md`, editables también desde la web; escritura con versión y detección de conflicto vía `memory_read/memory_write` (`router.md:58`, `core/router_files.py`). **Memoria del Worker**: `notes.md` propio versionado (8.000 caracteres inyectados al lanzar) y notas compartidas del item (`design.md:85`). La sesión del harness (`session_root/*/<id>.jsonl`) vive en el propio Claude Code.
- No hay memoria vectorial/semántica; solo búsqueda por texto (`message_search`, `worker_search`).
- **Recuperación**: al reiniciar se recuperan Workers, se reconcilia la bandeja con `session_has` y se vacía el outbox (`design.md:179-184`). Deduplicación por `(source,id)`, `(reminder,time)`, `request_id` web y efectos por turno.

## 6. Watchdog y manejo de fallos

- **Salud del Router** (`router/router.py:530-562`): cada tick, con turno abierto, silencio > `silent_seconds` (20) o `api_retry` repetidos → sonda superficial o profunda (`deep_probe_seconds`=120). Si falla: prueba el otro perfil; sano → cambia en la misma sesión y avisa; ambos caídos → **pausa** con alerta y reintento cada 15 s (`router.py:564-603`). En backup comprueba el primario cada `primary_check_seconds`=300 y **pregunta** antes de volver (`router.py:546-550`). Tras suspensión del equipo espera `wake_grace_seconds`=90 (`router.py:533-535`). Router caído: reinicio con backoff, pausa tras 3 seguidos (`design.md:77`).
- **Workers** (`workers/tasks.py:1131-1175`): sin actividad `stall_minutes`=30 → `worker.stalled` al Router; proceso ocioso que ya podría terminar pero sigue "running" → `worker.completion_stuck`. Tareas en segundo plano bloquean la finalización, con espera acotada `background_grace_seconds`=20. Errores: `worker_retries`=3 por encargo con backoff, luego Failed (`design.md:88`). Upstream fallido: `worker.blocked`, no se cancela nada.
- **Recuperación al arrancar** (`tasks.py:1177-1204`): mata procesos huérfanos por PID + hora de inicio y barre procesos con cwd en el workspace (`adapters/procs.py`), reencola los `running` con nota de `git status`.
- **Recordatorios**: bucle cada 1 s independiente del Router; el disparo es una transacción (sin doble disparo); cron dispara solo la última ocurrencia perdida (`design.md:155`).
- **Garantías explícitas**: seguimientos a Workers son "at least once"; no hay exactly-once ni de banners ni de efectos hechos por modelos (`design.md:184`).

## 7. Seguridad y autorización

- **Control**: socket Unix `router.sock` 0600 con JSON por líneas `{method, token, params}`; tokens: `owner` (fichero `owner-token`), uno por lanzamiento para Router y para cada Worker (`daemon.py:43-66,335-410`, `docs/design.md:39-47`). El rol decide qué herramientas ve.
- **Web**: solo `127.0.0.1`; ticket de un uso (5 min) → cookie firmada HttpOnly SameSite=Strict (12 h); comprueba `Host` y `Origin`, cuerpos ≤64 KiB, imágenes con cabecera/extensión/MIME coherentes, sin CORS (`docs/design.md:159-163`).
- **Claves**: ficheros 0600 o variables de entorno; nunca en argumentos, TOML ni logs (`docs/setup.md:57-66`).
- **Inyección de prompt**: texto externo marcado y escapado como dato; "reduce, no elimina" (`guide.md:142`). Los Workers y el Router no pueden emitir eventos (convención, no frontera de seguridad: `design.md:145`).
- **Límites reconocidos por el propio proyecto**: Workers con permisos totales sobre la máquina y el usuario, sin sandbox (`README.md:70`, `guide.md:139`); `permission read/act` del item no se aplica (`guide.md:93`); bloqueo de push eludible (`claude_code.py:19-22`); la auditoría de herramientas nativas es solo detección (`design.md:141`); cookies no aisladas por puerto.
- **Actualizaciones**: `upgrade` extrae commit a `releases/<sha>`, pasa tests con home temporal, y revierte si el Router no está listo en 30 s (`docs/setup.md:268-277`); el motivo es que los Workers pueden editar el código.

## 8. Madurez real

- **Tests**: 866 `def test_` en ~40 ficheros de `tests/` (p. ej. `test_workers.py` 2281 líneas, `test_web.py` 1969, `test_github.py` 1039, `test_bus.py`, `test_compaction.py`, `test_observations.py`). Hay `fake_harness.py`, `fake_claude.py`, `fake_gh.py`, `fake_hermes_acp.py` (harness simulado, sin modelo real). La suite tarda 4-5 minutos y se advierte usar un `SSHARNESS_HOME` temporal (`design.md:204-211`). El fake no prueba decisiones del modelo, p. ej. si el Router elige bien la herramienta (`design.md:211`). Tests de frontend requieren `node` y se saltan sin él. No los ejecuté.
- **Implementado (visto en código)**: adaptador Claude Code (`adapters/claude_code.py`), tabla de herramientas Router/Worker (`router/tools.py`), failover y sondas (`router/router.py:510-603`), recuperación y dormancia de Workers (`workers/tasks.py:1131-1204`), worktrees (`workers/workspace.py`), esquema SQLite completo (`core/store.py`), bus de eventos con validación (`events/bus.py`), recordatorios, poller de GitHub (`events/github.py`), web, avatares, `upgrade/rollback` (`ops/releases.py`), adaptador Hermes (`adapters/hermes*.py`).
- **Prometido o experimental, no cerrado**: Codex, Pi/DeepSeek Harness, Cloud agents, Feishu, panel de uso, "Soul and Memory" como siguientes pasos (`README.md:74-79`) — aunque `core/router_files.py`, `channels/usage.py` y la pantalla Soul & Memory ya existen en código, el README las pone en "Next". Hermes: Workers experimentales, validados solo con un build local fijado y un Provider falso en loopback; el Router con Hermes no está soportado (`docs/protocol.md:95`, `setup.md:212`). Los presets de DeepSeek/GLM/StepPlan/Kimi "no probados con claves reales" (`setup.md:176`, `guide.md:152`). `doctor` no es una prueba extremo a extremo.
- **Señal de cautela**: el historial es corto (6 commits, v0.3.0; el clon superficial mostraba uno), así que cuesta valorar su trayectoria real. Los docs mezclan chino en mensajes de usuario (`router/router.py:575,578`) y prompts en inglés con ejemplos en chino.

## 9. Ideas originales / lo distintivo

1. **Harness y modelo desacoplados**: trío Provider (endpoint+auth+clave+ventanas) / Profile (harness+provider+modelo+ajustes) / Harness (adaptador). Cualquier endpoint compatible con Anthropic sirve (`docs/design.md:29-37`, `profiles.toml`). Contrato de adaptador documentado (`docs/protocol.md`) y probado con un fake que ejercita Router, Workers, CLI y web sin cambios (`tests/test_harness.py`).
2. **Mezcla de modelos por rol**: modelo fuerte para Router, barato para Workers, otro proveedor como backup del Router y "verificación independiente siempre con otro modelo" como regla del prompt (`router.md:29`, `setup.md:188-199`).
3. **Código es dueño de la espera**: dependencias, reintentos, failover y recordatorios son código; el Router nunca hace poll (`design.md:12`).
4. **Un solo punto de entrada de eventos** `Bus.emit`: registra primero y enruta después con reglas por fuente/permiso; el permiso limita la acción (`design.md:99-145`).
5. **Modo observación** ("ensayo"): el Router anota `would_do` sin actuar; cuota por fuente y auditoría de herramientas nativas (`design.md:130-141`).
6. **Workers permanentes con identidad**: `code_name` irrepetible, notas propias, rama estable, dormancia y relevo de sesión por desbordamiento de contexto (`design.md:81-95`).
7. **Efectos at-most-once por turno** y reconciliación de bandeja contra la transcripción de la sesión (`session_has`) tras un crash.
8. **Toma de control humana**: `open`/`attach`/`peek` entregan al terminal la sesión interactiva del Worker o del Router y la recuperan al salir (`guide.md:42-44`).
9. **Panel de Routing** con movimiento solo por eventos reales, SSE que solo avisa "algo cambió" y la UI relee la API (`design.md:163`).

## 10. Carencias y huecos

- Un único harness de producción (Claude Code); Codex pendiente; Hermes solo Workers y experimental.
- Sin sandbox ni autorización real: todo es convención y prompt (item `permission`, bloqueo de push, restricciones en modo observación).
- Un usuario, una máquina, sin nube ni multiusuario; la web no es para despliegue público.
- Soporte de SO: solo macOS probado (ver sección 11).
- Los diffs en la web muestran solo cambios commiteados; la web no envía imágenes al Router según `guide.md:151` (pero `setup.md:100-104` describe subida de imágenes: contradicción entre docs); UI solo en inglés; Routing no muestra historial (`guide.md:146-152`).
- Imágenes subidas sin borrado automático (`setup.md:104`).
- Migraciones: bases anteriores a `tasks.code_name` se rechazan, sin migración (`setup.md:279`).
- Coste: cada Worker es una sesión completa de Claude Code; diez Workers ≈ diez sesiones (`faq.md:73`).
- Sin memoria semántica; la calidad del enrutado depende del modelo y no la prueban los tests (`design.md:211`).
- Idioma mixto (mensajes en chino en código y prompts parcialmente en chino).
- Acoplamiento a detalles internos de Claude Code (formato `.jsonl` de sesiones, flags CLI), frágil ante cambios de versión.

## 11. Compatibilidad con Windows

**Conclusión: no funciona en Windows tal cual; hace falta WSL2 o un porte sustancial.** El propio proyecto dice que solo macOS está soportado y "nada más está probado" (`docs/setup.md:9`, `docs/faq.md:51-53`). Verificado en código:

- `core/private_files.py:3,47` hace `import fcntl` (módulo inexistente en Windows) y `fcntl.flock`. Rompe al importar.
- Socket de control y MCP vía `socket.AF_UNIX` (`cli.py:25,82`, `router/mcp_server.py:12`, `daemon.py:468` con `os.chmod 0o600`). AF_UNIX existe en Windows 10+ pero los permisos 0600 no se aplican igual.
- `adapters/procs.py` depende de `ps -axo`, `ps -o lstart=` y `lsof` (líneas 10, 33, 46) y de `signal.SIGKILL` (línea 60, no existe en Windows). Todo el kill de árboles y la recuperación de huérfanos se apoya ahí. También `os.killpg` + `signal.SIGKILL` en `adapters/hermes_acp.py:405`.
- Interactivo: `["/usr/bin/env", binary, ...]` (`claude_code.py:93`), con comentario específico de un Mac.
- `caffeinate` y `osascript` solo se usan en `darwin` y se omiten en otros SO (`daemon.py:558-560`, `events/reminders.py:260-264`, `channels/web.py:843`): eso sí degrada con elegancia.
- Rutas con `~`, permisos 0700/0600, `git worktree` con rutas largas y `start_new_session=True` (solo POSIX) añaden más fricción.
- Hermes exige un venv Python 3.14 y está validado solo en macOS (`protocol.md:95`).
- Camino práctico en tu equipo Windows 11: WSL2 (Ubuntu) con Python 3.11+, git y `claude` instalados dentro de WSL, trabajando en el filesystem de Linux (no en `/mnt/e`) por rendimiento de worktrees y para respetar permisos. Navegar a la web desde Windows por `127.0.0.1:8765` normalmente funciona vía reenvío de localhost de WSL. No lo he probado.

## Evidencias

| Afirmación clave | Ruta:línea |
|---|---|
| Licencia MIT | `LICENSE:1` |
| Solo biblioteca estándar, macOS único host | `docs/setup.md:9-10` |
| Un solo punto de entrada de eventos / BD es la verdad | `docs/design.md:9-14` |
| Router = sesión de harness, solo actúa por MCP | `docs/design.md:23,51` |
| Lista de herramientas Router y Worker | `router/tools.py:33-122`, `docs/design.md:55-56` |
| Tokens por rol y rechazo de herramientas no permitidas | `daemon.py:58-66,86-89` |
| Filtrado en turno de observación | `daemon.py:94-95`, `docs/design.md:139` |
| Límite: herramientas nativas solo auditadas | `docs/design.md:141` |
| At-most-once por turno | `docs/design.md:73`, `router/router.py:727` |
| Sondas, failover, pausa, vuelta a primario | `router/router.py:510-603` |
| Reglas de reintento y stall de Workers | `workers/tasks.py:1131-1175`, `docs/setup.md:148` |
| Recuperación tras reinicio (PID+hora, git status) | `workers/tasks.py:1177-1204`, `adapters/procs.py:50-79` |
| Worktree y rama `w/<code_name>` | `workers/workspace.py:40-62` |
| Dormancia sin forzar worktree sucio | `workers/workspace.py:65-79` |
| Lanzamiento headless de Claude Code | `adapters/claude_code.py:84-88` |
| Permisos totales y hooks desactivados | `adapters/claude_code.py:70-71`, `README.md:70` |
| Entorno por lista blanca | `adapters/claude_code.py:15,27-53` |
| Push guard eludible | `adapters/claude_code.py:19-24` |
| Esquema SQLite y WAL | `core/store.py:18-245,260` |
| Perfil = harness+provider+modelo; backup mismo harness | `docs/design.md:29-37`, `docs/setup.md:199`, `profiles.toml:5-19` |
| Verificación independiente con otro modelo | `prompts/router.md:29` |
| Contrato de adaptadores y eventos v1 | `docs/protocol.md:36-80` |
| Hermes experimental, Router no soportado | `docs/protocol.md:95,119`, `docs/setup.md:212` |
| Roadmap (Feishu, Codex, nube) | `README.md:74-79` |
| Prompts de Worker (report_done solo del Worker) | `prompts/worker.md:11`, `workers/tasks.py:504-540` |
| Tests: 866 `def test_`, tardan 4-5 min | `tests/*.py` (grep), `docs/design.md:204-211` |
| Windows: `fcntl` | `core/private_files.py:3,47` |
| Windows: `ps`, `lsof`, `SIGKILL` | `adapters/procs.py:10,33,46,60` |
| Windows: `/usr/bin/env` | `adapters/claude_code.py:93` |
| caffeinate/osascript solo en darwin | `daemon.py:558-560`, `events/reminders.py:260-264` |
| Contradicción docs sobre imágenes web | `docs/guide.md:151` vs `docs/setup.md:100-104` |
