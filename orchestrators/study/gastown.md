# Estudio: Gas Town (gastownhall/gastown)

Estudio de solo lectura. Rutas relativas a `references/gastown/`. Nada se ejecutó.

## 1. Resumen

- **Qué es**: gestor de workspace multiagente ("Gas Town") que coordina muchos agentes de código (Claude Code, Copilot, Codex, Gemini, Cursor, OpenCode...) sobre uno o varios repos git, con tmux como "terminal" de los agentes, un ledger de trabajo persistente (Beads sobre Dolt) y una jerarquía de agentes supervisores (README.md:3-7, 86-96).
- **Stack**: Go (go.mod:3 `go 1.26.2`, módulo `github.com/steveyegge/gastown`, go.mod:1), depende de `steveyegge/beads v1.0.5` (go.mod:20) y Dolt (servidor SQL por town, puerto 3307, `docs/design/dolt-storage.md:6-20`). Binarios `gt` (este repo) y `bd` (beads, repo aparte).
- **Licencia verificada**: MIT, "Copyright (c) 2025 Steve Yegge" (LICENSE:1-3). Nota: el repo se llama gastownhall pero el copyright y el módulo son de steveyegge.
- **Tamaño**: ~1232 ficheros `.go`, 569 `_test.go`, ~70 paquetes en `internal/`; `daemon/daemon.go` 3098 líneas, `cmd/sling.go` 1427 líneas.
- **Madurez**: muy activo, alta complejidad, "experimental" según SECURITY.md:40; versión soportada "0.1.x" (SECURITY.md:56). CHANGELOG de 94 KB. Mucha documentación de diseño con estados "Proposal/Design".

## 2. Modelo de dominio

Vocabulario propio (docs/glossary.md, README.md:46-120):

- **Town**: directorio raíz del workspace (`~/gt/`); contiene todo y los agentes de nivel town (glossary.md:12-14).
- **Rig**: contenedor de un proyecto/repo git; no es un clon, sino un directorio con `mayor/rig` (clon canónico donde viven los beads), `refinery/`, `witness/`, `crew/`, `polecats/` (docs/design/architecture.md:114-128).
- **Roles town**: **Mayor** (coordinador "chief-of-staff", un Claude Code con contexto global), **Deacon** (supervisor/"beacon" con ciclos de patrulla), **Boot** (perro que vigila al Deacon), **Dogs** (trabajadores de mantenimiento cross-rig) (architecture.md:64-71).
- **Roles rig**: **Witness** (vigila polecats), **Refinery** (cola de merge tipo Bors), **Polecats** (workers: identidad persistente, sesión efímera), **Crew** (workspaces humanos, clones completos) (architecture.md:73-80).
- **Unidades de trabajo**: **Bead** (issue/tarea atómica en Dolt, IDs `gt-abc12`, `hq-x7k2m`; README.md:80), **Convoy** (agrupa beads, `mountain` = épico con detección de atasco; README.md:72-74), **Formula** (plantilla TOML), **Molecule** (formula instanciada con pasos rastreados; README.md:82-84), **Wisp** (bead efímero), **Hook** (bead anclado por agente = su cola de trabajo; glossary.md:~70), **MR** (merge request bead).
- **Mensajería**: `gt mail` (persistente) y `gt nudge` (entrega inmediata a la sesión tmux) (AGENTS.md:83-125).
- **Principios**: MEOW, GUPP ("si hay trabajo en tu hook, ejecútalo"; docs/concepts/propulsion-principle.md:1-20), NDI (idempotencia no determinista) (glossary.md:5-10).
- **Relaciones**: Town 1-N Rig; Rig 1-N Polecat/Crew + 1 Witness + 1 Refinery; Mayor -> convoys/beads -> `gt sling` pone un bead en el hook de un polecat; Polecat -> `gt done` -> MR -> Refinery -> main.
- Beads en dos niveles: town (`hq-*`, coordinación, identidad de agentes) y rig (prefijo de proyecto), con `routes.jsonl` para enrutado por prefijo (architecture.md:5-30, 175-191). Cada agente tiene un "agent bead" que referencia un "role bead" (architecture.md:32-60).

## 3. Router/Coordinador

- **Quién decide**: el **Mayor**, un agente LLM (Claude Code por defecto) priorizado con una plantilla de rol embebida (`internal/templates/roles/mayor.md.tmpl`, 14.9K). Recibe lo que el humano pide, crea convoys/beads y los "sling" a rigs/polecats. Manager de ciclo de vida en `internal/mayor/manager.go` (modos tmux/ACP; manager.go:23-47).
- **Despacho determinista**: `gt sling` (`internal/cmd/sling.go`, `sling_dispatch.go`, `sling_target.go`) pone el bead en el hook del destino; hay idempotencia, locks, validación, rollback (ficheros `sling_idempotency.go`, `sling_validate.go`, tests `sling_rollback_cleanup_test.go`). No leí el cuerpo de sling.go en detalle.
- **Scheduler de capacidad**: config `scheduler.max_polecats`; -1/0 = despacho directo, N>0 = diferido por daemon (docs/design/scheduler.md:20-40). Prevención de rate limits de API.
- **No hay un modelo router "barato" aparte**: el router es un agente completo, no un clasificador. Selección de modelo/runtime por agente mediante presets (`internal/config/agents.go:20-50`: claude, gemini, codex, cursor, copilot, opencode, etc.) y `docs/design/model-aware-molecules.md` (no leído).
- **Herramientas/filtrado**: el Mayor usa los comandos `gt`/`bd`; el contexto se inyecta con `gt prime` por hook SessionStart (architecture.md:131-133). Filtrado/permiso por rol: hooks de Claude gestionados (`internal/hooks/`), directivas por rol (`directives/<role>.md`) y overlays de formulas (architecture.md:~330-380). El Mayor está exento del E-stop (estop.go:4-6).

## 4. Worker

- **Polecat** = worker. **Aislamiento**: git worktree propio por polecat basado en `mayor/rig`, rama `polecat/<name>-<timestamp>` (architecture.md:135-146); sesión tmux propia (internal/polecat/session_manager.go). El aislamiento es solo de git/filesystem lógico: **comparten usuario y filesystem** (SECURITY.md:42).
- **Reporte**: `gt done` empuja la rama, crea el MR bead, limpia el hook, marca estado "done" y mata su propia sesión (docs/concepts/polecat-lifecycle.md:55-70); notifica por mail (`HandlePolecatDone`, internal/witness/handlers.go:153) y el Witness avisa al Refinery (handlers.go:341).
- **Ciclo de vida**: estados Idle -> Working -> Done, más Stalled y Zombie (polecat-lifecycle.md:18-26). Identidad (agent bead, CV chain) persiste; sesión no (polecat-lifecycle.md:5-10, 60-75). Modelo "retired completion": no se reutiliza sesión completada. Auto-gestionado: el Witness observa pero no autoriza la finalización (architecture.md:239-258).
- Nombres de polecat vía pool (`internal/polecat/namepool.go`), heartbeats (`internal/polecat/heartbeat.go`).

## 5. Estado y memoria

- **Fuente de verdad**: Dolt SQL Server único por town, bases por rig, sin SQLite ni modo embebido (dolt-storage.md:6-20; architecture.md:148-173). Escrituras directas a `main` con transacciones `BEGIN/DOLT_COMMIT/COMMIT`.
- **Formato**: beads (filas SQL versionadas), `routes.jsonl`, `town.json`, `rigs.json`, `daemon.json`, `settings/config.json`, `settings/escalation.json` (architecture.md:85-113). Worktrees usan `.beads/redirect` (max profundidad 3) hacia el beads canónico (architecture.md:193-204).
- **Sobrevive a reinicios**: hooks/beads, agent beads, mail, ramas git, MR evidence. No sobrevive: contexto LLM; se reconstruye con `gt prime` y `gt hook` (AGENTS.md:127-136). `gt seance` descubre sesiones previas via `.events.jsonl` para consultar a predecesores (README.md:106-113).
- **Ciclo de vida de datos**: CREATE->LIVE->CLOSE->DECAY->COMPACT->FLATTEN; etapas 1-3 automatizadas, 4-6 "se están entregando" vía Dogs (architecture.md:260-273). Backup JSONL cada 15 min a repo git (dolt-storage.md:11-15; `internal/daemon/jsonl_git_backup.go`).
- Persistencia de conocimiento: `bd remember` (AGENTS.md:158).

## 6. Watchdog y manejo de fallos

Si hay agentes supervisores; es el núcleo del diseño (README.md:86-92):

- **Daemon** Go (`internal/daemon/daemon.go`): heartbeat de 3 min, comprueba Dolt y lo reinicia con backoff (daemon.go:586; dolt-storage.md:~28), patrullas de 15 min, etc.
- **Boot**: "dog" efímero lanzado por el daemon en cada tick para decidir wake/nudge/nada sobre el Deacon (internal/boot/boot.go:1-4; cooldown daemon.go:89).
- **Deacon**: agente patrulla continua; detección de sesiones atascadas con ping HEALTH_CHECK (timeout 30s), 3 fallos consecutivos -> force-kill, cooldown 5 min (internal/deacon/stuck.go:14-30). También `redispatch.go`, `stale_hooks.go`.
- **Witness** por rig: monitoriza polecats, limpia, maneja mensajes de done/merged/merge-failed (internal/witness/handlers.go:153-525); recuperación de zombies.
- **Refinery** maneja fallos de MR (bisect Bors; architecture.md:206-237). Fases 1-3 marcadas "In progress"/"Blocked" en el propio doc, es decir, parcialmente prometido.
- **Escalado**: `gt escalate` con severidades P0-P2 enrutado Deacon->Mayor->Overseer (README.md:98-100; docs/design/escalation.md).
- **E-stop**: fichero centinela `ESTOP` en la raíz; congela agentes con SIGTSTP, el daemon no los reinicia, el Mayor exento (internal/estop/estop.go:1-10, 25-28). `gt thaw` para reanudar.
- Los supervisores son en parte agentes LLM (Witness/Deacon con plantillas de 12-17K) y en parte código determinista del daemon. Ver riesgo en sección 10.

## 7. Seguridad y autorización

- Postura declarada: experimental; agentes ejecutan comandos con el usuario; aislamiento "separate tmux sessions but share filesystem" (SECURITY.md:38-46). Recomienda entornos aislados.
- Permisos de agente: el preset de Claude usa `--dangerously-skip-permissions` (internal/config/agents.go:236), es decir, autonomía total.
- **Proxy mTLS** (`internal/proxy/server.go`, `ca.go`, `denylist.go`): servidor que ejecuta `gt`/`bd` en nombre de polecats sandboxed, con allowlist de comandos y subcomandos (server.go:26-33), CA propia, lista de revocación de certificados por serial (denylist.go:1-25) y rate limit (`golang.org/x/time/rate`). Documentado en docs/proxy-server.md (no leído en detalle).
- **Sandbox**: `docs/design/sandboxed-polecat-execution.md` (exitbox + daytona) está marcado "Status: Proposal" (línea 5) con el problema de seguridad descrito explícitamente (líneas 11-20); hay referencias a exitbox en `internal/config/{loader,types}.go` y un `sandbox_integration_test.go`, o sea hay implementación parcial; alcance exacto no verificado. También `docs/runtimes/macos-sandbox-exec.md` (no leído).
- Identidad: `BD_ACTOR` (docs/concepts/identity.md, no leído a fondo). Directivas de operador por rol (architecture.md:~330).
- Autorización entre agentes: no encontré modelo de permisos por rol aplicado en tiempo de ejecución más allá de proxy + hooks de Claude; el resto es confianza.

## 8. Madurez real

- Tests: 569 ficheros `_test.go` de 1232 `.go` (~46%), incluyendo tests de integración (p. ej. `convoy_manager_integration_test.go`), de guardas (`deacon_crashloop_guard_test.go`, `boot_spawn_frequency_test.go`) y e2e vía Dockerfile.e2e. No se ejecutaron.
- **Implementado (código verificado)**: daemon, Boot, Deacon stuck detection, Witness handlers, estop, proxy mTLS, polecat session manager, sling con scheduler, mail/nudge, role templates, presets multi-runtime, hooks.
- **Prometido/parcial**: etapas 4-6 del ciclo de datos (architecture.md:260-273), merge queue batch-then-bisect fases 2-3 (architecture.md:229-233), sandbox de polecats (Proposal), federación/Wasteland (README.md:115-117; docs/WASTELAND.md, no verificado en código), "escala cómoda a 20-30 agentes" (README.md:15, afirmación sin evidencia verificable por mí).
- Los docs de arquitectura tienen citas internas desactualizadas en parte (p. ej. architecture.md:189 usa `~/.gt/.beads` vs `~/gt/.beads`; AGENTS.md remite a un CLAUDE.md que no existe en el repo).
- `AGENTS.md` del repo es sobre todo instrucciones de Beads + comunicación Gas Town (objeto de estudio, no seguido).

## 9. Ideas originales / lo distintivo

1. **Identidad persistente, sesión efímera** (polecat): trabajo y reputación (CV chain) separados del proceso.
2. **Hook + GUPP**: la cola de trabajo como bead anclado al agente; el agente arranca y ejecuta (propulsion-principle.md).
3. **Cadena de supervisión de 3+ niveles** (daemon determinista -> Boot -> Deacon -> Witness -> Polecat) con "quién vigila al vigilante".
4. **Ledger versionado Dolt** como memoria compartida multiagente, con ciclo de vida de compactación.
5. **Refinery Bors-style** como parte del sistema multiagente.
6. **Molecules/Formulas + overlays y directivas** por town/rig (replace/append/skip) sin tocar binario.
7. **Seance**: preguntar a sesiones predecesoras.
8. **Wisps**, convoys `mountain` con stall detection, E-stop con SIGTSTP, nudge vs mail (efímero vs persistente), federación Wasteland.
9. Soporte multi-runtime con presets y `docs/research/w-gc-004-agent-framework-survey.md`.

## 10. Carencias y huecos

- Fuerte acoplamiento: tmux + Dolt + `bd` + git; muchos procesos (daemon, Dolt server, nudge pollers).
- Aislamiento débil por defecto (mismo UID, mismo FS, `--dangerously-skip-permissions`).
- Supervisores LLM consumen tokens y pueden alucinar; el propio doc reconoce "reliability" no garantizada y usa NDI como mitigación.
- Complejidad enorme (3098 líneas en daemon.go, ~70 paquetes) y vocabulario temático (polecat, deacon, wisp...) que eleva la curva de aprendizaje.
- Docs desfasados y mezcla de diseño/propuesta con realidad.
- Modelo del router: sin enrutado barato por clasificador; el Mayor es un LLM completo.
- No leído: grandes partes de `internal/cmd`, refinery, formula, wasteland, web/tui, acp, quota, krc.

## 11. Parecidos y diferencias con el linaje DeepSeek Router/Worker

Parecidos: coordinador (Mayor) que reparte a workers (polecats) aislados con identidad y estado propio; reporte estructurado al terminar (`gt done` + mail); vigilancia de atascos y reintentos.

Diferencias:
- El router de Gas Town es un agente conversacional persistente y humano-interactivo; en un linaje Router/Worker típico el router es una pieza ligera de decisión/filtrado.
- La memoria vive fuera del agente en un ledger SQL versionado (Dolt), no en el contexto ni en ficheros simples.
- Hay supervisión en capas con agentes supervisores y daemon, más Refinery de integración; el linaje Router/Worker típico es más plano.
- Aislamiento por git worktree + tmux, no por sandbox/contenedor (salvo propuesta).
- Aplica a código con merge queue; foco en desarrollo de software, no en tareas genéricas.
- Pesadez: infraestructura (Dolt, tmux, daemon) frente a un PoC ligero.

## 12. Compatibilidad con Windows

- **Parcial / "minimal CLI-only"**: docs/INSTALLING.md:90-101, 151: en Windows nativo solo uso mínimo CLI; para `--shell`, `gt up`, roles tmux y Mayor, usar WSL. La construcción nativa exige MSYS2 con ICU (INSTALLING.md:94).
- Hay ficheros `_windows.go` (28 detectados: tmux, daemon, lock, mayor, nudge, session, util, git) y 116 ficheros con `runtime.GOOS == "windows"`; releases incluyen binarios windows amd64/arm64 (architecture.md:~290).
- Funciones degradadas: el E-stop por SIGTSTP/SIGCONT no existe en Windows (`internal/cmd/estop_windows.go:17-20`, devuelve error "not supported"). tmux en Windows nativo no es un flujo soportado; todo el modelo de sesiones depende de tmux.
- Conclusión: para Windows 11 la vía práctica es WSL2.

## Evidencias

| Afirmación | Ruta:línea |
|---|---|
| Licencia MIT, copyright Steve Yegge | LICENSE:1-3 |
| Go 1.26.2, módulo steveyegge/gastown | go.mod:1,3 |
| Depende de beads v1.0.5 | go.mod:20 |
| Qué es / multi-runtime | README.md:3-7 |
| Roles Witness/Deacon/Dogs | README.md:86-92 |
| Refinery Bors-style | README.md:96; docs/design/architecture.md:206-237 |
| Taxonomía de agentes | docs/design/architecture.md:62-80 |
| Estructura de directorios town/rig | docs/design/architecture.md:84-129 |
| Polecat = worktree git | docs/design/architecture.md:135-146 |
| Dolt único por town, sin SQLite | docs/design/dolt-storage.md:6-20; architecture.md:148-173 |
| Redirect de beads | docs/design/architecture.md:193-204 |
| Ciclo de datos (4-6 en curso) | docs/design/architecture.md:260-273 |
| Polecat auto-gestionado | docs/design/architecture.md:239-258 |
| Estados y completion del polecat | docs/concepts/polecat-lifecycle.md:18-75 |
| GUPP / propulsion | docs/concepts/propulsion-principle.md:1-20 |
| Glosario MEOW/GUPP/NDI | docs/glossary.md:5-10 |
| Mail vs nudge | AGENTS.md:83-125 |
| Contexto vía gt prime | AGENTS.md:127-136; architecture.md:131-133 |
| Scheduler max_polecats | docs/design/scheduler.md:20-40 |
| Presets de agentes | internal/config/agents.go:20-50 |
| Claude con skip-permissions | internal/config/agents.go:236 |
| Mayor manager | internal/mayor/manager.go:23-47 |
| Boot watchdog | internal/boot/boot.go:1-4 |
| Stuck detection (30s, 3 fallos, 5 min) | internal/deacon/stuck.go:14-30 |
| Daemon heartbeat 3 min | internal/daemon/daemon.go:586 |
| Witness handlers done/merged | internal/witness/handlers.go:153,341,439,493 |
| E-stop centinela, Mayor exento | internal/estop/estop.go:1-10,25-28 |
| E-stop no soportado en Windows | internal/cmd/estop_windows.go:17-20 |
| Proxy allowlist subcomandos | internal/proxy/server.go:26-33 |
| Denylist de certificados | internal/proxy/denylist.go:1-25 |
| Sandbox = Proposal | docs/design/sandboxed-polecat-execution.md:5,11-20 |
| Aislamiento compartido, experimental | SECURITY.md:38-46 |
| Windows: solo CLI mínimo, usar WSL | docs/INSTALLING.md:90-101,151 |
| 569 tests / 1232 .go | recuento Glob/find sobre el repo |

Cobertura: ~15-20% del repo leído. Leído: README (líneas 1-135), AGENTS.md, SECURITY.md, LICENSE (cabecera), glossary, architecture.md (casi entero), parte de polecat-lifecycle, propulsion-principle, scheduler y sandboxed-polecat-execution (cabeceras), dolt-storage (cabecera), y cabeceras de internal/estop, proxy, boot, deacon/stuck, session_manager, witness/handlers, mayor/manager, config/agents. No leído: cuerpo de sling.go y daemon.go, refinery, formula, convoy, mail, hooks, templates de rol, wasteland, acp, web/tui, plugins, scripts, docker, gt-model-eval, pr-sheriff-evidence, la mayoría de los tests.


## 13. Profundización (2026-11-21, solo lectura)

**Rutas abreviadas en este apartado**: `daemon.go` = `internal/daemon/daemon.go` (no `internal/cmd/daemon.go`); `dolt.go` = `internal/daemon/dolt.go`; `stuck.go` = `internal/deacon/stuck.go`; `estop.go` = `internal/estop/estop.go`, salvo `estop.go:292,321`, que es `internal/cmd/estop.go` (verificado); `manager.go` = `internal/mayor/manager.go`.

Segunda pasada. ≈3-4 % del repo por número de ficheros, pero ≈35-45 % del contenido de los ficheros objetivo (`daemon.go` ~1.900 de 3.098 líneas). Nada se ejecutó. **Sin leer aún**: `daemon.go` 1700-2600 y 2750-3098, `convoy_manager.go`, el cuerpo de `engineer.go` (99 KB), `witness/`, `plugin/` (`stuck-agent-dog`), `ca.go` y `denylist.go` del proxy, `doltserver`, `acp`, `quota` y `wasteland`.

| Punto | Veredicto | Evidencia |
|---|---|---|
| Heartbeat y backoff | **CONFIRMADO con matices** | Heartbeat de 3 min: default compilado (`internal/config/operational.go:48`), no el `daemon.go:586` que citaba la nota (es un comentario). **Backoff = tres mecanismos distintos**: Dolt (`restartWithBackoff`, máx. 5 reinicios por ventana, delay que se duplica hasta 5 min, `dolt.go:62-95,511-573`); agentes (`RestartTracker`: 30 s → 10 min, crash-loop a los 5 reinicios en 15 min, `restart_tracker.go:44-52`); y **ninguno por actividad en el heartbeat** (`daemon.go:832`). Dolt tiene su propio ticker de 30 s (`daemon.go:584-595`) |
| Qué es código y qué es LLM | **CORREGIDO** | El código del daemon asegura que Dolt, Deacon, Witness, Refinery y Mayor estén vivos, detecta polecats muertos con trabajo en el hook y reapea sesiones idle (`daemon.go:853-1002`). **Pero la reacción a un polecat caído es solo un mail al Witness**: "Restart deferred to stuck-agent-dog plugin" (`daemon.go:2742-2744,2853-2870`). Boot es una sesión Claude en tmux (`daemon.go:1385-1395`) con tres cortafuegos antes de gastar un LLM (cooldown de 2 min, supresión de 15 min tras "nothing", idle guard) y un modo degradado determinista (`GT_DEGRADED`, 1357-1363) |
| **Detección de atasco (Deacon)** | **CORREGIDO: no es automática** | `internal/deacon/stuck.go` solo contiene estado y contadores (`RecordFailure`, `ShouldForceKill`, `IsInCooldown`); los valores 30 s / 3 fallos / 5 min están en `stuck.go:19-21`. La lógica está en comandos CLI (`gt deacon health-check` y `gt deacon force-kill`, `internal/cmd/deacon.go:865-1086`) que **nadie invoca automáticamente**: no aparecen en `internal/daemon`, `internal/templates` ni `formulas`, solo en docs y en una mención de `mol-deacon-patrol.formula.toml:595`. `LoadStuckConfig` no tiene llamadores fuera de tests: esa configuración no está conectada. El patrol del Deacon (prompt) usa `gt nudge --mode=queue` y una matriz "you decide the thresholds" (`mol-deacon-patrol.formula.toml:586-624`) |
| Único detector automático determinista | **NUEVO** | El daemon vigila el heartbeat del Deacon y, por código, hace nudge entre 5 y 20 min y kill por encima de 20 (`daemon.go:1584-1635`, `deacon/heartbeat.go:18,24`) |
| `gt sling` | **CONFIRMADO con matices** | Dos flocks (por bead, no bloqueante; por assignee, 20 intentos x 500 ms; `sling.go:1290-1343`). Idempotente **solo con target inequívoco**: los atajos `rig/name` y los pools quedan excluidos (`sling_idempotency.go:43-50`). El rollback es best-effort, no transaccional: los fallos de cada paso solo se loguean (`sling.go:1364-1427`). Admisión de capacidad con reservas y lock (`polecat_capacity.go:111-150`) |
| Refinery (batch-then-bisect) | **CORREGIDO, en sentido inverso** | El doc dice "In progress/Blocked" (`architecture.md:229-233`), pero el código ya tiene `AssembleBatch`, `BuildRebaseStack`, `ProcessBatch` y `bisectBatch` completos (`batch.go:58,97,211,487-573`). **Pero `ProcessBatch` y `ProcessMRInfo` no tienen llamadores fuera de tests**; `gt refinery` solo hace claim, release y list (`internal/cmd/refinery.go:627-869`), y **la fusión real la hace un agente LLM** con `mol-refinery-patrol` y `gt mq`. NO CONCLUYENTE sobre una ruta indirecta no vista |
| Sandbox | **CORREGIDO: no hay implementación de exitbox** | `exec_wrapper` es un prefijo de comando genérico (`internal/config/types.go:775-779`, `loader.go:2308-2361,2718-2730`). No hay código de exitbox ni perfil `gastown-polecat` fuera de tests y docs. `sandbox_integration_test.go` prueba el `sandbox-exec` de macOS y se salta si no está (`:64,71`). El diseño sigue en "Proposal" (`sandboxed-polecat-execution.md:5`) |
| Proxy mTLS | **CONFIRMADO, pero sin uso por defecto** | Allowlist de comandos y subcomandos (`proxy/server.go:24-33,105-120`), TLS 1.3 con `RequireAndVerifyClientCert` (211-213), revocación por serial (219-225), semáforo de 32 ejecuciones y 10 req/s por CN (49-60). Cliente: `gt-proxy-client`. **Nada en `internal/` fija `GT_PROXY_*` al lanzar polecats**. Por defecto el servidor escucha en `0.0.0.0:9876` (`cmd/gt-proxy-server/main.go:30-35`) |
| "20-30 agentes" | **NO CONCLUYENTE (solo docs)** | Aparece en `README.md:15` y en un survey de terceros (`docs/research/w-gc-004-…:476,567`), sin medidas. Sin benchmarks ni tests de carga. **`scheduler.max_polecats` por defecto es -1: sin tope** (`internal/scheduler/capacity/config.go:35-51`) |
| Windows | **CONFIRMADO con matiz** | `estop_windows.go:17-21` devuelve "process-group signaling not supported on Windows", así que el congelado falla; el fichero centinela `ESTOP` sí es portable. 28 ficheros `_windows.go`. **Hallazgo**: hay soporte de **psmux** (tmux para Windows) en `internal/tmux/tmux.go:370,1101,2711` y `config/loader.go:2294,2552`, aunque `INSTALLING.md` sigue diciendo uso mínimo en nativo. Sin verificar que funcione |

**Hallazgos nuevos**: muchos "dogs" opcionales como tickers del daemon (compactor, doctor, checkpoint, quota, wisp reaper, backups; `daemon.go:597-718`); detección de *mass death* y guarda de `spawning` de 5 min (`daemon.go:2696-2794`); el daemon no mata un Deacon parado en un prompt de límite de uso (`daemon.go:1658-1675`).
