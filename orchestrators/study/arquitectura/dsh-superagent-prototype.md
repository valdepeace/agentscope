# Arquitectura: dsh-superagent-prototype

Lectura de solo lectura sobre `references/dsh-superagent-prototype` (1 commit, 216 ficheros versionados). Rutas relativas a la raíz del repo. Nada se ejecutó; el análisis de imports lo hice con un script propio de lectura (regex sobre `import`/`export from`, ignora imports multilínea), por lo que sus cifras son aproximadas.

## 1. Resumen y veredicto

- Plugin de DeepSeek Harness (DSH/Cordis) que convierte eventos de GitHub (y Feishu, opcional) en Tickets ejecutados por una sesión de agente por Ticket (src/index.ts:7-25, src/executor/dsh.ts:24).
- Núcleo de dominio limpio: `src/core` no importa nada fuera de `core` (comprobado por script; docs/007-architecture.md:258 lo afirma igual) y toda mutación pasa por `Engine` en transacciones SQLite síncronas (src/core/engine.ts:24-48, src/app/store.ts:140-153).
- Fiabilidad por diseño: cola de efectos durable con reintentos, un escritor por estado, recuperación al reiniciar (src/app/effects.ts:42-66, src/core/assignments.ts:83-92).
- Debilidades: capa `src/app` con un ciclo de imports de valores entre ~10 ficheros, aislamiento de workers inexistente (src/app/sessions.ts:152), sin purga de datos (docs/007-architecture.md:484), dependencia total de DSH alpha.
- Veredicto: arquitectura sólida y poco común para un prototipo en el dominio y la persistencia; frágil en seguridad de ejecución y en el acoplamiento de la capa de aplicación. Cobertura de lectura baja (~32 % de las líneas de src), ver sección 9.

## 2. Mapa de módulos y capas

Ficheros/líneas medidos con `git ls-files src | xargs wc -l` (`src/client` aparte).

| Módulo | Responsabilidad | Tamaño | Depende de (imports de valor) |
| --- | --- | --- | --- |
| src/core | Dominio: modelo, puertos, Engine y sus 5 partes (Intake, Evals, Tickets, Assignments, Replies), repos en memoria | 15 ficheros, 2088 l. | nada fuera de core (src/core/engine.ts:2-11) |
| src/policy.ts, admission.ts, limits.ts, kv.ts, deployment.ts, setup.ts, navigation.ts | Esquemas de settings, gate y revalidación de Mandate, claves KV | 8 ficheros, 544 l. (raíz) | core, infra, app (3 aristas) |
| src/app | Raíz de composición (`Runtime`), Store SQLite, Effects, Outbox, Sessions, Settings, Prompts, Passes, HTTP, assessor | 28 ficheros, 4230 l. | core, infra, sources, executor, views, tools |
| src/app/tools | Tools del router (14) y del executor (6) | 3 ficheros, 441 l. | app, core, sources |
| src/app/views | Proyecciones de lectura para panel y tools | 7 ficheros, 621 l. | app, core, executor |
| src/executor | `DshExecutor` (sesiones, slots, reintentos) y control de trabajo en background | 2 ficheros, 838 l. | core, app (store, effects, sessions) |
| src/sources | Contrato `SourceKind`, registro, GitHub y Feishu (polling), lectura de contexto | 16 ficheros, 2059 l. | core, infra, src raíz; ninguna arista a app |
| src/infra | Comandos CLI, entorno/shims, health, texto, rutas | 5 ficheros, 347 l. | raíz, sources (1) |
| src/client | Panel React | 27 ficheros, 4419 l. | solo valores de core/errors, core/reasons, sources/host, api.ts (filtros) |

Fuera de src: prompts/ (15 ficheros), docs/ (9, 1973 l.), scripts/dsh-superagent (250 l., Python según README, no leído), cordis.patch.yml (194 l.), skills/ (2).

## 3. Fronteras y acoplamiento

- Hecho verificado: `core` sin imports externos; `sources` sin aristas a `app` (script); el panel importa solo valores permitidos (docs/007-architecture.md:258 coincide con lo medido).
- Las dependencias se inyectan por interfaces `Pick<>` (`ExecutorHost` src/executor/dsh.ts:59-82, `SessionsHost` src/app/sessions.ts:24-39, `AssessorHost` src/app/assessment.ts:49-64, `OutboxHost` src/app/outbox.ts:9-15).
- Ciclo de valores en `app`: el script encontró un componente fuertemente conexo con app/prompts, app/views/records, app/resources, app/tools/tool, app/context, app/links, app/notices, executor/dsh, app/cards y app/assessment. Pares confirmados a mano: dsh.ts importa `resultRelay` de notices (src/executor/dsh.ts:18) y notices importa `triggerMessage` de dsh (src/app/notices.ts:4); tool.ts importa resources (src/app/tools/tool.ts:4) y resources importa dsh (src/app/resources.ts:4). Los ciclos solo de tipos (p. ej. sessions ↔ dsh) no cuentan.
- Interpretación: el executor, las notificaciones y los recursos que lee el modelo forman un bloque entrelazado; la separación "executor" frente a "app" es nominal.
- Ficheros con más salidas: runtime.ts (33 imports, composition root, src/app/runtime.ts:16-47), http.ts (22), resources.ts (16). Con más entradas: core/index.ts (41), store.ts (23).
- "Módulo dios": `Runtime` es raíz de composición deliberada (src/app/runtime.ts:54-58); `dsh.ts` (673 l.) concentra prepare, slots, entrega, reintentos y vigilancia de background.
- Cruce de capas notable: `SourceKind` mezcla UI (campos de formulario, textos zh/en), política, canal y entorno en un solo descriptor (src/sources/kind.ts:85-113).

## 4. Estado y persistencia

- Un SQLite (`node:sqlite`, WAL, synchronous=FULL, permisos 0600) con tablas trigger, disposition, thread, ticket, decision, outbox, trigger_thread, effect, kv (src/app/store.ts:2,64,71-87). Agregados guardados como JSON en columna `value`, con columnas generadas e índices parciales (`open`, `active`, src/app/store.ts:30-44).
- Escritor único: Engine escribe Triggers/Threads/Decisions/Tickets (src/core/engine.ts:19-23); Outbox es el único escritor de recibos (src/app/store.ts:253-254); cada Source escribe sus cursores vía `SourceContext.state` (src/sources/host.ts:45).
- Transacciones: `atomic` con SAVEPOINT anidados y callbacks `afterCommit` que se descartan en rollback (src/app/store.ts:140-162). Efectos externos se encolan en la misma transacción (src/app/effects.ts:61-66) y se ejecutan después.
- Migración: `SCHEMA_VERSION=11`, `OLDEST_SCHEMA=10`; solo se migra 10→11 en una transacción y rechaza versiones menores o mayores (src/app/migration.ts:20-25, src/app/store.ts:65-96). Cuerpo de migration.ts (616 l.) no leído salvo firmas.
- Bloqueo de directorio con `proper-lockfile` antes de abrir la BD (src/app/runtime.ts:90-94).
- Estado en memoria que se pierde: `TextCache` (src/app/runtime.ts:77,118), mapas del executor (`transient`, `settling`, `left`, `personal`, src/executor/dsh.ts:211-224).
- Settings y prompts: revisiones versionadas en `kv` (src/app/store.ts:183-199).
- Sesiones y su historial viven en DSH, fuera de este repo (src/app/sessions.ts:140-146).

## 5. Flujo de control de una tarea

1. Un Source hace poll de GitHub (cada >=60 s, src/sources/github.ts:25,144) y llama a `ctx.receive(trigger)` (github.ts:356,367,400) que va a `Passes.receive` → `Engine.receive` (src/app/runtime.ts:161, src/app/passes.ts:34-39).
2. `Intake.receive` deduplica, filtra por `SourceScope`, guarda el Trigger, lo reparte como Input a Tickets suscritos y, si ningún worker lo decide, lo encola para Eval (src/core/intake.ts:19-60).
3. Cada 2 s `Passes` ejecuta `effects.kick()` y `Engine.evaluateDue()` (src/app/passes.ts:28,63-68); este arranca Evals pendientes (src/core/engine.ts:75-85).
4. `Evals.start` aplica el gate determinista por Trigger, elige el más fuerte y pide el assessment (src/core/evals.ts:71-118; gate en src/policy.ts:191-211).
5. `MasterAssessor.request` encola el efecto `assessment`, que entrega la tarjeta a la sesión principal y arranca el reloj (src/app/assessment.ts:96-104).
6. La sesión principal (LLM) llama `orchestrator_eval_decide` → `assessmentOutcome` (src/policy.ts:40-61) → `Engine.decide` (src/app/assessment.ts:272-278, src/core/engine.ts:100). Las peticiones explícitas se enrutan con `orchestrator_route` (src/app/assessment.ts:39).
7. `Evals.accept` crea el Mandate y `Tickets.create` guarda ticket, suscripciones, primera Assignment y llama `executor.start` (src/core/evals.ts:306-316, src/core/tickets.ts:37-56).
8. `DshExecutor.start` encola `launch`; el handler reserva slot, prepara directorio (`sh -c prepare`, 10 min), asegura la sesión `worker-<id>` y envía el primer mensaje (src/executor/dsh.ts:264-285,424-470).
9. El worker trabaja con 6 tools propias y reporta con `orchestrator_report` → `Engine.report` (src/app/tools/executor.ts:75-102, src/core/assignments.ts:99-133).
10. Fin de turno: `workerIdle` espera a que no haya background (límite 30 min) y registra `waiting` (src/executor/dsh.ts:601-671).
11. Cierre: un report `ready` termina el Ticket (o espera a que se cierren los PRs que sigue, `followsDone`, src/core/tickets.ts:95-101); un mensaje humano posterior lo reanuda sin Eval (`Tickets.wake`, src/core/tickets.ts:252-270).
12. Los mensajes salientes salen por `Outbox` con recibo y efecto `reply` (src/app/outbox.ts:27-51).

## 6. Puntos de extensión

- Source/canal: implementar `SourceKind` y registrarlo con `ctx.orchestrator.sources.register` en un plugin que inyecta `orchestrator` (src/sources/github.ts:110-122; contrato src/sources/kind.ts:85-113). Ficheros a tocar según el patrón de Feishu/GitHub: un fichero de kind, uno por trigger/canal/lectura (github tiene 6), entrada nueva en tsdown.config.ts y en `exports` y `cordis.patch.yml` (tsdown.config.ts:11, package.json exports, cordis.patch.yml:4-12). Estimación: 5-8 ficheros. docs/008-cookbook.md existe (425 l.) pero no lo leí.
- Herramienta del router/worker: añadir un `tool({...})` en src/app/tools/main.ts o executor.ts; el registro es automático (src/app/sessions.ts:261). Si el worker necesita un tool de DSH, hay que añadirlo a `WORK_TOOLS` (src/app/sessions.ts:61-62). 1-3 ficheros más docs/prompts (hay tests de coherencia, docs/007-architecture.md:432-446 según nota previa, no releído).
- Rol nuevo o harness/proveedor distinto: no hay extensión; `Executor` es un puerto (src/core/ports.ts:59-69) con una única implementación `DshExecutor`, pero el modelo y los presets están atados a DSH (src/app/sessions.ts:48, cordis.patch.yml). Cambiar de harness implica reescribir executor, sessions, tools, presets. El modelo por rol es configurable (`eval.model`, `executor.model`, src/policy.ts:83-97).
- Regla de política: dato en settings, no código (src/policy.ts:62-65).

## 7. Comportamiento ante fallos a mitad

- Reinicio del host: `resume()` cancela sesiones vivas, para el trabajo en background y llama `assignmentInterrupted` (src/executor/dsh.ts:524-537). Si el último report era waiting/ready/blocked/failed y nada llegó después, conserva la sesión; si no, reinicia la Assignment con nota de recuperación (src/core/assignments.ts:21,83-92).
- Efectos pendientes sobreviven en SQLite y se reintentan al arrancar (src/app/effects.ts:136-145). Backoff exponencial; 3 fallos ponen la salud en error (effects.ts:7,141).
- Lanzamiento: error transitorio hasta 3 intentos (`retry: [5000, 300000]`, dsh.ts:30,232-233,282); fallo de `prepare` no se reintenta y marca `failed` (dsh.ts:43,282-283,410-415).
- Turno fallido por error transitorio: reintento con 15/30/60 s (dsh.ts:555-576). Error no transitorio: report `failed` (dsh.ts:505-511).
- Eval sin respuesta: se reencola 2 veces y a la 3.ª `unavailable` (src/core/evals.ts:19,60-68); el reloj cuenta desde la entrega (evals.ts:48-55); en pausa se desplaza el deadline (evals.ts:35-46).
- Crash entre commit y ejecución: el efecto sigue en la tabla, por lo que no se pierde trabajo; la idempotencia depende de ids (`prompt` por requestId, src/app/sessions.ts:162-167; `Outbox.add` ignora duplicados, store.ts:266-269).
- Cierre ordenado: cancela agentes con `keepInbox`, drena efectos, para background (10 s), cierra BD antes de soltar el lock (src/app/runtime.ts:230-252).
- Lo que no encontré: detector de worker colgado dentro de un turno activo (sin timeout de turno) en lo leído; el límite de 30 min solo aplica tras fin de turno (dsh.ts:32,629-641). Tampoco límite de coste por Ticket.

## 8. Decisiones de diseño notables y su coste

- Mandate frente a Subscription: autorizar es distinto de dar contexto (src/core/intake.ts:38-45, src/core/tickets.ts:416-420). Coste: mucha lógica de casos límite (`letGo`, `unfinished`, `attach`) en tickets.ts, 424 líneas densas.
- Agregados como JSON en una columna: simple y flexible; coste: consultas vía JSON/columnas generadas y migraciones manuales (src/app/store.ts:30-44, migration.ts).
- Engine síncrono y efectos tras commit (patrón outbox): consistencia fuerte; coste: toda la espera/red queda fuera del dominio y el estado de ejecución se reconstruye con heurísticas (`busy`, `handedOff`, dsh.ts:614-618).
- Router como LLM con tools: flexible; autorización la decide código (`decide` revalida, src/core/evals.ts:152-154). Coste: el modelo decide enrutado/attach sin tests con modelo real.
- Polling en vez de webhooks: sin infraestructura entrante; latencia >=60 s y consumo de rate limit (github.ts:25,203-209).
- Sin aislamiento: workers con `danger-full-access` (src/app/sessions.ts:152, docs/006-deployment.md:19); los shims son por nombre y reconocidos como no-frontera (src/infra/environment.ts:1-5, docs/003-execution.md:80).
- Preset de herramientas con lista blanca: el docstring dice que el worker no recibe herramientas que "inicien o envíen mensajes a otras sesiones" (src/app/sessions.ts:55-60), pero `WORK_TOOLS` incluye `send_message`, `list_agents` e `interrupt_agent` (sessions.ts:61-62). Desfase código/comentario; el efecto real depende de qué tools existan en el preset (`restrict` solo permite las existentes, sessions.ts:292), no verificado.
- Compatibilidad heredada ya en un prototipo: inputs `reopen` (dsh.ts:154), marcadores `dsh-tag/dsh-go/dsh-freelancer` (src/infra/environment.ts:16), Evals sin `primary` (src/core/tickets.ts:40).

## 9. Cobertura

- Leí completos o casi: src/index.ts, app/runtime, passes, effects, store, sessions, outbox, assessment (300 l.), tools/executor, policy, core/engine, ports, intake, evals, tickets, assignments, executor/dsh, sources/kind, registry, host, poll, github.ts (~83 %), infra/environment, cordis.patch.yml, tsdown/tsconfig/package.json. Parcial: docs/001-007 (extractos puntuales), tests/fixtures/host.ts (60 l.).
- No leí: app/migration (solo firmas), settings, prompts, http (solo grep de origen), resources, setup/deployment, tools/main, views/*, sources/feishu*, github-read/pull/channel/trigger, core/model, replies, stage, memoria, scripts/dsh-superagent, prompts/*.md, docs/008, src/client (salvo imports), tests (salvo conteos).
- Cifra aproximada: ~5000 de 11168 líneas no-client leídas (~45 %), ~32 % de todo src (15587 l.), ~35 de 111 ficheros completos o casi; tests <2 %; docs ~5 %.

## Correcciones a la nota previa (study/dsh-superagent-prototype.md)

- `orchestrator_report` no acepta `waiting`: el enum es progress/ready/blocked/failed/dropped (src/app/tools/executor.ts:78); `waiting` lo genera el Executor al fin de turno (src/executor/dsh.ts:665). La nota decía que el worker lo reporta.
- Tamaño: 216 ficheros versionados (no ~245); src no-client 11168 l. + client 4419 l.; tests 14960 l. en 59 specs (65 con fixtures), no 57; la nota subestimaba el código (~12,4k sin tests).
- Faltaba mencionar que Sources hacen polling (mín. 60 s) y no webhooks (src/sources/github.ts:25).
- Faltaba el ciclo de imports en app y la discrepancia de `WORK_TOOLS` (sección 8).
- Confirmados: 14 tools del router, 6 del executor, passes de 2 s, `danger-full-access`, reintentos Eval 2+1, backoff 15/30/60 s, límite de background de 30 min, loopback+same-origin (src/app/http.ts:76-78,231).
- No verificable desde el repo: lo que hacen `dsh-bash-sandbox`/`pwsh-sandbox` (solo se ve su config de timeouts, cordis.patch.yml:45-56).
