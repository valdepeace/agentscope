# agent-orchestrator: estudio de ARQUITECTURA

Repo: `OrchestratorInc/agent-orchestrator` (clon superficial, sin historial). Rutas relativas a la raíz del repo. Nada se ejecutó; contadores con `git ls-files`, `wc`, `git grep`. Complementa y corrige `study/agent-orchestrator.md`.

## 1. Resumen y veredicto

- Daemon Go local (módulo `github.com/aoagents/agent-orchestrator/backend`, `backend/go.mod:1-3`) + app Electron/React + app móvil. Cada tarea es una sesión con worktree y rama propios.
- Núcleo: `session_manager` (spawn/kill/restore), `lifecycle` (reglas y nudges), `observe/*` (sondas que solo reportan hechos), SQLite con migraciones goose.
- La separación por puertos (`ports`, `domain`) es real y se cumple en gran parte, pero hay fugas concretas documentadas abajo (adapters importando core/storage; storage importando service).
- Puntos débiles: ficheros "dios" (`session_manager/manager.go` 8382 líneas, `daemon.Run` ~1550 líneas) y añadir un harness toca ~25 ficheros de producción más una reescritura del CHECK de SQLite.
- Veredicto: arquitectura cuidada y con buen manejo de fallos a mitad, pero con coste alto de cambio en los bordes y varios módulos monolíticos.

## 2. Mapa de módulos y capas

Ficheros/líneas: fuente / tests, medidos con `git ls-files` + `wc -l` (los "gen" van aparte en la nota). Dependencias: import graph calculado con `grep` de imports en ficheros no `_test`.

| Módulo | Responsabilidad | Fuente (ficheros/líneas) | Tests (ficheros/líneas) | Depende de |
|---|---|---|---|---|
| `backend/internal/domain` | vocabulario, IDs, enums, registros | 54 / 9135 | 30 / 2450 | hoja (solo stdlib) |
| `backend/internal/ports` | interfaces core-adaptadores | 32 / 5875 | 4 / 187 | domain |
| `backend/internal/session_manager` | spawn, kill, restore, reconcile, envío | 52 / 20567 | 97 / 39667 | domain, ports, adapters/agent, service(2), sessionguard, process... |
| `backend/internal/lifecycle` | LCM: facts a estado, nudges, reaper hooks | 10 / 4288 | 17 / 9051 | domain, ports, sessionguard, deliverytelemetry |
| `backend/internal/observe/*` | sondas: scm, reaper, activity, usage, automations | 28 / 10785 | 46 / 16762 | domain, ports, service(usage) |
| `backend/internal/service/*` (32 subpaquetes) | casos de uso para HTTP/CLI | 246 / 94707 | 361 / 125302 | domain, ports, httpd(apierr), session_manager |
| `backend/internal/storage/sqlite` | store, migraciones, CDC | 59 / 22919 (+23475 gen) | 176 / 37164 | domain, ports, cdc, service(shellterm) |
| `backend/internal/adapters/agent` | un paquete por harness | 332 / 55429 | 257 / 59357 | ports, domain, process |
| `backend/internal/adapters/chatdriver` | controladores Chat (ACP, Codex app-server) | 108 / 26509 (+5365 gen) | 131 / 37455 | ports, domain |
| `backend/internal/adapters/{runtime,scm,workspace,tracker,reviewer,...}` | PTY/ConPTY/tmux, GitHub/GitLab, gitworktree | ~120 / ~25000 | ~100 / ~33000 | ports, domain |
| `backend/internal/httpd` | API HTTP chi + OpenAPI | 85 / 31054 | 111 / 33098 | service(61 imports), domain, ports |
| `backend/internal/cli` | CLI Cobra | 59 / 16934 | 64 / 21839 | adapters, config, daemon... |
| `backend/internal/daemon` | cableado y arranque | 34 / 6150 | 39 / 5591 | casi todo (53 imports de service, 14 de storage) |
| `frontend/src/renderer` | UI React | 892 / 216588 | 662 / 202822 | api generada |
| `frontend/src/main` + `main.ts` | proceso Electron | 124 / 36555 (+5101 main.ts) | 113 / 40441 | — |
| `packages/mobile` | app Expo | 727 / 90268 | 473 / 59132 | cloud-client, shared |
| `packages/{product-ui,cloud-client,shared}` | libs compartidas | 40 / ~10000 | 14 / ~5100 | — |

Capas declaradas: CLI/HTTP, Services, Core (session_manager, lifecycle, observe), Data (domain, ports, storage, cdc), Infra (adapters, terminal, daemon, config) (`docs/backend-code-structure.md:19-62`). Reglas: dependencias solo hacia abajo y "Adapters are leaves" (`docs/backend-code-structure.md:73,107-109`; `docs/architecture.md:1364`).

## 3. Fronteras y acoplamiento

Método: conté imports `.../backend/internal/<pkg>` por paquete de primer nivel (no `_test`).

- Fan-in: `domain` lo importan 30 paquetes, `ports` 16, `process` 16. Los adaptadores usan sobre todo ports (328 imports) y domain (176); correcto según el diseño.
- **Violaciones de la regla "adapters son hojas"** (comprobadas leyendo la línea del import):
  - `backend/internal/adapters/scm/multi/provider.go:14` importa `observe/scm`.
  - `backend/internal/adapters/telemetry/localsqlite.go:13` importa `storage/sqlite/store`.
  - `backend/internal/adapters/reviewer/agy/agy.go:17` y `.../pi/pi.go:17` importan `reviewgateway`.
- **Storage importa service**: `backend/internal/storage/sqlite/store/shell_terminal_store.go:10` importa `service/shellterm`.
- **Observe importa service**: `backend/internal/observe/usage/ingestor.go:17` (y `legacy_repair.go:15`, `sqlite_source.go:16`) importa `service/usage`.
- **Service importa httpd**: ~74 imports, los vistos son `httpd/apierr` (p. ej. `service/usage/report.go`, `service/shellterm/service.go`). Es un paquete de errores, pero la capa de servicio queda atada a un paquete HTTP. `telemetrymeta/errors.go:20` también importa `httpd/apierr`.
- **Session_manager importa service**: `session_manager/session_import.go:25` (`service/sessionimport`) y `terminal_fork.go:12` (`service/sessionfork`), contra "Services → Core".
- Ciclos de paquetes Go: no existen (el compilador los impide). A nivel de directorios de primer nivel sí hay ida y vuelta (service↔httpd, storage↔service) resuelta por subpaquetes distintos.
- Módulos "dios" (por líneas, no generados): `session_manager/manager.go` 8382; `storage/sqlite/store/conversation_store.go` 5681; `httpd/controllers/dto.go` 4826; `service/chat/controller.go` 4589; `httpd/apispec/specgen/build.go` 4542; `observe/scm/observer.go` 2478; `lifecycle/manager.go` 1997; `storage/sqlite/db.go` 2621. `daemon.Run` ocupa `daemon/daemon.go:174-1724` (una función de ~1550 líneas).
- Cableado por setters: `session_manager/manager.go:648-744` y `lifecycle/manager.go:337-360` exponen una docena de `SetX` posteriores a la construcción; el grafo de dependencias se completa en `daemon.Run`, no en los constructores.

## 4. Estado y persistencia

- Un único SQLite `ao.db` en el data dir (`storage/sqlite/db.go:59-60`). Pragmas: WAL, `busy_timeout(5000)`, `foreign_keys(ON)`, `synchronous(NORMAL)` (`db.go:43-47`); hasta 8 lectores (`db.go:54`).
- Candado de data dir antes de abrir la BD, para que dos daemons no migren a la vez (`daemon/daemon.go:262-290`). Run-file con comprobación de `/healthz` (`daemon.go:248-260`).
- Migraciones: **263 ficheros** en `storage/sqlite/migrations` (189 numerados `NNNN_` y 74 con marca de tiempo `2026...`), 262 con sección Down. Se aplican con `goose.Up(..., WithAllowMissing())` (`db.go:478`). Corrige la nota anterior ("~215").
- Antes de goose corren ~22 funciones `repair*/prepare*` en `db.go` (conteo `grep -cE '^func (repair|prepare)'`) que reescriben historiales renumerados (`db.go:441-471`, `489-513`), y tras goose `reconcileSchema` (`db.go:481`). `AGENTS.md:118` prohíbe tocar migraciones ya fusionadas; las reparaciones existen porque hubo ramas con migraciones renumeradas.
- CHECK de harness: SQLite no puede alterar un CHECK, así que cada harness nuevo reescribe `sqlite_master` con `PRAGMA writable_schema` (`db.go:2303-2557`; migración tipo `20261119090000000_allow_mastra_code_harness.sql`).
- Transacciones: `db.Begin()` en las rutas de migración (`db.go:559,660,724,...`); en `store/` hay ~97 usos de `withTx/inTx/BeginTx` repartidos en 27 ficheros. La dedup de nudges NO es transaccional: JSON `{seen,attempts}` en `pr.last_nudge_signature` (`migrations/0005_pr_last_nudge_signature.sql`) protegido por mutex en memoria (nota anterior, §13; `lifecycle/reactions.go:1043-1044`).
- Quién escribe: el estado de sesión solo se muta por `lifecycle.Manager.mutate` (`lifecycle/manager.go:417`); el reaper "solo reporta hechos" y nunca escribe filas (`observe/reaper/reaper.go:1-6`). El observer SCM persiste con `WriteSCMObservation` (`observe/scm/observer.go:745`) y luego avisa al lifecycle (`:759`).
- CDC: triggers a `change_log`, poller y broadcaster a SSE (`daemon/daemon.go:292-295`; `docs/STATUS.md:30-37`).
- Estado fuera de SQLite: worktrees y todo bajo `~/.ao` (`CLAUDE.md`, `AGENTS.md:125`).

## 5. Flujo de control de una tarea, de punta a punta

1. Petición de spawn (HTTP/CLI/automation/tracker intake) llega a `service/session` y de ahí a `session_manager.Manager.Spawn` (`session_manager/manager.go:1110`).
2. Idempotencia por `ClientRequestID`: si existe, devuelve la sesión previa (`manager.go:1121-1130`).
3. Carga el proyecto y extiende el presupuesto de tiempo con el `PostCreate` (`manager.go:1131-1137`); resuelve harness efectivo por rol (`:1172`) y toma un permiso de uso de harness (`:1176`).
4. Inserta una fila semilla (`CreateSession`, `manager.go:1346`); construye prompt y system prompt (`:1408,1422`); crea o promueve el worktree (`:1434-1561`).
5. Prepara hooks/lanzamiento y supervisión del proceso (`manager.go:1742-1753`), reserva el lanzamiento en el LCM (`PrepareLaunch`, `:1751`).
6. Crea el runtime (PTY/ConPTY/tmux o controlador Chat): `m.runtime.Create` (`manager.go:1763`).
7. `lcm.MarkSpawned` confirma la sesión (`manager.go:1809`); si el prompt va después del arranque, `deliverAfterStartPrompt` (`:1816`); cierra el `ClientRequest` (`:1829`).
8. En marcha, los hooks del agente llegan como `ApplyActivitySignal` (`lifecycle/manager.go:567`) y el reaper reporta liveness cada 5 s (`reaper.go:23`) vía `ApplyRuntimeObservation` (`lifecycle/manager.go:463`).
9. El observer SCM sondea cada 30 s (`observe/scm/observer.go:32`), escribe el PR y llama `ApplySCMObservation` (`observer.go:745-759`, `lifecycle/reactions.go:476`), que deriva a `ApplyPRObservation` (`reactions.go:480`) y a `sendOnce` (`reactions.go:1039`): los nudges (CI, review, conflicto) vuelven al worker dueño.
10. Fin: merge del PR sin otros abiertos termina la sesión (nota anterior, `reactions.go:87-118`) o Kill (`manager.go:3061-3072`); `MarkTerminated` (`lifecycle/manager.go:1795`) y limpieza de contenedores (`:1871`).

Llamadores comprobados: `ApplySCMObservation` tiene 3 llamadores en código no test (`observe/scm/observer.go:759`, `service/pr/action_service.go:301,674`); `service/pr/manager.go:42-53` es el camino alternativo `ApplyObservation` (escribe y reenvía a `ApplyPRObservation`).

## 6. Puntos de extensión

- **Harness nuevo.** El comentario de `adapters/agent/registry/registry.go:1-5,107-109` dice que añadir un harness es "una sola edición" (constructor + constante). Falso en la práctica. Tomando `mastra-code`, `git grep -il` da 57 ficheros; ~35 no son tests ni docs y tocan:
  - adaptador `adapters/agent/mastracode/` (11 ficheros), `registry/registry.go:98`, `domain/harness.go:46,66`;
  - `activitydispatch/dispatch.go`, `modelcatalog/catalog.go`, `session_manager/manager.go:2766-2768`, `cli/spawn.go:276`, `cli/hooks.go:983`, `cli/doctor.go`;
  - `service/agentauth/{plans,provider_env,service}.go`, `service/systeminstall/{agentplans,systeminstall}.go:89,112`, `service/shellagent/classify.go`;
  - `httpd/controllers/dto.go:696,1530` (listas `enum:"..."`), `httpd/apispec/openapi.yaml`, `frontend/src/api/schema.ts`;
  - `storage/sqlite/db.go:2330-2618` + una migración SQL de 84 líneas;
  - frontend (5 ficheros), móvil (4) y `.github/workflows/go.yml:140`.
  Interfaz mínima: `ports.Agent` con 6 métodos (`ports/agent.go:225-248`) más interfaces opcionales.
- **Rol.** `SessionKind` = worker|orchestrator (`domain/session.go:20-25`); override de harness por rol en proyecto (`session_manager/manager.go:2718-2750`). Un rol nuevo exige tocar dominio, prompt (`session_manager/prompt.go`) y esquema; no hay registro de roles.
- **Herramienta.** No hay registro de herramientas propio: el agente usa el CLI `ao` y el servidor MCP (`backend/internal/mcpserver`, 6 ficheros). No leí su contenido: "no encontrado en lo leído" para un mecanismo de plugins de herramientas.
- **Canal/proveedor.** SCM y Tracker son puertos con implementaciones `github`, `gitlab`, `multi` (`docs/backend-code-structure.md`, tabla de puertos); notificaciones en `notify` y `push`. No medí cuántos ficheros exige un proveedor nuevo.
- Registro de adaptadores: `adapters/registry.go` (83 líneas, `Manifest` con capacidades agent|issue-tracker).

## 7. Comportamiento ante fallos a mitad

- **Fallo en spawn**: cada etapa devuelve un sentinela (`wrapSpawnStage`, `session_manager/manager.go:178`) y hace rollback del worktree/runtime (`:1744-1772,2580-2700`); si falla `MarkSpawned` destruye runtime, intenta destruir workspace y marca terminada (`:1809-1813`). Un fallo en `CommitClientRequestSession` (`:1829`) no hace rollback: deja la sesión viva con error.
- **Crash del daemon**: al arrancar, `ReconcileStartupSafety` cierra transiciones de interfaz interrumpidas y marca "provisioning" huérfanos como fallidos ("Orchestrator restarted before this session finished starting"), **antes** de abrir el puerto (`manager.go:4787-4821`, `4858-4859`; llamada en `daemon/daemon.go:1150`; si falla, el daemon aborta `:1157`).
- Tras abrir el listener, en segundo plano: hosts de chat persistentes, reviewers, `ReconcileBackground` y `ReconcileRuntime`; solo entonces arranca el reaper (`daemon.go:1491-1514`). Un runtime muerto mientras AO estaba caído se juzga en esa comprobación, no se lee como crash (`reaper.go:41-46`).
- `ReconcileBackground` **no relanza agentes**: solo chequea salud (`checkSessionHealth`, `session_manager/startup_health.go:17`), limpia preparaciones interrumpidas y entrega mensajes de transición (`manager.go:4826-4883`). Relanzar exige Resume/Restore explícito (`manager.go:4774-4777`); `reconcileLive` (`:4581`) solo se llama desde resume (`:3870`).
- **Sonda fallida != muerte**: error de sonda deja la sesión como está (`manager.go:4622-4624`); el reaper termina solo con runtime claramente muerto, sin mutación en curso (`lifecycle/manager.go:493-498`) y repite la comprobación bajo la mutación (`:512-516`). Freno anti-terminación masiva: >=5 sesiones y >=50 % muertas en un ciclo (`reaper.go:26-34`).
- **Timeout**: spawn con presupuesto propio y relabel de timeout (`manager.go:1113-1117`, `spawn_timeout.go`); lifecycle del observer 30 s para PRs abiertos y sin tope para merged/closed (`observe/scm/observer.go:751-758`). Si falla el aviso al lifecycle, no hay acuse y se reintenta en el siguiente poll (`:761-766`), es decir at-least-once.
- **Apagado**: `stop()` cancela workers y espera en orden con timeouts por etapa (`daemon.go:1636-1720`); el candado espera hasta 8T+I+18 s (`daemon.go:272-284`). Al salir por cierre de escritorio se paran agentes; en reemplazo de daemon se desacoplan los providers (`daemon.go:1676-1692`).
- **Nudge sin techo para CI**: el firmante incluye commit y cola del log; un worker que falla en bucle recibe un nudge por push (nota anterior §13, `reactions.go:848-854,1062-1066`; no reverificado).
- Reaper de workers: solo liveness; el "atascado pero vivo" sigue sin acción automática (`no_signal` es etiqueta, nota anterior §13).

## 8. Decisiones de diseño notables y su coste

| Decisión | Beneficio | Coste (con cita) |
|---|---|---|
| Un único SQLite + CDC por triggers | un solo camino de escritura, SSE sencillo | 263 migraciones, 22 funciones de reparación, CHECK reescrito a mano (`db.go:2303-2557`) |
| Estado derivado en lectura, hechos persistidos | menos incoherencias | lógica de estado repartida en `pkg/contract/status.go`, `service/session/status.go` y lifecycle |
| Observers que solo reportan hechos; un único escritor (LCM) | fallos de sonda no corrompen estado | LCM con `ApplyActivitySignal` de ~600 líneas (`lifecycle/manager.go:567-1170`) |
| Puertos y adaptadores por harness (44 constantes en `domain/harness.go`) | catálogo amplio | alta de harness ~35 ficheros (§6) |
| Chat y TUI sobre las mismas sesiones, con controlador por generación | continuidad | `session_manager` de 8382 líneas y ~50 ficheros |
| Daemon local sin sandbox de SO | simplicidad | ver nota anterior §7; Codex en modo `default` equivale a bypass (nota anterior §13) |
| Cableado manual en `daemon.Run` | trazable | función de ~1550 líneas y setters tardíos |

## 9. Cobertura

- **Conteos (todo el repo, por comandos)**: 7950 ficheros (`git ls-files`); `.go` 3106, `.ts` 2011, `.tsx` 1308, `.sql` 305, `.md/.mdx` 213. Importes entre paquetes: leídos por `grep` en todo `backend/internal`.
- **Abiertos y leídos (rangos)**: `session_manager/manager.go` 1110-1180, 1330-1350, 1425-1470, 1740-1842, 4581-4670, 4770-4992; `daemon/daemon.go` 245-300, 1136-1160, 1470-1560, 1630-1722; `lifecycle/manager.go` 463-556; `session_manager/startup_health.go` 17-67; `observe/scm/observer.go` 25-40, 735-775; `observe/reaper/reaper.go` 1-80; `storage/sqlite/db.go` 30-60, 440-520, 2290-2335, 2530-2621; `service/pr/manager.go` 1-60; `adapters/agent/registry/registry.go`, `adapters/registry.go`, `ports/agent.go` 225-285; migraciones 0005 y 2026111909; `docs/backend-code-structure.md` 1-200; `.github/workflows/ci.yml`.
- **Estimación**: ~45 ficheros abiertos de 7950 (~0,6 %) y ~2500 líneas leídas de ~1,65 M de líneas de fuente+tests (<0,2 %). Con los ~50 ficheros de la lectura anterior sube a ~1,2 % de ficheros. La cobertura por búsqueda (imports, referencias, conteos) es total para `backend/`.
- **Sin ver**: frontend y móvil (solo conteos), contenido de `chatdriver`, `adapters/agent/*` (salvo registro), `service/*` salvo pr, `httpd` (solo imports), `mcpserver`, `private/ao-cloud` (submódulo privado), `contracts/`, y el interior de `ApplyActivitySignal`. Los puntos marcados "no reverificado" proceden de la nota anterior.

## Correcciones a la nota anterior

- Migraciones: 263 ficheros (no ~215); 189 numerados + 74 con timestamp.
- Constantes de harness: 44 (`grep -c 'AgentHarness = ' domain/harness.go`), no 45; no reverifiqué el número de drivers de chat (19).
- "Añadir un harness" no es una sola edición pese al comentario del registro (`registry.go:1-5`).
- El arranque no relanza agentes: relanzar es explícito (`manager.go:4774-4777`); la nota afirmaba "reconciliación de arranque... hosts readoptados" sin distinguir.
- Ficheros: 7950 con `git ls-files` (la nota daba 7978 por `find`).
