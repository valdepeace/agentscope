# Arquitectura: LofiSu/dsh-super-agent (paquete `ds-agent`)

Estudio de solo lectura sobre un clon superficial (1 commit) en `references/deepseek-super-agent/`. Rutas relativas a la raíz del repo. No se ejecutó nada del repo. Cobertura baja y dirigida (sección 9). Lo no abierto se marca como "no leído" o "heredado de la nota anterior" (`study/deepseek-super-agent.md`).

## 1. Resumen y veredicto

- Motor de dominio síncrono (`src/core`, clase `Engine`, una transacción SQLite por comando público: `src/core/engine.ts:27-31`) rodeado por un arnés asíncrono (`src/harness`) que ejecuta sesiones de modelo ReAct en el mismo proceso Node.
- La separación núcleo/arnés está bien pensada: puertos por rol impiden que un Worker llame a `reply`/`decide` (`src/core/engine.ts:54-63,194-195`, `src/harness/tools.ts:109-119`).
- El cableado se concentra en un módulo dios: `src/runtime.ts` (3231 líneas; `Runtime.create` va de la l.533 a ~2095, unas 1560 líneas en un método estático).
- Persistencia: SQLite con almacén genérico `documents(kind,id,payload JSON)`; sin savepoints, las transacciones anidadas se aplanan (`src/store/sqlite.ts:57-75`).
- Veredicto: dominio sólido y bien documentado para un autor único; deuda real en el tamaño de `runtime.ts` y `loop.ts` y en ciclos entre módulos. Un host, un proceso: no hay aislamiento de proceso entre Router, Worker y base de datos.

## 2. Mapa de módulos y capas

Ficheros y líneas por `git ls-files src/<dir>` + `cat | wc -l` (src total: 35 281 líneas, 172 ficheros .ts). "Depende de" sale de las importaciones relativas entre directorios (script propio; incluye `import type`).

| Módulo | Responsabilidad | Ficheros / líneas | Depende de |
|---|---|---|---|
| `src/runtime.ts` + `src/index.ts` | Cableado, ingress, recuperación, mesh remoto | 2 / 3293 | harness, env, channels, core, ingress, memory, metrics, gateway, shared, store, watchdog, speculate |
| `src/shared` | Tipos de dominio, puertos por rol, errores | 7 / 598 | store (1 arista; fichero no identificado) |
| `src/core` | Engine: intake, evals, tickets, assignments, collab, replies, efectos externos, recuperación | 21 / 2952 | shared (33 imports), speculate, memory, store, harness (tipo), channels, ingress, metrics |
| `src/store` | SQLite, esquema y migraciones, backup | 5 / 1419 | shared, core (tipo) |
| `src/harness` | Bucle ReAct, sesiones, herramientas, modelo, prompts, mesh, rutinas, proactivo | 20 / 7061 | shared, env, store, channels, core, metrics, memory, ingress, gateway, speculate |
| `src/env` | Entorno del Worker: workspace, sandbox, vault, sanitizado, agentes locales, browser errands | 8 / 2560 | shared |
| `src/ingress` | Filtro rápido, slow path, batch | 6 / 498 | shared |
| `src/memory` | Spill, ACL, grafo de refs, memoria a largo plazo, embedding | 7 / 1234 | shared, store |
| `src/watchdog` | Asignaciones atascadas, salud | 5 / 402 | harness (tipos), env, shared, store, metrics |
| `src/speculate` | Gate de entrega, trabajo especulativo | 5 / 117 | evolution, shared |
| `src/evolution` | Evolución controlada del DeliverGate | 27 / 3770 | env, speculate (tipo), shared |
| `src/channels` | GitHub, Feishu, iMessage, A2A HTTP, outbox | 13 / 3006 | store, shared, core |
| `src/gateway` | Servidor HTTP, config de proveedores | 17 / 2108 | harness |
| `src/cli` | CLI `ds-agent`, demos | 20 / 4400 | 14 módulos |
| `src/ui`, `src/metrics`, `src/measurement` | Panel, métricas, medición T48 | 2 / 749; 5 / 640; 3 / 474 | core, store, watchdog, memory |

Capas observadas: shared -> store/core -> harness/env/channels -> runtime -> cli/gateway/ui. Módulos que importan a cada uno: shared 14, store 11, core 7, metrics 6.

## 3. Fronteras y acoplamiento

- Módulo dios: `src/runtime.ts` importa de 12 directorios; constructor l.465, `static create` l.533, 36 métodos hasta l.3165. `src/harness/loop.ts` (1159 líneas) mezcla la clase `Harness` (l.134), la cola de mesh (l.356-537) y el bucle ReAct (l.1029).
- Frontera núcleo/runtime: los hooks de `Host` (`src/core/host.ts:6-32`: `onTicketCreated`, `onTicketEnded`, `onDecision`, `onAssignmentEnded`, `onAssessmentNeeded`) se implementan en `src/runtime.ts:1548-1637`. El Engine no conoce el arnés. Inversión limpia.
- Ciclos entre directorios:
  - core <-> channels: `src/core/replies.ts:1-2` importa de `channels/norms.ts` y `channels/outbox-adapter.ts`; `src/channels/outbox-adapter.ts:1` importa de `core/external-effects.ts`. Ciclo de módulo con valores en ambos sentidos; ciclo a nivel de fichero no comprobado.
  - core -> speculate -> evolution: `src/core/assignments.ts:3` -> `src/speculate/resolve-gate.ts:1` -> `src/evolution/promotion.ts`; la vuelta `src/evolution/strategy-gate.ts:2` es `import type`. `speculate/configurable-gate.ts:1,4` importa de evolution (tipo y valor).
  - Dependencias de tipo hacia arriba: `src/core/assessment.ts:2` -> harness/model; `src/store/sqlite.ts:4` -> core/lifecycle; `src/watchdog/watchdog.ts:1` y `health.ts:9` -> harness. Valor: `src/gateway/config.ts:11` -> harness/providers.
  - shared -> store: 1 arista, fichero no abierto; una capa `shared` que depende de `store` merece revisión.
  - core <-> metrics: `src/metrics/batching-bench.ts:2-3` importa Engine y Host.
- Cruce de capas: el ingress hace redacción, spill y enrutado a shards dentro de `Runtime.commitIngress` (`src/runtime.ts:2853-2896`), no en `src/ingress` (498 líneas). La admisión está repartida entre `src/ingress/*`, `Runtime.ingest` y `Intake.receive` (`src/core/intake.ts:35-37` acepta un veredicto precalculado en `trust.admission`).
- 50 ficheros de test importan `src/runtime.ts`/`index.ts` (regex sobre `from '../src/...'`).

## 4. Estado y persistencia

| Estado | Dónde vive | Quién escribe | Cita |
|---|---|---|---|
| Triggers, threads, tickets, decisiones, disposiciones, equipos, recovery | tabla `documents(kind,id,payload JSON)` | Engine (dentro de `atomic`) | `src/store/sqlite.ts:86-105`; `src/core/intake.ts:49-91` |
| Contadores de id | `id_seq` | `Store.nextId` | `src/store/sqlite.ts:77-84` |
| Spills, grants, jobs, sesiones, mensajes, replies, outbox, subscriptions, artifacts, events, lifecycle_events | tablas propias | Store / Runtime / harness | `src/store/schema.ts:6-97,143` |
| Journal de sesión | `messages` + ficheros en `<home>/sessions` | `SessionJournal` | `src/runtime.ts:1640` |
| Nudges del watchdog | memoria del proceso | watchdog | heredado (`src/watchdog/watchdog.ts:87-89`), no recomprobado |
| Cola de mesh, `activeWorkers`, `running` | memoria del proceso | Harness | `src/harness/loop.ts:246,374-390,527` |
| Memoria a largo plazo | `src/memory/long-term.ts` sobre Store | hooks `onTicketEnded` / `onDecision` | `src/runtime.ts:1588,1616` |

- Migraciones: lista ordenada versión 1 a 4 con ledger `schema_migrations` (`src/store/schema.ts:97-170`); una base de datos de versión superior falla cerrada antes de activar WAL (`src/store/sqlite.ts:30-32`). Existe `test/store-migration.test.ts` (solo el nombre).
- Transacciones: `atomic` hace `BEGIN` solo en nivel 0; anidado ejecuta `work()` sin savepoint (`src/store/sqlite.ts:57-75`). `Engine.recoverInterrupted` captura el error de un ticket dentro de la transacción exterior (`src/core/engine.ts:99-121`). Interpretación mía de ambos fragmentos: las escrituras parciales de un ticket que falle quedan confirmadas con las del resto. No lo ejecuté.
- Modelo documento: `putDoc` serializa el ticket entero con sus assignments e inputs (`src/core/assignments.ts:185`); `listDocs('ticket')` lee y parsea todos (`engine.ts:102`, `sqlite.ts:100-105`); `neighborTriggerRefs` lista todos los triggers (`src/runtime.ts:2951-2955`). Coste lineal en el histórico.

## 5. Flujo de control de una tarea, de punta a punta

1. Un canal o el CLI llama a `Runtime.ingest` / `ingestAsync` (`src/runtime.ts:2812,2824`): redacta el texto (l.2815), detecta duplicados (l.2841) y calcula el veredicto `filterTrigger` (rápido, y slow path con modelo si está activo; l.2816, 2829).
2. `commitIngress` (l.2853): métricas, inline o spill con Chain of Recall (l.2867-2881), shards del Router (l.2883), sidecar `trust` con `admission`, `strength`, `spillId` (l.2886-2893) y `engine.receive` (l.2896).
3. `Engine.receive` en una transacción (`src/core/engine.ts:48-51`) -> `Intake.receive` (`src/core/intake.ts:26`): duplicado (l.28), filtrado con disposición (l.48-54), reparto a tickets suscritos vivos como `trigger` o `context` (l.69-90), cola `pending` y `onQueued` si requiere evaluación (l.103-108).
4. Tras la ventana de batch (`scheduleEval`, `src/runtime.ts:2994-3005`), `flushEvals` llama a `engine.evaluateDue()`; con `accept` y ticket, `harness.launch(ticket)` (l.2979-2992). Si el Gate difiere, `onAssessmentNeeded` dispara `assessPending` (l.2074, 2258): turno del Router con `assess_trigger`.
5. `Harness.launch` (`src/harness/loop.ts:238`): comprueba pausa (l.239); turno del Router de cada shard primero (l.292-299); un ticket proactivo o un chat directo ya respondido no arranca Worker (l.268-285); si no, `startOrQueue` -> `startWorkerTurn` (l.509), que llama a `assignmentStarted` (l.529; `runtime.ts:1688-1691`) y encola la sesión del Worker.
6. Bucle ReAct: `maxSteps` 10 por defecto (l.1029), pausa comprobada en cada paso (l.1031) y antes de cada herramienta (l.1065). Un error de modelo termina el turno y deja la asignación `running` (l.1041-1053).
7. El Worker usa `report_to_router` (`src/harness/tools.ts:400`) -> `Engine.report` (`engine.ts:84`) -> `Assignments.report`: con ticket en pausa el informe se retiene con veredicto `hold` (`assignments.ts:98-115`); si es especulativo pasa por el gate (l.116-123); `ready`/`dropped` cierran ticket y asignación (l.175-178); `failed`/`blocked`/`waiting` solo la asignación (l.179-184). Se persiste y luego se avisa a observadores (l.185-188).
8. `onTicketEnded` (`src/runtime.ts:1586-1613`): memoria a largo plazo, liberación del hueco de mesh por microtarea (l.1544-1547), eventos de ciclo de vida. El Router responde con `reply_user` -> `Engine.reply` (`engine.ts:194-229`), que retiene la respuesta con pausa o ticket especulativo silenciado y si no la encola en outbox; el adaptador reintenta (`src/channels/outbox-adapter.ts:135-155`).

No leído en profundidad: el turno de Router al recibir un informe (heredado), `assessPending` (l.2258-2328) y el despacho remoto.

## 6. Puntos de extensión

| Qué | Cómo | Ficheros a tocar (mínimo) |
|---|---|---|
| Proveedor compatible OpenAI | Entrada en `PROVIDER_PRESETS` (`src/harness/providers.ts:39`) | 1 (+ test y doc) |
| Modelo no OpenAI | Implementar `Model` (`src/harness/model.ts`) y cablear en `runtime.ts` | no verificado; >= 2 |
| Herramienta | Objeto con `name`, `parameters`, `roles` en `src/harness/tools.ts` (filtrado por rol l.109-114; opcionales con spread condicionado l.102-105) | 1-3 |
| Herramientas externas | Config MCP (`src/runtime.ts:3166`); solo rol worker (`tools.ts:117`) | 0 de código |
| Canal | Fichero en `src/channels`, export en `channels/index.ts`, cableado en `src/cli/channels.ts`/`runtime.ts` (el CLI importa de channels 22 veces) | >= 3; cableado no leído |
| Rol | `Role` (`src/shared/types.ts:1`, heredado), `shared/ports.ts`, `core/role-ports.ts`, `session-role.ts`, filtros de herramientas | >= 4; no verificado |

No hay registro de plugins en tiempo de ejecución para canales ni herramientas: todo se cablea en código. Existe `test/gateway-plugins.test.ts` (solo el nombre).

## 7. Comportamiento ante fallos a mitad

- Reinicio: `Runtime.start` arma Routers, recupera, arranca el daemon y reanuda lo diferido (`src/runtime.ts:2345-2358`). `recoverInterrupted` (l.2382) no toca nada si el modelo es `MockModel` (l.2393-2402); si no, finaliza efectos externos inciertos (l.2404) y llama a `Engine.recoverInterrupted` (l.2425): cada ticket abierto, sin pausa ni espera de aprobación, con asignación `running` se cierra y recibe una de reemplazo (`engine.ts:99-125`). El error de un ticket se registra y se sigue.
- Una pausa humana o una aprobación pendiente no cuentan como caída (`engine.ts:105-108`).
- Fallo de modelo: el turno termina y la asignación queda `running` (`loop.ts:1041-1053`) esperando al watchdog. No encontré en lo leído un timeout por llamada de modelo en `loop.ts` (su único `setTimeout` está en `scheduleJob`, l.755); `src/harness/model.ts` no lo revisé.
- Watchdog: el daemon llama a `watchdog.tick` al arrancar y cada `intervalMs`, capturando excepciones por tick (`src/harness/daemon.ts`, `start` y `safeTick`). El `stuckMs` por defecto del runtime no lo leí; las demos usan 60 000 ms (`src/cli/demo.ts:56`). Política nudge y luego fail: heredada (`watchdog.ts:22-51,129`), no recomprobada.
- Informe sobre ticket ya terminado: no-op si viene del watchdog, `UserError` en otro caso (`assignments.ts:90-96`).
- Outbox: `MAX_ATTEMPTS = 5`, retroceso exponencial acotado (`src/channels/outbox-adapter.ts:29,135-155`); errores no reintentables fallan al primer intento (l.12-14).
- Si el arranque de un Worker lanza, se liberan el marcador activo y el hueco de mesh (`loop.ts:356-371,459-486`).
- Hueco: la cola de mesh vive en memoria (`loop.ts:374-390`); tras un reinicio los tickets vuelven solo por la recuperación de asignaciones.

## 8. Decisiones de diseño notables y su coste

1. Engine síncrono, una transacción por comando. Gana invariantes simples; cuesta reescribir el ticket completo en cada `putDoc` y anidado sin savepoints.
2. Inversión por hooks de `Host`. Gana desacoplo; cuesta que la semántica viva en lambdas dentro de `Runtime.create`.
3. Un proceso para Router, Worker y almacén; aislamiento por comando con bwrap/sandbox-exec sin alternativa si falta el binario (`src/env/seccomp.ts:160-172,190`; `sandboxEnabledFromEnv` devuelve siempre `true`, l.194). Sin sandbox en Windows (l.190 lanza `UserError`). Confirmado.
4. Puertos por rol tipados (`bindRolePorts(engine)`, `src/runtime.ts:1656`) en vez de solo prompt.
5. Documento genérico en SQLite: migraciones baratas; sin integridad relacional ni índices por campo.
6. El gate de entrega es heurístico y evolucionable: 27 ficheros / 3770 líneas de `evolution` frente a 117 líneas de `speculate` (conteo propio). Desproporción notable.
7. Configuración por entorno: 68 variables `DS_AGENT_*` distintas en `src` (`git grep -ohE "DS_AGENT_[A-Z0-9_]+" -- src | sort -u | wc -l`).

## 9. Cobertura

- Abiertos (completos o por tramo): `src/core/{host,engine,intake}.ts` completos; `src/core/assignments.ts` l.90-200; `src/runtime.ts` l.1540-1700, 2330-2470, 2808-2900, 2940-3100; `src/harness/loop.ts` l.238-300, 356-540, 745-770, 1025-1075; `src/store/sqlite.ts` 1-140; `src/harness/daemon.ts`; `src/env/seccomp.ts` 155-222; `src/harness/tools.ts` 100-128; `package.json`, `tsconfig.json`, `vitest.config.ts`, los 4 workflows, `CLAUDE.md`, fragmentos de `scripts/lint.ts` y `docs/BROWSER_ERRANDS.md`.
- Aproximado: unas 1830 de 35 281 líneas de `src` (5 %) y unos 10 de 172 ficheros (6 %). Todo el grafo de importaciones se calculó sobre los 172 ficheros, pero eso no es lectura.
- Sin ver: `src/evolution`, `src/channels` (salvo cabeceras), `src/gateway`, `src/cli`, internos de `src/memory`, `harness/{remote-node,proactive,mesh,mcp,prompts}`, `src/env` salvo seccomp, tests (solo nombres y conteos), docs (solo lista).
- Correcciones a la nota anterior: (a) `docs/BROWSER_ERRANDS.md:3` dice "PR-only ... not merged to main", pero `src/harness/tools.ts:105` y `src/runtime.ts:127` lo cablean y GitHub muestra la PR #172 "owner-gated browser errand adapter" fusionada el 2026-11-19T12:00:52Z: doc desfasado, no duda de rama. (b) "153 entradas en test/": `git ls-files test` da 167 ficheros (148 `*.test.ts`, más fixtures y helpers). (c) "unos 45 docs": `git ls-files docs` da 60 ficheros (con assets, layouts y 7 de `superpowers/`). (d) Faltaba: la recuperación se difiere con `MockModel` (`runtime.ts:2393-2402`) y el anidado de transacciones sin savepoint. Sin poder comprobar: la fusión sin aprobación (#132) y el CI "709/15". Confirmados: sandbox fail-closed sin Windows, `sandboxEnabledFromEnv` siempre `true`, MCP solo para worker, tamaños de `runtime.ts`/`loop.ts`/`cli/main.ts`.
