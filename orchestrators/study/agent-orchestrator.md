# Estudio: agent-orchestrator (OrchestratorInc, antes ComposioHQ)

Estudio de solo lectura sobre `references/agent-orchestrator` (en adelante `AO`). Rutas relativas a esa carpeta salvo que se indique. Nada se ejecutó; AGENTS.md/CLAUDE.md se trataron como objeto de estudio.

## 1. Resumen

- **Qué es**: app de escritorio (Electron) + daemon local que supervisa una flota de agentes de código. Cada tarea = un agente, una rama y un git worktree. Un "orquestador" por proyecto (un agente más, `KindOrchestrator`, `backend/internal/domain/session.go:20-25`) planifica y lanza workers vía `ao spawn`/API. El daemon observa PRs y reenvía fallos de CI, comentarios de review y conflictos al worker dueño (README.md:52-56).
- **Stack (corrección importante)**: el encargo decía TypeScript, pero el estado actual es una **reescritura en Go** (daemon, CLI Cobra, SQLite+sqlc, chi) con frontend Electron+React 19 y móvil Expo (AGENTS.md:3,9-11; docs/architecture.md:3-6). AGENTS.md:3 habla de "la antigua implementación TypeScript" como cosa del pasado. Recuento aproximado de ficheros: 7978 totales, ~3106 `.go`, ~3319 `.ts/.tsx`, 1633 `*_test.go`, 1313 `*.test.ts(x)` (conteo por `find`).
- **Licencia verificada** leyendo `LICENSE`: Apache License 2.0 (texto estándar; línea 189 "Copyright 2026 Untrivial"). README.md:222-224 coincide. Nótese que el código y la STATUS aún referencian `Untrivial-ai/agent-orchestrator` y el módulo Go es `github.com/aoagents/agent-orchestrator` (rebranding en curso).
- **Tamaño**: muy grande; solo `backend/internal/` tiene >100 paquetes (listado parcial: lifecycle, observe, review, adapters/agent con ~40 harnesses, session_manager, service, storage/sqlite, cdc, httpd...).
- **Madurez**: producto real y comercial (releases firmadas, auto-update, telemetría PostHog, cloud, móvil). Pero es un producto de escritorio monousuario, no un framework embebible. `docs/STATUS.md:3-6`.

## 2. Modelo de dominio

Entidades (docs/architecture.md:55-61, 729-802; `backend/internal/domain/*`):

- **Project**: repo + config (`projectconfig.go`), reglas de agente/orquestador, `autoReview`.
- **Session** (`domain/session.go`): `kind` worker|orchestrator; `harness`; `session_mode` (tui|chat); handle de runtime; `activity_state` (active, idle, waiting_input, blocked, exited); `is_terminated`; `controller_generation`; metadata. Puede ser "standalone" sin proyecto (workspace = directorio plano).
- **Conversation** (turnos, mensajes, actividades) con ámbito orquestador-de-proyecto o worker-de-sesión.
- **PullRequest** (`sessions ||--o{ pull_requests`) con `pr_checks`, `pr_review_threads`, `pr_comments`, y `last_nudge_signature` (dedup de reacciones). Una sesión puede poseer varios PRs (reactions.go:82-86).
- **Reacción**: no es entidad con DSL declarativo; es código en `lifecycle/reactions.go` (nudges CI-failure / review-feedback / merge-conflict) más flags por PR `AutoInjectCI` / `AutoInjectReview` (`domain/pr.go:116,212,247`).
- **ReviewRun** (`domain/review.go`, `review/run_lifecycle.go`): estados queued, launching, running, cancelling, completed, failed, cancelled, superseded (docs/architecture.md:209-211). Un "reviewer" es un agente sin fila de sesión.
- **Notification**, **Automation** (cron/RRULE), **Cue** (prompts reutilizables), **Tracker issue** (intake GitHub/GitLab que puede spawnear workers).
- **Evento**: no hay bus de eventos de dominio de primera clase; el flujo es `observación -> hecho durable -> derivar`. Los "eventos" visibles son filas de `change_log` por triggers SQLite (CDC) emitidas por SSE (docs/architecture.md:236-248, 804-820).

Relaciones clave: proyecto 1-N sesión; sesión 1-N PR; PR 1-N checks/comentarios; el PR pertenece al worker que lo creó (o lo reclama con `ao session claim-pr`, scm-observer.md componente ClaimPath).

## 3. Router/Orquestador

- **No hay un router LLM que clasifique eventos.** El enrutado de eventos externos es **determinista, en código Go**: SCM Observer (poll cada 30 s, ETag + diff semántico) -> `lifecycle.Manager.ApplyPRObservation` -> `sendOnce` -> mensajero consciente del modo (TUI pega en el PTY; Chat encola turno) (docs/architecture.md:631-720; `backend/internal/lifecycle/reactions.go:71-319`).
- **El orquestador-agente** (sesión `KindOrchestrator`) es un agente de código normal con un system prompt de rol (`session_manager/prompt.go:11-60`, reglas por proyecto `OrchestratorRules`) que llama al CLI `ao spawn` / API de delegación (`service/session/delegation.go:30-50`, `DelegateTaskInput`: proyecto, brief, agente, modelo, esfuerzo, modo de aprobación). Qué modelo usa lo elige el usuario/proyecto (harness por defecto); no hay modelo fijo.
- **Filtrado de eventos**: el filtro es de reglas: se ignoran los comentarios del propio autor del PR (reactions.go:179-186), solo se inyecta si `AutoInjectCI`/`AutoInjectReview` (reactions.go:195,213,236), conflicto solo en la base de una pila (no se avisa si el PR depende de un padre abierto, reactions.go:275-282), dedup por firma persistida y máximo de reintentos (`reviewMaxNudge = 3`, reactions.go:19), sin límite para CI/conflicto (`maxAttempts: 0`, líneas 205, 291; ver sendOnce, no leído).
- Un freno de "auto-spawn": tras 3 fallos iguales seguidos se pausan arranques automáticos, con sonda tras 30 min (`service/session/auto_spawn_breaker.go:15-25`).
- Mensajes de lifecycle durante un traspaso TUI<->Chat van a un outbox durable (architecture.md:564-570).

## 4. Worker

- **Aislamiento**: git worktree por sesión en `worktrees/<project-id>/<session-id>`, rama temporal `ao/<session-id>/root` renombrada luego a `ao/<slug>/root` (architecture.md:387-400); o directorio plano para standalone. **No hay contenedor ni sandbox de SO**: el agente corre en el host con los permisos del usuario. Lo único de contenedores es un `dockerreap` que, al terminar la sesión, borra contenedores etiquetados `ao.session=<id>` que el propio worker lanzó (`adapters/container/dockerreap/reap.go:1-30`), con sesgo a "no borrar si hay duda". Los docs lo admiten: "native modes and prompts are not OS or network isolation" (STATUS.md:98-101); el reviewer contenido es futuro (STATUS.md:102-106, adr/0002).
- **Runtimes**: PTY nativo (macOS/Linux), ConPTY (Windows), tmux legado; modo Chat con controladores nativos (Codex app-server, ACP) en hosts detached por sesión que sobreviven al reinicio del daemon (architecture.md:8-19; STATUS.md:46-61).
- **Cómo reporta**: hooks del agente (`ao hooks`, `adapters/agent/activitydispatch`) -> `activity_state`; `report.go` (worker reports con attention state y artefactos/PR) (STATUS.md:154-155; `domain/report.go`, no leído en detalle). Además observación del PR.
- **Ciclo de vida**: Spawn -> MarkSpawned -> active/idle/waiting/exited -> Terminated; Kill, restore, resume nativo, cambio de cuenta (architecture.md:927-960). Terminación automática al fusionarse el PR sin PR abiertos y con agente no activo (reactions.go:87-118).
- 40 harnesses en `adapters/agent/` (Claude Code, Codex, Cursor, Aider, DeepSeek harness, etc.; README.md:62-141), catálogo con soporte distinto Worker/Chat/Reviewer.

## 5. Estado y memoria

- **SQLite** (goose + sqlc) en `~/.ao` (`AO_DATA_DIR`) (AGENTS.md:125); tablas en architecture.md:729-802. CDC por triggers a `change_log` -> poller -> broadcaster -> SSE `/api/v1/events` con `Last-Event-ID` (STATUS.md:30-37).
- **Principio**: no se guarda el estado de pantalla; solo hechos duraderos y el status (working, ci_failed, changes_requested, merge_conflict, mergeable...) se **deriva en lectura** (architecture.md:49,824-861; AGENTS.md:115).
- **Memoria conversacional**: conversaciones durables con paginación, archivo de eventos crudos, compactación, rollback, continuidad del orquestador por proyecto (STATUS.md:62-65). `handoff_note.go` para notas de traspaso (no leído).
- Dedup de reacciones persistido en `pr.last_nudge_signature` JSON `{seen, attempts}` (reactions.go:42-48): sobrevive reinicios.
- No hay memoria semántica/vectorial ni "lecciones aprendidas" del orquestador: no encontrado.

## 6. Watchdog y manejo de fallos

- **Runtime Reaper** (`observe/reaper/reaper.go`, tick por defecto 5 s, línea 23): sondea el runtime de cada sesión y reporta hechos al LCM; también supervisa reviewers con límites de tiempo (lanzamiento, primera actividad, inactividad, duración máxima, cancelación) y mata el árbol de procesos si ignoran el cancel (architecture.md:205-234).
- **Regla clave**: una sonda fallida o desconocida NO prueba que la sesión esté muerta; solo se termina si runtime y proceso están muertos y no hay actividad reciente ni PR fusionado (architecture.md:962-985; AGENTS.md:116).
- Reconciliación de arranque tras caída del daemon: transiciones interrumpidas, runs de review huérfanos (boot pass), hosts detached readoptados.
- Tiempo no observado (suspensión de máquina) no cuenta contra una run (architecture.md:219-220).
- Freno anti-bucle de auto-spawn (ver 3). Reintentos de entrega con clave de idempotencia en Chat.
- **No encontrado**: un watchdog que detecte "worker atascado sin avanzar pero vivo" y lo reasigne/escale; el equivalente es la notificación al humano (`needs_input`, etc.) y el estado `no_signal` (architecture.md:856-857).

## 7. Seguridad y autorización

- **Daemon**: listener primario en `127.0.0.1` sin auth (por diseño), segundo listener LAN opt-in para móvil con bearer-password y lockout, una sola ruta sin auth (`GET /api/v1/identity`, ADR 0003) (AGENTS.md:111-112). Cloudflare/Tailscale opcionales (ADR 0004).
- **Inyección de prompts**: solo hay defensa contra **inyección de terminal**: `SanitizeControlChars` elimina secuencias de control de lo que va al PTY (comentarios de reviewer, colas de logs de CI) (`domain/text.go:8-24`). Contra inyección semántica (un comentario de review malicioso que le diga al agente que haga algo) **no encontré defensa**: el texto del comentario se pega como instrucción (reactions.go:217-224). Solo se filtran respuestas del propio autor.
- **Guardia de escritura** `sessionguard`: relee el estado justo antes de pegar y se niega si la sesión está bloqueada en un diálogo de permisos (un Enter automático respondería por el usuario); falla cerrado si no puede leer estado; lease de entrada para evitar carrera (`sessionguard/guard.go:1-45`). Los nudges de conflicto son "urgentes" pero solo entran en prompt idle probado por el harness (reactions.go:145-158).
- **Permisos de agente**: `PermissionMode` default/accept-edits/auto/bypass-permissions mapeado a flags nativos de cada agente (`domain/agentconfig.go:11-19`); sin sandbox propio. Algunos harness (Pi, Unreal) solo admiten Chat con bypass explícito (STATUS.md:66-85).
- **Credenciales**: AO "nunca lee un token"; cuentas = carpetas de config del CLI (`CLAUDE_CONFIG_DIR`, `CODEX_HOME`) (STATUS.md:139-143); telemetría con usuario GitHub e ID de instalación (README.md:218-220), desactivable.
- Todo el estado bajo `~/.ao`, con excepciones documentadas (AGENTS.md:125).

## 8. Madurez real

- **Tests**: ~1633 ficheros `_test.go` y ~1313 `*.test.ts(x)` (por `find`, no ejecutados). Los tests de lifecycle/reaper/guard son extensos (p. ej. `review_test.go` 167 KB, `guard_test.go` 24 KB). CI con `go test -race`, golangci-lint, drift de OpenAPI/sqlc (AGENTS.md:16-36, STATUS.md:14-23).
- **Implementado (verificado en código)**: nudges de CI/review/conflicto con dedup persistente (`lifecycle/reactions.go`), reaper, sessionguard, sanitizado, dockerreap, auto-spawn breaker, delegación, ciclo de review runs.
- **Implementado según docs, no verificado en código**: handoff TUI<->Chat, hosts ACP detached, cloud, móvil, browser automation.
- **Declarado en vuelo / no implementado** (STATUS.md:259-295): Cloud no universal, aceptación de browser en macOS/Linux, import de historial terminal bruto, portabilidad de tool calls en vuelo, espejo hacia tracker (solo lectura de issues), exposición de hechos PR crudos (#110/#111), reviewers contenidos.
- Los docs son muy detallados y honestos sobre límites; hay riesgo de deriva (por eso `documentation-map.md` define cuál gana).

## 9. Ideas originales / lo distintivo

1. **Enrutado de feedback al agente dueño del PR**: propiedad sesión->PR; el observer persiste hechos y el LCM decide nudges **independientes** por condición (CI, comentario por comentario, review "changes requested", conflicto) en cola y los envía juntos para que ninguno oculte a otro (reactions.go:159-166).
2. **Dedup con re-armado inteligente**: la firma del conflicto solo se limpia con un `mergeable/unstable` definitivo, nunca con `unknown` (flapping de GitHub) ni `blocked` sintético (reactions.go:380-396); sobrevive reinicios.
3. **Conflictos no se avisan en PRs apilados** sobre un padre abierto (reactions.go:275-282).
4. **Estado derivado, nunca almacenado** + CDC por triggers (un único camino de escritura).
5. **Guardia de entrada**: no responder por el humano en diálogos de permisos (`sessionguard`).
6. **Un controlador vivo por sesión** con fencing por generación y handoff TUI<->Chat con native resume.
7. **Reviewer agent** que lee cada PR antes que un humano, con ciclo de run con límites.
8. Dedup excluyendo las respuestas del propio worker en el hilo del PR (reactions.go:179-186).
9. Interruptor del usuario por PR/sesión para auto-inyectar CI/review.

## 10. Carencias y huecos

- Sin aislamiento de SO/red: el worker corre en el host del usuario con sus credenciales. Mitigación solo por modo de permisos del agente.
- Sin defensa semántica contra prompt injection en comentarios de review/CI.
- Sin router LLM ni clasificador de eventos: solo reglas fijas en Go; añadir otro tipo de evento exige código.
- Orquestador = agente de código con prompt, sin garantías duras (no hay "mandato"/presupuesto/autorización formal por worker más allá del modo de permisos).
- Monousuario y local; la multi-máquina depende de AO Cloud (plano de control privado, no en el repo; AGENTS.md:13).
- Telemetría por defecto con usuario GitHub (README.md:218).
- Producto de GUI: acoplar el daemon a otro sistema implica pasar por su API HTTP/OpenAPI.
- No leí `sendOnce` completo ni el reducer de actividad, así que los topes de reintentos/backoff exactos quedan sin verificar.

## 11. Parecidos y diferencias con el linaje DeepSeek Router/Worker

(Basado en el vocabulario del encargo: Mandate/Subscription/Eval, Watchdog, aislamiento; no leí el linaje, comparo con lo que el encargo nombra.)

- **Router**: allí un Router decide; aquí el "router" es el LCM determinista + el orquestador-agente. No hay Subscription declarativa a eventos: la suscripción es implícita (el dueño del PR recibe sus eventos).
- **Mandate**: AO no tiene mandato formal; lo más cercano es el brief de `DelegateTaskInput` + `PermissionMode` + flags `AutoInject*` + reglas de proyecto. Sin presupuesto/alcance verificable.
- **Eval**: lo más cercano es el reviewer agent y el estado derivado (CI/review/mergeable); no hay evaluación de calidad con métricas propia del worker.
- **Watchdog**: el Reaper es equivalente en liveness (con la regla "sonda fallida != muerto", útil de copiar); no vigila progreso/atasco.
- **Aislamiento**: worktree sí; contenedor/sandbox no. DeepSeek-style aislamiento fuerte sería una mejora sobre AO.
- Parecido: worker por rama/worktree, reporte por hooks, estado durable en SQLite.
- Diferencia mayor: AO trata el agente como un CLI interactivo (PTY) al que se le "pega" texto; no hay protocolo de herramientas propio con el orquestador (salvo el modo Chat/ACP).

## 12. Compatibilidad con Windows

- Soportado oficialmente: instalador `agent-orchestrator-win32-x64.exe` (README.md:174), `platforms.mdx` "windows=full", ConPTY nativo sin tmux (`frontend/src/docs/content/platforms.mdx`), `adapters/runtime/conpty/host_conpty_windows.go`, `spawn_windows.go`, `pidalive_windows.go`, `lifecycle/force_stop_freeze_windows.go`, `account_home_windows.go`.
- Matices: `dev:warm` siempre hace `npm ci` en Windows (AGENTS.md:61); STATUS.md:270-273 dice empaquetado Windows x64 verificado y macOS/Linux pendiente; la verificación de artefactos y parte del tooling de desarrollo son macOS-céntricos; el árbol de procesos y los worktrees dependen de git en Windows (no verificado). Rutas largas de worktree no verificadas.

## Evidencias

| Afirmación | Ruta:línea |
|---|---|
| Licencia Apache 2.0 | `LICENSE` (cabecera; línea 189 "Copyright 2026 Untrivial"); README.md:222-224 |
| Es reescritura en Go (no TypeScript) | AGENTS.md:3,9-11; docs/architecture.md:3 |
| Fallos CI/review/conflictos vuelven al worker dueño | README.md:52-56; docs/architecture.md:659-689 |
| Nudges independientes en cola | backend/internal/lifecycle/reactions.go:159-166,299-303 |
| Filtra respuestas del propio autor | reactions.go:179-186 |
| Dedup persistido en `last_nudge_signature` | reactions.go:42-48 |
| Max 3 reintentos para review | reactions.go:19,230,246 |
| Conflicto solo en base de pila | reactions.go:275-282 |
| Re-armado solo con mergeable/unstable | reactions.go:380-396 |
| Termina sesión al merge sin PR abierto | reactions.go:87-118,342-360 |
| Status derivado, no almacenado | docs/architecture.md:49,65; AGENTS.md:115 |
| Fallo de sonda != muerte | docs/architecture.md:985; AGENTS.md:116 |
| Reaper tick 5 s | backend/internal/observe/reaper/reaper.go:23 |
| Límites y kill de reviewer | docs/architecture.md:205-234 |
| Worktree por sesión, rama temporal | docs/architecture.md:387-400 |
| Sin sandbox SO/red declarado | docs/STATUS.md:98-101 |
| dockerreap solo reapea contenedores etiquetados | backend/internal/adapters/container/dockerreap/reap.go:1-30 |
| Sanitiza control chars hacia el PTY | backend/internal/domain/text.go:8-24 |
| Guardia contra responder diálogos de permisos | backend/internal/sessionguard/guard.go:1-45; reactions.go:145-158 |
| Modos de permiso | backend/internal/domain/agentconfig.go:11-19 |
| Sesión kind worker/orchestrator | backend/internal/domain/session.go:20-25 |
| Delegación de tareas | backend/internal/service/session/delegation.go:30-50 |
| Freno de auto-spawn (3 fallos, 30 min) | backend/internal/service/session/auto_spawn_breaker.go:15-25 |
| Prompt por rol orquestador/worker | backend/internal/session_manager/prompt.go:11-60 |
| Listener loopback sin auth + LAN opt-in | AGENTS.md:111-112 |
| Estado bajo `~/.ao` | AGENTS.md:125; CLAUDE.md |
| CDC por triggers + SSE | docs/STATUS.md:30-37; docs/architecture.md:804-820 |
| Pendiente/en vuelo | docs/STATUS.md:259-295 |
| Windows ConPTY sin tmux | frontend/src/docs/content/platforms.mdx (sección Windows); README.md:174 |
| No leí `sendOnce` ni cuerpo de manager.go | (laguna declarada) |

Cobertura: ~3-4% de los ficheros por número (~250 de ~7978), pero cubrí las fuentes clave: README, AGENTS.md, CLAUDE.md, LICENSE, docs/architecture.md (líneas 1-1000 de ~1400+), docs/STATUS.md completo, cabecera de scm-observer.md, `lifecycle/reactions.go` (1-420), cabeceras de sessionguard, dockerreap, reaper, domain/text.go, prompt.go, delegation.go, auto_spawn_breaker.go, platforms.mdx y listados de directorios. Sin leer: frontend, mobile, adapters de agente concretos, chatdriver, storage/sqlite (esquema solo por docs), review engine interno, cloud, `sendOnce`, el resto de architecture.md.


## 13. Profundización (2026-11-21, solo lectura)

**Rutas abreviadas en este apartado**: `claudecode.go` = `backend/internal/adapters/agent/claudecode/claudecode.go` (hay otro homónimo en `adapters/reviewer/claudecode/`).

Segunda pasada dirigida a los huecos de la nota. Se leyeron unos 50 ficheros más, sobre todo de `backend/internal/` (lifecycle, sessionguard, Windows, migraciones, registro de drivers). Es ≈0,6 % de los ficheros por número y ≈4-5 % del backend de lifecycle y estado. Nada se ejecutó. **Sin leer aún**: `lifecycle/manager.go`, `observe/` (SCM observer), `session_manager/`, el `runtimeMessenger.Send` real, las transacciones de `store/` y el frontend.

| Punto | Veredicto | Evidencia |
|---|---|---|
| `sendOnce` y `maxAttempts: 0` | **CORREGIDO** | No hay reintento explícito ni backoff: el "reintento" es el re-poll del observer más la dedup (`reactions.go:1039-1114`). `maxAttempts: 0` significa "sin tope fijo" (comprobación `maxAttempts > 0 && attempts >= maxAttempts`, línea 1066). Lo que frena el bucle es la firma: `if m.react.seen[key] == sig` no envía (1062). La firma de CI incluye `CommitHash` y `LogTail` (848-854), así que **un worker que empuja y falla en bucle recibe un nudge por push, sin techo**. Review y comentarios: tope 3 por clave (`reviewMaxNudge`, 19). Orden: enviar, mutar memoria, persistir; un fallo de persistencia puede dar un nudge de más tras reinicio, nunca uno perdido (1100-1105) |
| ConPTY nativo en Windows | **CONFIRMADO** | Implementación real con `go-pty` y `ClosePseudoConsole` (`host_conpty_windows.go:1-136`); tests con `cmd.exe` real (`host_conpty_windows_test.go`); ~20 ficheros `_test.go` con `go:build windows`; job `windows-latest` en CI que ejecuta solo tests seleccionados (`.github/workflows/go.yml:82-127`). `host_conpty_other.go` no leído |
| Detector de "vivo pero atascado" | **CORREGIDO** | `no_signal` es un estado derivado en lectura (`domain/status.go:25-29`; `pkg/contract/status.go:107-144`): se activa si se espera señal, nunca llegó ningún hook y pasan 90 s (`service/session/status.go:17`). Solo cambia la etiqueta: no mata, no reenvía y **no avisa**. Si el worker mandó un hook y luego se cuelga, vuelve a verse como `idle` o `working`. Los únicos timeouts reales son de reviewers (`review/run_lifecycle.go:19-49`). Reaper de workers: solo liveness cada 5 s (`reaper.go:23`) |
| Inyección de prompt | **CORREGIDO parcialmente** | `SanitizeControlChars` solo quita caracteres de control (`domain/text.go:19-26`) y se aplica a autor, fichero, URL y cuerpo (`reactions.go:931,960-975`). **El log de CI sí va entre fence markdown**, que crece si el log trae backticks (`markdownCodeFence`, 1002-1019), aunque sin etiqueta de "no confiable". Los comentarios de review van **en crudo** (973) y el mensaje dice "Address each comment and push fixes" (982) |
| SQLite y outbox | **CORREGIDO** | ~215 migraciones y ~120 `CREATE TABLE` (incluidas temporales). `worker_idle_events` (`0025`) se eliminó en `0037`. El outbox vigente es `session_interface_transition_messages` (`0078`); `0079` añade `client_message_id` para idempotencia en Chat. **La entrega a terminal es best-effort: tmux no tiene ack.** La dedup de nudges no es tabla: es el JSON `{seen,attempts}` en `pr.last_nudge_signature` (`0005`), protegido por un mutex **en memoria** (`reactions.go:1043-1044`), sin transacción SQL. WAL, `busy_timeout(5000)`, escritor único (`db.go:38-51,162,212`) |
| Número de adaptadores | **CORREGIDO** | 45 constantes en `domain/harness.go`; 44 seleccionables (`fake` es de fixtures). 19 drivers Chat registrados (`chatdriver/registry/registry.go:94-137`), ~25 harnesses solo TUI |
| `PermissionMode` | **CONFIRMADO** | Claude: `--permission-mode` (`claudecode.go:148-167`). **Aviso: en Codex el modo `default` equivale a bypass total** (`--dangerously-bypass-approvals-and-sandbox` en TUI, `codex_test.go:615-648`; `never` + `danger-full-access` en Chat, `codexappserver/driver.go:794-803`) |
| Aislamiento | **CONFIRMADO** | Solo worktree más `dockerreap`, que únicamente limpia contenedores etiquetados `ao.session=<id>` al matar la sesión (`container/dockerreap/reap.go:1-136`). Sin sandbox de SO propio (buscado `sandbox|seatbelt|bwrap|landlock`) |

**Hallazgos nuevos**: guarda "urgent" que entrega aunque la sesión esté en `waiting_input`, salvo harnesses que enmascaran permisos como ese estado (`guard.go:329-361`); *mass-death circuit breaker* en el reaper (`reaper.go:173,231-235`).
