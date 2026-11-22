# Arquitectura de los siete: comparativa

Segunda pasada sobre siete repos (2026-11-21), solo lectura. Notas por repo en `study/arquitectura/`. La mantenibilidad va aparte en `MANTENIBILIDAD.md`. Los clones son superficiales (1 commit), así que no hay análisis de historial.

**Confianza.** La lectura línea a línea es muy baja (del ~0,2 % al ~32 % según repo, ver tabla). Los tamaños, los recuentos y los grafos de imports salen de comandos y scripts sobre todos los ficheros; las afirmaciones de comportamiento, de la parte leída. Un «no encontrado» significa «no encontrado en lo leído».

## Comparativa

| Repo | Forma | Estado y persistencia | Módulo dios | Fronteras de capas | Añadir un harness/pieza | Fallo a mitad de tarea |
|---|---|---|---|---|---|---|
| [`dsh-superagent-prototype`](study/arquitectura/dsh-superagent-prototype.md) | Plugin DSH/Cordis; un Ticket = una sesión | SQLite síncrono; cola de efectos durable | no destacado | `src/core` no importa nada de fuera (comprobado); ciclo de imports de valores en `src/app` (~10 ficheros) | n/d | recuperación al reiniciar; sin detector de colgado dentro de un turno |
| [`LofiSu/dsh-super-agent`](study/arquitectura/LofiSu-dsh-super-agent.md) | Motor síncrono + arnés ReAct en el mismo proceso | SQLite genérico `documents(kind,id,payload)`; sin savepoints (anidadas se aplanan) | `src/runtime.ts` 3.231 líneas (`Runtime.create` ~1.560) | puertos por rol; ciclos core↔channels | n/d | recuperación diferida con `MockModel`; riesgo de escrituras parciales (deducción, no ejecutado) |
| [`deer-flow`](study/arquitectura/deer-flow.md) | Lead agent + 50 clases Middleware dentro del Gateway FastAPI | SQL + checkpoints | ficheros de 3.000–4.700 líneas | **la más limpia hacia fuera** (test AST: harness no importa `app`); 23 pares con import mutuo dentro | extension-api | runs: leases y reconciliación; subagentes `task` en memoria, no se reanudan |
| [`agent-orchestrator`](study/arquitectura/agent-orchestrator.md) | Daemon Go + Electron/React + móvil; sesión = worktree + rama | SQLite (goose, 263 migraciones) | `session_manager/manager.go` 8.382 líneas; `daemon.Run` ~1.550 | puertos reales pero con fugas (adapters→core/storage; storage→service) | ~35 ficheros + reescribir un CHECK de SQLite a mano; el comentario dice «una edición» | el arranque no relanza agentes; hay reconciliación |
| [`agent-swarm`](study/arquitectura/agent-swarm.md) | API/MCP Bun dueña de SQLite + workers por HTTP | SQLite WAL, 261 migraciones | `db.ts` 14.753; `runner.ts` 8.334 | vigiladas en CI (grep + dependency-cruiser), pero la regla transitiva está en `warn`; muchos ciclos | 12 o más ficheros por proveedor | poll + claim atómico + barrido de heartbeat 90 s; atasco con heartbeat fresco solo se registra; sin tope de duración por tarea (en lo leído) |
| [`cli-agent-orchestrator`](study/arquitectura/cli-agent-orchestrator.md) | FastAPI local + CLIs en tmux + MCP | SQLite (22 tablas) + ficheros; 24 funciones `_migrate_*` a mano, sin Alembic | `api/main.py` 12.562 líneas (~123 rutas) | capas declaradas, muy cruzadas; 128 imports diferidos | n/d | `run_agent_step` cubre timeout, error y cancelación; fallo reconocido al reutilizar terminal (`agent_step.py:826-831`) |
| [`gastown`](study/arquitectura/gastown.md) | Un binario Go (`gt`) + tmux + Dolt vía `bd` | Dolt + ficheros del town | `internal/cmd` 103.691 líneas (41,6 %); `runDone` 1.313, `runSling` 945 | sin ciclos entre paquetes de primer nivel; el daemon llama a `gt` por subproceso | añadir un rol toca ~80 ficheros | el estado vive en hooks y beads; parte del control lo llevan agentes LLM |

## Tipo de arquitectura: pretendida frente a real

Qué dice ser cada repo (en sus docs o en sus nombres de paquete) frente a lo que se ve en el código leído. «Distancia» es mi lectura, no una medida; la evidencia está en las notas de `study/arquitectura/`.

| Repo | Pretende | Lo que hay de verdad | Distancia |
|---|---|---|---|
| `dsh-superagent-prototype` | Núcleo de dominio puro con anfitriones inyectados (`docs/007-architecture.md:258`: el núcleo no importa nada externo) | **Núcleo funcional con cáscara imperativa**: `src/core` sí es puro (comprobado) y las dependencias entran por interfaces `Pick<>`, con cola de efectos tipo *outbox* transaccional. La separación «executor» frente a «app» es nominal: forman un bloque entrelazado de ~10 ficheros | **Corta**: cumple en el núcleo, no en la capa de aplicación |
| `LofiSu/dsh-super-agent` | Motor de dominio síncrono rodeado por un arnés asíncrono, con puertos por rol | **Hexagonal en el núcleo** (el `Engine` no conoce el arnés, hooks de `Host` en `src/core/host.ts:6-32`) con un **módulo dios de cableado** (`runtime.ts` 3.231 líneas) y ciclos core↔channels | **Media**: buena idea, mal repartido el cableado |
| `deer-flow` | Monolito modular en tres paquetes (`harness`, `extension-api`, `app`) con regla «el harness no importa `app`» | **Monolito modular con microkernel de extensiones y cadena de 50 *middlewares* sobre LangGraph.** La frontera exterior se cumple y la hace cumplir un test AST; por dentro no hay capas: 23 pares de subpaquetes con import mutuo y ficheros de 3.000 a 4.700 líneas | **Media**: exterior limpio, interior enmarañado |
| `agent-orchestrator` | **Hexagonal** (puertos y adaptadores): «los adaptadores son hojas», «servicios → núcleo» | **Hexagonal con fugas**: adaptadores que importan `observe` o `storage`, `storage` que importa `service`, `service` atado a `httpd` (~74 imports). Más un **daemon orientado a eventos** (sondas que reportan hechos y un ciclo de vida que reacciona) | **Larga**: las reglas que documenta están rotas en puntos medibles |
| `agent-swarm` | **Hub y radios**: API única dueña de la base de datos y workers que solo hablan HTTP; módulos `be`, `commands`, `providers` separados | **Cola de trabajo con reclamo atómico** (poll + `UPDATE`) y 12 adaptadores intercambiables. La frontera «el worker no importa `be/db`» la vigila CI, pero la regla transitiva está en `warn` y `be` se acopla con casi todo | **Media**: el borde está vigilado, el centro no |
| `cli-agent-orchestrator` | **Capas** declaradas: CLI/Web/MCP → ruta → servicio → backend → proveedor → proceso (`CODEBASE.md:62-71`) | **Capas cruzadas.** Un FastAPI con `api/main.py` de 12.562 líneas, ciclos entre paquetes salvados con 128 imports diferidos y tmux referenciado directamente en 8 ficheros. Lo estable son los **registros de proveedores y backends** (patrón estrategia) | **Larga**: la arquitectura declarada es una aspiración |
| `gastown` | Una **jerarquía de roles** (Mayor, Witness, Refinery, Deacon…) coordinada sobre un libro mayor compartido | **Monolito CLI en Go (`internal/cmd` 41,6 %) más un daemon, con parte del control llevada por agentes LLM que siguen fórmulas TOML.** Hay motor de merge en Go sin llamador, y tres caminos de acceso al mismo dato (subproceso `bd`, store en proceso y SQL directo) | **Larga**: el diseño es coherente en el papel, pero el control real vive en prompts y en código sin conectar |

**Resumen del patrón.** Los que **pretenden capas o hexagonal** (AO, CAO) son los que más se alejan de lo que dicen. Los que pretenden algo más modesto (un núcleo limpio, como los dos DSH, o una frontera exterior, como deer-flow) lo cumplen donde lo dicen y fallan donde no se comprometieron. En cuatro de los siete la regla que sí se hace cumplir por máquina (un test AST, un script de CI, el compilador de Go) es justo la que aguanta.

## Lo que se repite

1. **Cinco de los siete tienen un fichero o una función gigante en el núcleo** (swarm, CAO, AO, gastown, LofiSu). Es el patrón más claro.
2. **Las capas declaradas no se cumplen del todo.** AO incumple las reglas que documenta; CAO salva ciclos con imports diferidos; swarm deja la regla transitiva en `warn`. La excepción es deer-flow en su frontera exterior, hecha cumplir por un test.
3. **SQLite domina el estado** en seis de los siete (gastown usa Dolt). Las migraciones a mano o con coste oculto aparecen en AO, swarm y CAO.
4. **Añadir una pieza cuesta mucho más de lo que la documentación sugiere.** AO ~35 ficheros, gastown ~80 para un rol, swarm 12 o más.
5. **La recuperación tras fallo es lo mejor resuelto** en deer-flow (leases), swarm (barrido) y AO (reconciliación), y lo menos en los dos DSH, donde falta un detector de colgado.

## Cobertura de lectura y correcciones

| Repo | Lectura línea a línea | Correcciones a la nota anterior |
|---|---|---|
| dsh | ~32 % de `src`; <2 % de tests | `orchestrator_report` no acepta `waiting`; 216 ficheros versionados y 59 specs; fuentes con polling ≥60 s |
| LofiSu | ~5 % de `src` | `docs/BROWSER_ERRANDS.md` desfasado; 167 ficheros de test y 60 docs |
| deer-flow | ~1–2 % | timeout de subagentes 1800 s, no 900; «máx. 3» es solo el valor por defecto (hasta 64); 50 Middleware; 23 skills; `allow_host_bash` es false por defecto |
| agent-orchestrator | ~0,2 % de líneas | 263 migraciones, no ~215; 44 harnesses; el arranque no relanza agentes; 7.950 ficheros |
| agent-swarm | ~0,7 % | 1.139 tests; 103 líneas con `leadOnly`; `docker.sock` está comentado en `docker-compose.local.yml:145` |
| cli-agent-orchestrator | ~1 % | 317 ficheros y 166.505 líneas de código (no 477 ni ~115k); 3.004 versionados |
| gastown | ~1,5 % | `HandlePolecatDone` no tiene llamador: el aviso de `gt done` va por nudge más metadatos del agent bead |

Estas correcciones **no se han propagado aún** a `STUDY.md`, `TABLAS.md` ni al dashboard.
