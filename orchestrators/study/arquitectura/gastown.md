# Gas Town (gastownhall/gastown) - eje ARQUITECTURA

Estudio de solo lectura. Rutas relativas a la raíz del repo (`references/gastown/`). Nada se ejecutó. Commit leído: 94cc82289d2e (2026-11-12, "docs(docker): describe the Go tarball checksum..."), clon superficial.

## 1. Resumen y veredicto

- Un único binario Go (`gt`, cmd/gt/main.go:10-12) orquesta agentes de código en sesiones tmux; el estado vive en Dolt a través del CLI `bd` (beads) y en ficheros del town.
- Capas limpias a nivel de paquetes: no hay ciclos entre paquetes de primer nivel de `internal/` (análisis de imports propio, 73 directorios), y solo `cmd/gt/main.go` importa `internal/cmd`.
- Pero la lógica se concentra en `internal/cmd` (253 ficheros, 103.691 líneas, 41,6 % del código no-test) y en funciones gigantes (`runDone` 1.313 líneas, `runSling` 945).
- Gran parte del "control" lo ejecutan agentes LLM siguiendo fórmulas TOML (Witness, Refinery, Deacon), no código Go; hay código Go de merge/handlers sin llamador en producción.
- Veredicto: arquitectura ambiciosa y coherente en el diseño, con mucha deuda de acoplamiento a subprocesos (`bd`, `gt`, tmux, git) y código muerto o duplicado; difícil de razonar de punta a punta.

## 2. Mapa de módulos y capas

Medición: `find`+`cat | wc -l` sobre `*.go` (src = no `_test.go`). Total: 663 ficheros src / 248.970 líneas; 569 test / 225.826 líneas. "Depende de" sale de `grep` de imports internos por directorio (solo nivel 1).

| Módulo | Responsabilidad | Ficheros/líneas src | Depende de (principal) |
|---|---|---|---|
| cmd/gt, gt-proxy-server, gt-proxy-client | binarios | 1/12; 2/290; 1/137 | internal/cmd; proxy |
| internal/cmd | CLI cobra: 111 `rootCmd.AddCommand`, sling, done, convoy, rig, polecat... | 253/103.691 | 55 paquetes (casi todos) |
| internal/doctor | comprobaciones `gt doctor` | 75/20.484 | 26 |
| internal/daemon | daemon: heartbeat, Dolt, Boot/Deacon/Witness/Refinery, dogs | 36/13.456 | 28 |
| internal/beads | envoltorio de `bd` + store in-process | 30/12.246 | config, telemetry, util |
| internal/config | presets de agente, roles, loader, env | 10/8.577 | constants, util |
| internal/doltserver | servidor Dolt por town, migración de wisps | 9/7.733 | - |
| internal/web (+tui, ui) | dashboard web / TUI | 7/6.659; 14/4.319 | beads, mail |
| internal/polecat | gestor y sesión de polecats | 12/6.151 | 15 |
| internal/tmux | envoltorio tmux | 14/5.581 | - (18 importadores) |
| internal/mail | mail sobre beads | 8/5.351 | beads, nudge, tmux |
| internal/witness | patrullas y handlers del Witness | 7/5.201 | 17 |
| internal/refinery | motor de merge (engineer, batch) | 11/5.091 | 15 |
| internal/git, rig, crew, session, deacon, dog, formula, hooks, plugin, quota, scheduler, proxy, acp... | auxiliares | 1-12 ficheros cada uno | variable |

Fan-in más alto (de mi grafo): util 24, config 23, constants 22, tmux 18, beads 15, session 14. Fan-out más alto: cmd 55, daemon 28, doctor 26, witness 17. Hojas sin deps internas: activity, atomicfile, constants, estop, formula, git, mq, reaper, scheduler, telemetry, townlog, etc.

Capas implícitas: (a) infraestructura (tmux, git, beads, doltserver, atomicfile, lock), (b) dominio (polecat, witness, refinery, deacon, mail, convoy, formula), (c) orquestación (daemon, scheduler), (d) presentación (cmd, web, tui).

## 3. Fronteras y acoplamiento

- **Ciclos**: Tarjan sobre el grafo de paquetes de primer nivel: 0 componentes fuertemente conexos. Salvedad: ignoré subpaquetes (agent/provider, scheduler/capacity, config/roles...). El daemon evita el ciclo con cmd ejecutando `gt` como subproceso: "Shells out to `gt scheduler run` to avoid circular import between daemon and cmd" (internal/daemon/daemon.go ~3080, función dispatchQueuedWork en 3083-3098 usa `exec.CommandContext(ctx, "gt", "scheduler", "run")`, 3086). Coste: depende de `gt` en PATH.
- **Módulos "dios"**: ficheros >2.000 líneas: 18; >1.000: 47. Los mayores: doltserver.go 4.646, tmux.go 4.454, witness/handlers.go 3.577, git.go 3.573, polecat/manager.go 3.150, daemon.go 3.098. Funciones: 48 de más de 200 líneas y 234 de más de 100 (awk propio sobre `^func`/`^}`).
- **Frontera a subprocesos**: beads se invoca con `exec.CommandContext(ctx, "bd", ...)` (internal/beads/beads.go:794, 866) con timeout, reintento sin `--flat` (beads.go:817-830) y telemetría. En paralelo existe un store in-process opcional (internal/beads/store.go:1-8, "eliminates ~600ms per operation"). Resultado: dos rutas de acceso al mismo dato. 181 ficheros src usan `exec.Command`; 97 importan tmux.
- **Acceso SQL directo a Dolt** (driver mysql) en 10 ficheros no-test, p. ej. internal/beads/config_sql.go:11, internal/doltserver/doltserver.go, internal/daemon/compactor_dog.go: un tercer camino.
- **Guarda arquitectónica**: internal/beads/bd_subprocess_policy_test.go:16 impide `bd` ad hoc en deacon, plugin, refinery y witness (test AST).
- **Constantes duplicadas**: el tipo `Role` y `RoleMayor` está definido en internal/cmd/prime.go:55, internal/constants/constants.go:250 e internal/session/identity.go:13. Nombres de rol aparecen en 80 ficheros src (`RoleWitness|"witness"`).
- **Cruce de capas**: `internal/cmd` contiene lógica de dominio (done.go push, MR, checkpoints; sling.go spawn, hook, rollback) que debería estar en polecat/refinery.
- **Duplicación declarada**: executeSling (batch y scheduler) y runSling (CLI) son dos implementaciones del despacho; "TODO(scheduler-unify): Migrate single-sling rig dispatch to use executeSling()" (internal/cmd/sling.go:698; sling_dispatch.go:60-63).

## 4. Estado y persistencia

| Estado | Dónde | Quién escribe | Cita |
|---|---|---|---|
| Beads (issues, MR, convoyes, agent beads, mail) | Dolt SQL server, un servidor por town, puerto 3307 | `bd` por subproceso (gt/daemon/agentes) y store in-process | internal/doltserver/doltserver.go:140 (DefaultPort=3307); docs/design/dolt-storage.md:13 |
| Estado de agente / hook | agent bead (campos), completion metadata | `gt done`, sling | internal/beads/beads_agent.go:50-56, 619-623 |
| Checkpoints de `gt done` | etiquetas en agent bead | gt done | internal/cmd/done.go:2164-2166, 2172, 2186 |
| Config town/rig | settings/config.json, mayor/rigs.json, daemon.json, agents.json | gt install/config | internal/config/types.go:37-38, 635-637 |
| E-stop | fichero ESTOP en raíz del town | gt estop | internal/estop/estop.go:1-10 |
| Estado del daemon | daemon/daemon.lock (flock), PID, logs | daemon | internal/daemon/daemon.go (Run, ~452-470) |
| Sesión/heartbeat de polecat | ficheros de heartbeat + env de tmux | cada comando gt | internal/polecat/session_manager.go (TouchSessionHeartbeat) |
| Código | worktrees git por polecat, rama polecat/* | polecat | docs/design/architecture.md:135-146 (doc; no verificado en código) |

- **Transacciones**: no hay transacción entre pasos de sling/done; el rollback es por compensación best-effort (sling_dispatch.go:263-270, rollbackSlingArtifactsFn). Los commits Dolt son por operación de `bd`. No encontré en lo leído una transacción multi-bead.
- **Migraciones**: `gt upgrade` orquesta 5 pasos (doctor --fix, CLAUDE.md, daemon.json, hooks, formulas; internal/cmd/upgrade.go:80-96). Esquema versionado con enteros (CurrentTownSettingsVersion=1, CurrentTownVersion=2; config/types.go:38, 637). Migración de datos Dolt: MigrateAgentBeadsToWisps con copia a tabla legacy y verificación de conteo (internal/doltserver/wisps_migrate.go:47-54, 232, 254-259). No existe un framework de migraciones generales.

## 5. Flujo de control de una tarea, de punta a punta

1. El humano habla con el Mayor (agente LLM en tmux; plantilla internal/templates/roles/mayor.md.tmpl, que le indica `gt sling <bead-id> <rig>`: "Spawns polecat, hooks work, starts session", línea 216).
2. Opcional: encolado diferido; el heartbeat del daemon ejecuta `gt scheduler run` si no hay presión (internal/daemon/daemon.go: paso 14 de heartbeat, `checkPressure("polecat")` y `dispatchQueuedWork`, 3083-3098) y se llama a executeSling (internal/cmd/capacity_dispatch.go:723).
3. `executeSling` (internal/cmd/sling_dispatch.go:87-430): flock por bead (101), rig aparcado (118), estado del bead (129-158), quema moléculas obsoletas (208-233), spawn de polecat (250), convoy automático (271-284), cook e instanciación de fórmula (288-348), flock por assignee y hook con reintento (375-396), evento, estado del agente, campos del bead (401-410), arranque de sesión (418).
4. `SpawnedPolecatInfo.StartSession` (internal/cmd/polecat_spawn.go:390) llama a `SessionManager.Start` (internal/polecat/session_manager.go:320): valida el issue, resuelve el runtime (config.ResolveRoleAgentConfig), asegura settings por rol, construye comando y entorno (GT_ROLE, BD_ACTOR...), crea la sesión tmux con `NewSessionWithCommandAndEnv` (~481), instala pane-died hook, espera readiness, entrega nudge de arranque y valida que `GT_AGENT` esté definido (~591).
5. El polecat (CLI de agente) arranca, `gt prime` carga contexto, ejecuta el trabajo (fórmula mol-polecat-work.formula.toml).
6. `gt done` (internal/cmd/done.go:668-1980): escribe etiqueta done-intent (915), push de la rama con fallback a bare repo (742-757), verifica el commit remoto (779), crea el MR bead con `bd.Create` (1090), y en `notifyWitness` (1881) hace nudge al Refinery, escribe completion metadata en el agent bead, y `nudgeWitness(... "POLECAT_DONE ... exit=...")` (1933). Estrategias alternativas: direct y local (515-576).
7. El Refinery es un agente LLM que sigue mol-refinery-patrol.formula.toml: checkout, merge, tests, push (líneas ~356-407, 610-630 de la fórmula) y usa `gt mq ...` para estado/post-merge. El motor Go (`Engineer.doMerge`, ProcessBatch) no tiene llamador en producción, ver sección 8.
8. El Witness (LLM) corre `gt patrol scan` (internal/cmd/patrol_scan.go:162-168) que llama DetectZombiePolecats, DetectStalledPolecats, DiscoverCompletions.
9. Cierre: el polecat queda retirado tras `gt done` (done.go ~1936-1960) y el slot se libera para el scheduler.

## 6. Puntos de extensión

- **Harness/proveedor de agente**: añadir una entrada a `builtinPresets` (internal/config/agents.go:232) + constante (agents.go:22-45), más instalador de hooks opcional en internal/hooks/templates/<agente>/ (9 plantillas de agente: claude, codex, copilot, cursor, gemini, omp, opencode, pi, vibe). También por usuario, sin recompilar: JSON de registro cargado por loadAgentRegistryFromPathLocked (agents.go:575-613) con ruta DefaultAgentRegistryPath. Estimación: 1 fichero (data) + 1-2 si necesita hooks. **Pero** el comentario "No provider-string switch statements should exist outside this registry" (agents.go:~62) no se cumple: switch por nombre en internal/agentlog/event.go:54, internal/cmd/down.go:781, internal/doctor/orphan_check.go:427,435, internal/util/orphan.go:412, config/types.go:923-925, y 14 comparaciones `== "claude|codex|..."` en 8 ficheros. Un agente nuevo exigiría revisar esos sitios.
- **Rol**: internal/config/roles/<rol>.toml (embed, roles.go:12-13) + internal/templates/roles/<rol>.md.tmpl + constantes duplicadas en 3 sitios + referencias en ~76-80 ficheros. Coste alto.
- **Herramienta**: no hay registro de herramientas propio; los agentes usan `gt`/`bd` por CLI y las herramientas de su harness. Se añade un subcomando cobra (1 fichero en internal/cmd + registro vía init()). Comandos slash por agente en internal/templates/commands (provision.go).
- **Plugin periódico**: directorio con plugin.md (frontmatter TOML: gate, tracking, execution) y run.sh (plugins/rebuild-gt/plugin.md:1-17); descubrimiento en internal/plugin/scanner.go:26-50 con override rig sobre town; ejecución por el handler del daemon (internal/daemon/handler.go:249-271).
- **Canal**: no hay interfaz de canal externa (Slack/Telegram) en lo leído; `grep -ril slack|discord|telegram` en src solo da escalate_impl.go, config/types.go y daemon.go. Hay canales de mail/anuncios (internal/channelevents, mail_channel.go). Para añadir un canal hay que modificar escalate/daemon.
- **Fórmula**: 48 ficheros en internal/formula/formulas/ (TOML embebido); overlays/directivas por town/rig sin recompilar (docs/design/directives-and-overlays.md; no leído).

## 7. Comportamiento ante fallos a mitad

- **Daemon**: lock exclusivo por flock (daemon.go:~452-470); si hay shutdown (`isShutdownInProgress`) o E-stop activo, el heartbeat se salta (daemon.go:853-868). Heartbeat de recuperación por defecto 3 min (internal/config/operational.go:48).
- **Dolt caído**: DoltServerManager reinicia con backoff exponencial (5 s hasta 5 min), máx. 5 reinicios por ventana de 10 min y luego escala por mail al Mayor (internal/daemon/dolt.go:62-95, 511-545). Check propio de 30 s. Aviso: `DefaultDoltServerConfig` pone `Enabled: false` y `Port: 3306` (dolt.go:86-87) mientras el resto del código usa 3307; `normalizeDoltServerConfig` lo sobrescribe si hay endpoint gestionado (dolt.go:170-186). No concluyente qué ocurre sin endpoint gestionado.
- **Agentes caídos (Deacon, Witness, Refinery, Mayor)**: el heartbeat los relanza (`ensure*Running`, daemon.go:1463, 1723, 1800, 1900) con RestartTracker: backoff 30 s a 10 min, crash-loop a 5 reinicios en 15 min (internal/daemon/restart_tracker.go:44-52).
- **Deacon colgado**: heartbeat obsoleto 5-20 min, nudge (solo si hay trabajo activo); más de 20 min, kill y reinicio (daemon.go:1525-1640).
- **Polecat caído**: el daemon solo avisa; "Restart deferred to stuck-agent-dog plugin" (la nota previa lo cita en daemon.go:2742-2744; no lo reabrí). El Witness detecta zombis con `DetectZombiePolecats` (witness/handlers.go:1627) y maneja reinicio (2110).
- **Crash dentro de `gt done`**: etiqueta done-intent escrita antes del push (done.go:915; 2109-2125), checkpoints "pushed", "mr-created", "witness-notified" (done.go:2164-2166) permiten reanudar sin repetir push (714-717). Si el push o el MR fallan, se conserva el hook para que el Witness recupere (done.go:1928-1930 comentario; 1016, 1105).
- **Fallo a mitad de sling**: rollback por compensación (sling_dispatch.go:263-270, 377, 393, 421); los pasos secundarios solo imprimen warning (409). Idempotencia solo con target inequívoco (sling_idempotency.go:43-50, según nota previa; no reabierto).
- **Arranque de sesión**: si el comando muere o `GT_AGENT` falta, se mata la sesión y se devuelve error (session_manager.go ~575-595).
- **E-stop**: sentinel ESTOP; el daemon no reinicia agentes (daemon.go:859-863); en Windows el congelado devuelve error (nota previa: estop_windows.go:17-21; no reabierto).

## 8. Decisiones de diseño notables y su coste

1. **Agentes LLM como supervisores** (Witness/Refinery/Deacon con fórmulas). Ventaja: flexibilidad. Coste: comportamiento no determinista y consumo de tokens. Evidencia: refinery.md.tmpl:86 "You are the decision maker. All merge/conflict decisions are made by you, not Go code".
2. **Código Go sin llamador en producción** (comprobado por grep de llamadores, excluyendo tests): `Engineer.ProcessMRInfo`, `ProcessBatch`, `AssembleBatch` (refinery/batch.go:58, 211; engineer.go:1321); `doMerge` solo lo llaman ProcessMRInfo (engineer.go:1349) y batch.go:325. `gt refinery` solo usa ClaimMR/ListReadyMRs (internal/cmd/refinery.go:627, 748). `witness.HandlePolecatDone`, `HandlePolecatDoneFromBead`, `HandleLifecycleShutdown`, `HandleHelp`, `HandleSwarmStart`, `EscalateRecoveryNeeded`: 0 referencias no-test. Paquetes enteros sin ningún importador en el repo: internal/protocol (1.374 líneas src), internal/connection (689), internal/github (421), internal/keepalive (137), internal/mq (58), internal/agent/state.go (62). "No encontrado en lo leído" un uso por reflexión o plantilla; las búsquedas fueron por nombre de import y de símbolo.
3. **Completado por nudge+bead en vez de mail**: done.go:1933 y beads_agent.go:54,621 ("instead of POLECAT_DONE mail (nudge-over-mail redesign, gt-x7t9)"). Corrige la nota previa, que atribuía el aviso a mail vía `HandlePolecatDone` (handlers.go:153), que no tiene llamador. docs/design/mail-protocol.md:12-30 sigue describiendo POLECAT_DONE como mail.
4. **bd como subproceso**: aísla de beads, pero ~600 ms por operación (beads/store.go:4) y 150 sitios con "bd" literal en src.
5. **Daemon por subprocesos** (gt scheduler run) para evitar ciclo: simple, frágil por PATH.
6. **Comentarios de pasos desfasados**: la cabecera de executeSling lista 12 pasos con "11. Create Dolt branch" (sling_dispatch.go:74-86), pero el código tiene 11 pasos y ninguno crea rama Dolt.
7. **Windows nativo degradado** (INSTALLING.md:90-101 según nota previa); existe soporte psmux (nota previa).

## 9. Cobertura

- **Leído directamente (líneas abiertas)**: cmd/gt/main.go entero; sling_dispatch.go 60-431; polecat_spawn.go 380-470 y lista de funciones; session_manager.go 320-626 (filtrado por líneas de llamadas); done.go 1881-1940 y 2109-2190 más extractos filtrados de 668-1980; daemon.go 446-470, 853-1002, 1525-1640, 3083-3098; daemon/dolt.go 55-100, 155-190, 505-575; restart_tracker.go 30-60; estop.go 1-30; deacon/stuck.go 14-24; patrol_scan.go 130-200; config/agents.go 20-62, 545-615; config/roles.go 1-60; beads.go 755-850; beads/store.go 1-50; config_sql.go 1-40; mail/router.go 1-30; witness/manager.go 1-60; plugin/scanner.go 1-50; plugins/rebuild-gt/plugin.md 1-40; proxy main.go 28-36; ci.yml, .golangci.yml, Makefile 1-140, .githooks/pre-push 1-80, CONTRIBUTING (casi entero), codecov.yml, renovate.json.
- **Estimación**: unas 3.500-4.500 líneas de código leídas de forma continua, es decir ~1,5 % de las 248.970 líneas src (~0,8 % si cuento tests); ~45 de 1.232 ficheros .go (~3,6 %). El resto se cubrió por búsqueda dirigida (grep) sobre el 100 % de los ficheros: grafo de imports, callers, TODO, tamaños.
- **No visto**: cuerpo de runSling (200-1156), engineer.go (solo índice de funciones y callers), doltserver.go, tmux.go, git.go, convoy*, rig/manager.go AddRig, doctor/ (75 ficheros), web/, tui/, acp/, quota/, krc/, wasteland/, telemetry/, formulas TOML (solo greps), templates de rol (solo greps), docs de diseño (solo cabeceras y las comparadas).
- **Afirmaciones de la nota anterior comprobadas**: confirmadas (stuck.go:19-21; LoadStuckConfig sin llamadores; daemon heartbeat; refinery sin llamadores para ProcessBatch; proxy 0.0.0.0:9876 en cmd/gt-proxy-server/main.go:31; `GT_PROXY_*` solo se lee en cliente y exec.go, no se fija al lanzar polecats). Corregidas: ver punto 3 de la sección 8 (aviso de done por mail) y que `handlers.go:153` no es ruta viva. Dato añadido: SECURITY.md:56 declara "0.1.x" como versión soportada mientras Version="1.2.1" (internal/cmd/version.go:17).
