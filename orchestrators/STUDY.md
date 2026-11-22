# Estudio comparativo: implementaciones de "Super-agent"

Fecha: 2026-11-21. Material: nueve repositorios clonados en `references/` (clones superficiales, `--depth 1`): cinco en la primera ronda y cuatro en la ampliación (§10). Notas por repositorio en `study/`. Todo el análisis es de **solo lectura**: no se ejecutó ni se instaló nada de los repositorios.

## 0. Cómo leer este documento

- Cada afirmación procede de las notas de `study/<repo>.md`, que citan `ruta:línea`. Las notas las produjo un agente por repositorio con una plantilla común; yo las he leído enteras y las he cruzado, y las citas se han verificado por muestreo (ver abajo).
- Grado de confianza por repositorio:

| Repo | Cobertura | Confianza |
|---|---|---|
| `dsh-superagent-prototype` | Docs 000-008 y código leídos, 27 llamadas | Alta |
| `cc-super-agent` | Repo casi entero (35 ficheros) | Alta |
| `super-super-harness` | Docs y código, 26 llamadas | Alta |
| `deepseek-super-agent` (LofiSu) | Muestreo sobre 437 ficheros; el agente lo avisa | Media |
| `deer-flow` | Muestreo sobre 4770 ficheros; el agente avisa de que los docs comprimidos quedaron parciales | Media-baja |
| `agent-orchestrator` | ~3-4 % de los ficheros; sobre todo docs y cabeceras. Ampliado en §12 con ~50 ficheros de lifecycle, Windows y SQLite | **Baja** |
| `agent-swarm` | ~15 %. Ampliado en §12: `heartbeat.ts` (sweep, clasificador, CAS), `queue-stall-alarm.ts`, `docker.ts`, auth, Slack/GitHub y `scrubSecrets` | **Baja** |
| `cli-agent-orchestrator` | ~12 %. Ampliado en §12: `api/main.py` (auth, 123 rutas), broker K8s, tres `get_status` y la cabecera de los e2e | **Baja** |
| `gastown` | ~15-20 %. Ampliado en §12: ~1.900 de las 3.098 líneas de `daemon.go`, `stuck.go`, `sling.go`, proxy y sandbox | **Baja** |

Las secciones §1-§9 comparan los cinco primeros. Las tablas de los nueve están en `TABLAS.md`, y §10 y §11 recogen la ampliación y la síntesis.

- Ninguna cifra de tests, rendimiento o CI se ha reproducido. Son cifras declaradas o contadas por regex.
- **Verificación de citas (2026-11-21).** Se extrajeron las **807 citas `ruta:línea` únicas** de las nueve notas y se comprobó mecánicamente que cada una apunta a un fichero que existe y a una línea dentro de su longitud. Tras normalizar rutas abreviadas (prefijo del paquete, `R/`, nombres sin directorio) **todas resuelven**; 64 son ambiguas por nombre de fichero repetido (sobre todo `README.md` y `CLAUDE.md`) y las rutas abreviadas de `agent-swarm` se completaron. Después se contrastó una **muestra de 72 citas (8 por repositorio, elegidas al azar entre las de hasta 14 líneas de las notas originales)** con el código real: **67 respaldan lo que la nota afirma, 4 estaban desplazadas 1-3 líneas (corregidas), 1 es no concluyente y 0 contradicen la afirmación**. Límites: la muestra no cubre las citas largas ni las de los apartados §13, que redactaron los agentes de la segunda pasada; y comprobar que una cita respalda una frase no prueba que la frase entera sea cierta.

## 1. Qué es "Super-agent"

DeepSeek publicó un artículo de diseño ("Exploring Super-agent Design") y un prototipo MIT. La idea: un **Router** residente decide qué merece atención y reparte; **Workers** aislados ejecutan; solo el Router habla con el usuario; un **Watchdog** vigila. Cuatro de los cinco repositorios son variantes de esa idea. **DeerFlow no lo es**: se incluyó como contraste.

## 2. Resumen de un vistazo

*Primera ronda (cinco repos). Los nueve, columna a columna, en `TABLAS.md`.*

| | dsh-superagent-prototype | cc-super-agent | deepseek-super-agent | super-super-harness | deer-flow |
|---|---|---|---|---|---|
| Autoría | DeepSeek (no oficial) | 1 autor | 1 autor | 1 autor | ByteDance |
| Lenguaje | TypeScript | Bash + Node | TypeScript | Python (solo stdlib) | Python + Next.js |
| Licencia | MIT | MIT | MIT | MIT | MIT (verificada, `LICENSE:1`) |
| Tamaño | ~12k líneas sin tests | ~1.300 líneas | ~35k líneas en `src/` | ~27k con tests | ~4770 ficheros |
| Tests | ~794 casos, host simulado | **ninguno** | 148 ficheros, 15 saltados | 866 `def test_`, harness falso | ~1377 ficheros |
| Harness de agente | DeepSeek Harness (alpha) | Claude Code | Runtime ReAct propio | Claude Code (Hermes experimental) | LangGraph |
| Estado | SQLite | Markdown en disco | SQLite (`node:sqlite`) | SQLite WAL | SQL + checkpoints |
| Sandbox real | **No** | **No** | Sí en macOS/Linux, no en Windows | **No** | `local` flojo; `aio` monta socket Docker |
| Corre en Windows | No verificado | No (WSL2) | Sin sandbox en Windows | No (WSL2) | CI con "Windows live policy" |
| Madurez | Prototipo declarado | Experimental | v0.1.0, 1 autor | v0.3.0, 6 commits, macOS | Sin release oficial |

## 3. Comparativa por ejes

### 3.1 Modelo de dominio

- **dsh** es el más formal: `Source → Trigger → Thread → Eval → Ticket(Mandate) → Assignment → Executor`. Separa **Mandate** (permiso, inmutable) de **Subscription** (contexto sin autoridad). Seguir una conversación no concede poder.
- **deepseek-super-agent (LofiSu)** copia ese vocabulario casi tal cual (Trigger, Thread, Eval, Mandate, Ticket, Assignment) y añade `TriggerTrust` separado del `meta`.
- **super-super-harness** usa un vocabulario propio: Mensaje, Item (asunto del usuario), Worker/Task, Assignment, Evento, Fuente, Regla, Observación. Los Workers son **permanentes y con identidad** (`code_name`).
- **cc-super-agent** no tiene modelo formal: Task, Project e INDEX en Markdown. No hay eventos ni triggers externos.
- **DeerFlow**: Thread, Run, Lead agent, Subagente, Skill, Project, Channel. Es un modelo de conversación, no de eventos proactivos.

### 3.2 Router (o equivalente)

| | Qué es | Filtrado de entrada |
|---|---|---|
| dsh | Sesión LLM (`orchestrator`) sin write/edit | Gate determinista + assessment LLM obligado a juzgar cada regla y citar evidencia |
| cc-super-agent | Sesión interactiva de Claude Code; todo en prompt (`AGENTS.md`) | Ninguno en código: política de comunicación en el prompt |
| LofiSu | Router con puertos exclusivos (`reply`, `decide`) | `fastPath()` determinista + slow path por reglas, o por modelo (opcional, con timeout y fallback) |
| super-super-harness | Sesión de Claude Code que solo actúa vía 30 herramientas MCP | Reglas por fuente y permiso (drop/record/direct/wake/observe); texto externo escapado como dato |
| DeerFlow | Un único *lead agent* con ~40 middlewares; no hay Router separado | Middlewares, no un filtro de admisión |

Hallazgo clave: **solo LofiSu usa literalmente los términos "fast path / slow path"**. El prototipo oficial no los usa; sus equivalentes son el gate, el fan-out por Subscription y el assessment. El artículo y el repo oficial no coinciden en vocabulario.

### 3.3 Worker y aislamiento

| | Aislamiento | Cómo reporta |
|---|---|---|
| dsh | Una sesión y un directorio por Ticket (típicamente `git worktree`). **Sin sandbox**: `danger-full-access` | `orchestrator_report` con estados `progress/waiting/ready/blocked/failed/dropped` |
| cc-super-agent | `claude -p` en el directorio del proyecto con `--session-id` fijo. Sin sandbox | Último mensaje con `Status:`; formato impuesto solo por prompt |
| LofiSu | Mismo proceso Node; directorio por Worker; `bash` envuelto en `sandbox-exec`/`bwrap` sin fallback | `report_to_router`; el **Gate** (no el modelo) decide si el Router habla |
| super-super-harness | Worktree y rama por Worker, proceso propio. **Sin sandbox**: `--dangerously-skip-permissions` | MCP con token propio; `report_done` verificado por `tool_use_id` |
| DeerFlow | Subagentes comparten el sandbox del padre; máximo 3 concurrentes | Contrato de informe + `acceptance_checks.py`: aceptación decidida **por código** |

### 3.4 Estado y memoria

- Los cuatro con base de datos usan **SQLite**. cc-super-agent usa **Markdown editado por el LLM**, sin locking: el Watchdog y el Router pueden pisarse `INDEX.md`.
- **Efectos externos**: dsh usa cola durable dentro de la transacción (patrón outbox). super-super-harness da *at-most-once* por turno. LofiSu marca los efectos inciertos como `unknown_outcome` y exige aprobación humana para reintentar.
- **Memoria a largo plazo**: LofiSu tiene memoria caliente/fría, pero su embedding es un hash determinista, no semántico. DeerFlow tiene DeerMem (un `.md` por hecho) con adaptadores opcionales. Los demás se limitan a notas y prompts versionados.

### 3.5 Watchdog y fallos

| | ¿Hay Watchdog? | Detalle |
|---|---|---|
| dsh | **No existe como componente** | Timeouts de Eval, reintentos con backoff, recuperación al reiniciar. Sin detector de turno colgado |
| cc-super-agent | Sí: cron cada 15 min + `claude -p` | Detecta con juicio del LLM (Died, Silent, Looping...). **Solo observa**: marca ⚠, nunca reinicia |
| LofiSu | Sí, en código (200 líneas) | Nudge y luego fail. El estado de "ya avisado" vive en memoria: se pierde al reiniciar |
| super-super-harness | Distribuido en código | `stall_minutes`=30, failover del Router con sondas, recuperación de huérfanos por PID+hora |
| DeerFlow | No único | Detección de bucles, leases con heartbeat, timeouts. Reconciliación tras reinicio **prometida, no implementada** |

El Watchdog que el artículo presenta como pieza central **falta en el repo oficial**. Lo implementan, de forma muy distinta, cc-super-agent (LLM por cron, solo informa) y LofiSu (código, con acción).

### 3.6 Seguridad

- **Ninguno de los cuatro del linaje DeepSeek aísla de verdad al Worker en Windows.** dsh y super-super-harness no tienen sandbox y lo admiten. cc-super-agent depende de los permisos de Claude Code. LofiSu tiene sandbox solo en macOS/Linux.
- **Prompt injection**: en todos la defensa es textual ("esto es dato, no instrucción") más diseño. Ninguno tiene detector específico. El propio artículo de DeepSeek lo señala como el riesgo más concreto.
- **Barrera de aprobación por prompt**: en cc-super-agent y super-super-harness, el campo `permission` del item es solo una instrucción y no se aplica en código. En LofiSu la barrera depende de que el modelo llame a `request_approval`.
- **Superficie web**: cc-super-agent acepta el token en la URL (`?t=`) y deja aprobar permisos del Router remotamente enviando teclas. super-super-harness usa ticket de un solo uso y cookie firmada, que es bastante más cuidadoso.
- **Credenciales**: dsh es el más trabajado (gitconfig propio, shims que bloquean `gh` de otra cuenta, `GH_TOKEN` eliminado). LofiSu tiene Vault y redacción de secretos.

## 4. Parecidos

1. **Dos capas**: un coordinador que no trabaja y ejecutores que sí. Se repite en los cuatro del linaje.
2. **Solo el Router habla con el usuario.** Los Workers reportan al Router.
3. **Un Worker por unidad de trabajo**, con sesión y directorio propios.
4. **Autorización fuera del modelo en la intención** (gate, reglas, allowlist), aunque en la práctica varias barreras acaban siendo prompt.
5. **Estado persistente** para sobrevivir a reinicios, sobre todo SQLite.
6. **Chain of Recall** (referencias que se siguen un salto cada vez): aparece en cc-super-agent y en LofiSu, y es idea del artículo.
7. **Todos se declaran experimentales o prototipos** salvo DeerFlow, que tampoco tiene release oficial.

## 5. Diferencias que importan

1. **Dónde vive la lógica**: cc-super-agent es ~1.300 líneas y casi todo es prompt. LofiSu y super-super-harness meten la coordinación en código. Esa diferencia es la que más afecta a si el comportamiento es **verificable y determinista**.
2. **Quién es el harness**: dsh depende de DeepSeek Harness (alpha). Los otros usan Claude Code (cc, super-super), un runtime propio (LofiSu) o LangGraph (DeerFlow). super-super-harness es el único que desacopla harness, proveedor y modelo (Provider/Profile/Harness).
3. **Proactividad real**: dsh, LofiSu y super-super-harness reciben eventos (GitHub, Feishu...). cc-super-agent solo reacciona a mensajes del usuario.
4. **Worker efímero o permanente**: dsh crea uno por Ticket; super-super-harness los mantiene con identidad, notas y dormancia.
5. **Aceptación de resultados**: DeerFlow la decide en código sobre evidencia de herramientas. LofiSu usa un gate heurístico por regex. Los demás se fían del autoinforme del Worker.
6. **Alcance**: dsh es un solo repo GitHub. DeerFlow es una plataforma multiusuario. Los demás son personales, de un usuario y una máquina.

## 6. Huecos que ninguno cubre

Esto responde a la pregunta de partida (no copiar y no tropezar con lo ya hecho):

1. **Aislamiento real y portable del Worker.** Nadie lo hace de forma que funcione en Windows. Es el hueco más claro.
2. **Prompt injection con defensa técnica** (no solo textual) hacia un Executor con permisos plenos.
3. **Aceptación verificada por código** en el linaje Router/Worker. Existe en DeerFlow, pero no en ninguno de los cuatro.
4. **Control de coste por tarea**: ninguno limita tokens o dinero por Ticket; solo el número de Workers.
5. **Evidencia de que la arquitectura dual vale la pena.** El benchmark de DeepSeek es sintético (código simulado, 5 corridas). El `BAKEOFF` de LofiSu es de un solo ticket y no concluye nada. Nadie lo ha medido con código real.
6. **Windows nativo.** Tres de los cuatro necesitan WSL2 o un porte.

## 7. Cosas dudosas o con trampa

- **Autoevaluación de LofiSu**: `COMPETITIVE_GAP` y `BAKEOFF` son el autor puntuándose. Lo comprobable salió mayormente verificado; las comparaciones "ahead" frente a otros productos no son verificables.
- **Documentación desfasada en LofiSu**: `BROWSER_ERRANDS.md` dice "PR-only, no en main" mientras el código existe en `src/`.
- **super-super-harness**: los docs se contradicen sobre subir imágenes en la web; el README lista como "Next" cosas que ya están en código; los presets de DeepSeek/GLM/Kimi "no están probados con claves reales".
- **dsh**: un solo commit, dependencias alpha, y partes del artículo que "no están implementadas" sin decir cuáles.
- **Parsers frágiles**: cc-super-agent y super-super-harness dependen de detalles internos de Claude Code (formato `.jsonl`, flags, textos de prompts de permiso). Una actualización de Claude Code puede romperlos.
- **Contradicción menor**: el agente de DeerFlow indica que su CI incluye una "Windows live policy"; no se ha comprobado qué cubre exactamente.

## 8. Implicaciones para una prueba en este equipo (Windows 11)

*Escrito con los cinco primeros. Con los nueve, ver §11.5: `agent-orchestrator` cambia el panorama.*

- Ejecutar **tal cual** cualquiera de los cuatro del linaje en Windows nativo no es viable sin WSL2. cc-super-agent y super-super-harness dependen de `flock`, `tmux`, `fcntl`, `ps`/`lsof` y `/usr/bin/env`.
- Si se quiere **probar** y no construir: el camino más barato es WSL2 con cc-super-agent (35 ficheros, se lee entero en una tarde) o super-super-harness (solo stdlib, el más completo en coordinación).
- Si se quiere **construir algo propio** sin pisar lo existente, los huecos de la sección 6 marcan por dónde ir.

## 9. Pendiente de decidir

*Sigue abierto. Los huecos actualizados con los nueve están en §11.4.*

Qué se busca con la prueba. Las opciones razonables, sin orden de preferencia todavía:

1. **Entender**: ejecutar uno bajo WSL2 y observarlo.
2. **Construir un Router/Worker mínimo propio** sobre Claude Code, con verificación determinista de resultados (idea tomada de DeerFlow) y sin depender de `tmux`.
3. **Atacar un hueco concreto** (aislamiento portable, o medir si la arquitectura dual compensa con tareas reales).

## 10. Ampliación: cuatro orquestadores más

Tras la primera ronda se amplió la búsqueda a orquestadores de agentes de código que no siguen el diseño de DeepSeek. Se clonaron y estudiaron cuatro: `agent-orchestrator`, `agent-swarm`, `cli-agent-orchestrator` (CAO) y `gastown`. Notas en `study/`. Fuentes y estado en `REFS.md`.

**Aviso de confianza.** La cobertura de lectura es baja en los cuatro, y en los repos grandes los agentes leyeron sobre todo documentación y cabeceras de ficheros:

| Repo | Cobertura | Qué quedó sin leer |
|---|---|---|
| `agent-orchestrator` | ~3-4 % de los ficheros | `sendOnce`, frontend, adaptadores, esquema SQLite |
| `agent-swarm` | ~15 % | `heartbeat.ts` y `runner.ts` (el watchdog se describe desde el runbook, no desde el código), `src/tools`, memoria, RBAC |
| `cli-agent-orchestrator` | ~12 % | `api/main.py`, otros proveedores, cuerpo de los 907 tests |
| `gastown` | ~15-20 % | cuerpo de `daemon.go` y `sling.go`, refinery, formulas |

Lo que sigue son hallazgos con ese límite. Las afirmaciones sobre watchdogs, sandbox y rendimiento de cualquiera de los cuatro no están contrastadas con ejecución.

### 10.1 Resumen

| | agent-orchestrator | agent-swarm | cli-agent-orchestrator | gastown |
|---|---|---|---|---|
| Lenguaje real | **Go** (daemon) + Electron/React + Expo | Bun / TypeScript | Python + Rust (TUI) | Go |
| Licencia (verificada) | Apache-2.0 | MIT | Apache-2.0 | MIT |
| Quién coordina | Agente orquestador + **código determinista** que enruta eventos | Lead (LLM) + **enrutado determinista previo** | Supervisor (LLM) con herramientas MCP; sin router en código | Mayor (LLM) + `gt sling` y daemon deterministas |
| Worker | CLI interactivo (PTY) en git worktree | CLI en contenedor Docker | CLI real en ventana tmux | CLI en tmux + worktree |
| Estado | SQLite, estado **derivado en lectura** | SQLite (API única dueña) | SQLite + ficheros | **Dolt** (SQL versionado) |
| Aislamiento real | Solo worktree | Docker estándar, hardening opcional | Ninguno local; **pods desechables en K8s** (ejemplo) | Solo worktree; sandbox no implementado en el repo (§12) |
| Watchdog | Solo de vida (Reaper, 5 s) | Clasificador de atasco A/B/C | Regex sobre la pantalla del TUI | Daemon (código) y Boot, Deacon, Witness (agentes LLM); la detección de atasco del Deacon no se invoca sola (§12) |
| Windows | **Sí, nativo (ConPTY)** | Docker Desktop, sin probar | No (WSL2) | Parcial, con WSL |

### 10.2 Lo que cambia respecto a la primera ronda

1. **Aparece un patrón nuevo: "coordinador LLM más enrutado determinista en código".** En los cuatro, el coordinador es un agente LLM completo, no un clasificador barato. Pero el reparto de eventos lo hace código: en `agent-orchestrator` las reacciones a CI, review y conflictos (`lifecycle/reactions.go`); en `agent-swarm` la elegibilidad por rol, capacidades y harness; en `gastown` el despacho de `gt sling` y el daemon. El LLM decide qué trabajo existe, y el código decide a quién llega y cuándo. Es lo contrario de `cc-super-agent`, donde todo vive en el prompt.
2. **Windows deja de ser un bloqueo.** `agent-orchestrator` trae soporte nativo con ConPTY y sin tmux. Es el primero de los nueve.
3. **Aislamiento: hay algo, pero nada es fuerte por defecto.**
   - `agent-swarm` mete cada worker en Docker. El endurecimiento (`CapDrop ALL`, `no-new-privileges`) es opt-in y viene desactivado. No hay aislamiento de red. Las credenciales de proveedor y de GitHub van en el entorno. Claude corre con `bypassPermissions` dentro del contenedor.
   - **El patrón mejor pensado es el de CAO en Kubernetes**: el pod que ejecuta agentes no puede crear pods. Pide al *broker* un worker acotado; el broker (sin `pods/exec` ni `secrets`) construye todo lo demás. Workers desechables con lease y reaper, y `NetworkPolicy`. Es solo un ejemplo con partes marcadas "UNVERIFIED", pero es la idea más aprovechable de la ampliación.
   - `gastown` tiene un proxy mTLS con allowlist de subcomandos, que no se usa por defecto, y un sandbox que **no está implementado** en el repo (§12).
4. **Supervisión más rica, y con sus trampas.**
   - `agent-swarm`: clasificador de atasco con tres casos, remediación con CAS sobre `lastUpdatedAt` (solo en las clases A y B; la C solo registra, §12), y *resume pinning* (la tarea reanudada se fija al agente original y solo escala al lead si nadie la reclama).
   - `gastown`: cadena de vigilancia de varios niveles ("quién vigila al vigilante"), con E-stop por señal. Parte de la cadena son agentes LLM; el único detector automático y determinista es el del daemon sobre el Deacon (§12).
   - `agent-orchestrator`: solo vigila vida, pero con una regla que merece copiarse: **una sonda fallida no prueba que la sesión esté muerta.**
   - CAO deduce el estado leyendo la pantalla del TUI con regex, y su propio README admite el fallo: un "success" a los 3,1 s en una tarea de 36 s.
5. **La aceptación verificada por código sigue siendo solo de DeerFlow.** Ninguno de los cuatro nuevos comprueba el resultado del worker más allá de su propio informe o del estado del PR.

### 10.3 Ideas aprovechables, por origen

| Idea | De dónde | Para qué |
|---|---|---|
| Broker de privilegios mínimos y workers desechables con lease | CAO (K8s) | Aislamiento sin dar al supervisor poder sobre la infraestructura |
| Techo de delegación `delegateTools`: un agente no concede más de lo que tiene | CAO | Anti-escalada por delegación |
| Callback garantizado: si el worker queda mudo, el servidor reenvía su última respuesta | CAO | Que un worker callado no cuelgue al supervisor |
| Nudges independientes con deduplicación persistida y re-armado cuidadoso | agent-orchestrator | Enrutar CI, review y conflictos sin duplicar ni perder avisos |
| "Una sonda fallida no prueba que esté muerto" | agent-orchestrator | Evitar matar sesiones vivas por un fallo de lectura |
| API única dueña de la base de datos, frontera impuesta por CI | agent-swarm | Que ningún worker toque el estado directamente |
| Resume pinning y escalada al lead solo si falla | agent-swarm | Recuperación barata antes de gastar un LLM |
| Identidad persistente, sesión efímera | gastown | Separar el historial de un worker de su proceso |
| Estado derivado en lectura, nunca almacenado | agent-orchestrator | Menos estados que corromper |

### 10.4 Huecos actualizados

| Hueco de la sección 6 | Situación tras la ampliación |
|---|---|
| 1. Aislamiento real y portable del Worker | **Sigue abierto.** Hay Docker (`agent-swarm`) y K8s (CAO), pero endurecimiento opcional, red abierta y credenciales compartidas. Nada portable a Windows nativo |
| 2. Defensa técnica contra prompt injection | **Sigue abierto.** `agent-orchestrator` solo sanea caracteres de control hacia el PTY; CAO usa texto; en `agent-swarm` y `gastown` no se encontró defensa |
| 3. Aceptación verificada por código en Router/Worker | **Sigue abierto** fuera de DeerFlow |
| 4. Control de coste por tarea | **Parcial.** `agent-swarm` tiene `modelTier` y presupuestos diarios en USD al reclamar (§12), y `gastown` un tope de polecats (`scheduler.max_polecats`, por defecto sin tope). Ninguno pone tope por tarea ni por tokens |
| 5. Evidencia de que la arquitectura dual compensa | **Sigue abierto.** Ninguno de los nuevos aporta mediciones comparables |
| 6. Windows nativo | **Parcialmente cubierto** por `agent-orchestrator` (ConPTY) |

### 10.5 Correcciones a datos anteriores

- `agent-orchestrator`: la API de GitHub y mi ficha inicial lo daban como TypeScript. Es una reescritura en **Go** con frontend TypeScript. `AGENTS.md:3` habla de "la antigua implementación TypeScript". Los blogs que lo describen como un proyecto TypeScript están desfasados.
- El repo `ComposioHQ/agent-orchestrator` redirige a `OrchestratorInc/agent-orchestrator`. El módulo Go sigue llamándose `aoagents/agent-orchestrator`, así que el rebranding está a medias.
- `agent-swarm`: la afirmación de que fue "#1 en GitHub Trending" no se verificó.

## 11. Síntesis de los nueve repositorios

Este apartado cruza los nueve repositorios. Donde contradice a los §3-§8, escritos con cinco, **manda éste**. Las tablas completas, con una columna por repositorio, están en `TABLAS.md`. Confianza: la de §0 y el aviso de §10 (los cuatro de la ampliación se leyeron entre el 3 y el 20 %). Nada se ha ejecutado.

### 11.1 Dónde vive la lógica: un espectro

La diferencia que más separa a los nueve no es el lenguaje ni el harness, sino **quién decide qué: el modelo o el código**.

| Posición | Repos | Lo que decide el LLM | Lo que decide el código |
|---|---|---|---|
| Todo en prompt | `cc-super-agent` | Todo, incluida la política de comunicación | Casi nada; ~1.300 líneas |
| Coordinador LLM con herramientas, sin router en código | `cli-agent-orchestrator` | A quién delegar y cuándo (supervisor con herramientas MCP) | Autorización de delegación (`delegateTools`, allowlists), entrega del inbox |
| Coordinador LLM completo más enrutado determinista | `agent-orchestrator`, `agent-swarm`, `gastown` | Qué trabajo existe y cómo se descompone | A quién llega y cuándo: reacciones a CI/review (AO), elegibilidad por rol y capacidades (swarm), `gt sling` y scheduler (gastown) |
| Gate o router en código, LLM acotado | `dsh-superagent-prototype`, `LofiSu`, `super-super-harness` | Evaluar lo que el gate deja pasar; ejecutar | Admisión, permisos, canales de comunicación por rol |
| Lead único con middlewares | `deer-flow` | Casi todo el razonamiento | ~40 middlewares y la aceptación de resultados |

Lo que se deduce, con el límite de que es lectura y no medición: cuanta más lógica hay en código, más **verificable** es el comportamiento. No hay ninguna evidencia en los nueve de que más código dé **mejores resultados** (ver hueco 5 en §11.4).

### 11.2 Tres escaleras comparadas

**Aislamiento del Worker** (de menos a más; ninguno es fuerte por defecto):

| Nivel | Repos |
|---|---|
| Ninguno (permisos totales) | `dsh`, `cc-super-agent`, `super-super-harness` |
| Solo git worktree | `agent-orchestrator`, `gastown` (este último con un proxy mTLS para `gt`/`bd` que no se usa por defecto) |
| Contenedor Docker, endurecimiento opt-in, red abierta | `agent-swarm` |
| Sandbox del sistema operativo, solo macOS y Linux | `LofiSu` |
| Pods desechables con broker de privilegios mínimos (ejemplo, con partes "UNVERIFIED") | `cli-agent-orchestrator` |
| Sandbox compartido con el padre | `deer-flow` |

**Detección de un Worker colgado**: ninguna (`dsh`, `agent-orchestrator`: solo vigila que el proceso viva); juicio de un LLM por cron que solo avisa (`cc-super-agent`); regex sobre la pantalla del TUI (`cli-agent-orchestrator`); código con acción (`LofiSu`, `super-super-harness`, `agent-swarm`, `deer-flow` por bucles y heartbeats; en `gastown` solo el daemon sobre el Deacon, el resto depende de agentes LLM, §12).

**Aceptación del resultado**: solo `deer-flow` la decide por código sobre evidencia de herramientas. `agent-orchestrator` la deriva de hechos externos (estado del PR, CI, review), lo cual es evidencia, aunque no del trabajo en sí. Los demás se fían del autoinforme del Worker, y `cli-agent-orchestrator` documenta su propia "falsa victoria" (un éxito a los 3,1 s en una tarea de 36 s).

### 11.3 Hallazgos transversales

1. **La popularidad no mide el rigor.** El prototipo oficial tiene 24k estrellas con un commit; `gastown` tiene 18,9k y se declara experimental; `agent-orchestrator` 15k. Los dos de la ampliación con más señales de ingeniería en las notas (CI con muchos gates, e2e, docs que se contrastan con el código) tienen 1,1k y 1,6k. Son señales declaradas o contadas, no calidad medida.
2. **El patrón dominante fuera del linaje DeepSeek es "coordinador LLM completo más enrutado determinista".** En los cuatro de la ampliación el coordinador no es un clasificador barato, sino un agente completo. El código se ocupa del reparto, no de la decisión.
3. **El Watchdog que el artículo presenta como pieza central es lo menos homogéneo.** Va desde inexistente hasta una cadena de niveles con E-stop (`gastown`), aunque buena parte de esos niveles son agentes LLM y no código (§12). Hay una regla que merece copiarse tal cual: **una sonda fallida no prueba que la sesión esté muerta** (`agent-orchestrator`).
4. **La frontera de confianza suele ser la red o el host, no el programa.** `cli-agent-orchestrator` lo dice sin rodeos ("superficie de ejecución de comandos sin autenticar"); `agent-orchestrator` escucha en loopback sin auth por diseño; en `agent-swarm` una petición con la clave compartida y sin identidad se trata como operador. La mayoría ejecuta a los agentes con permisos amplios (varios con `--dangerously-skip-permissions` o equivalente).
5. **Varios dependen de la superficie interactiva de un CLI.** Al menos cuatro (`cc-super-agent`, `super-super-harness`, `cli-agent-orchestrator`, `agent-orchestrator`) leen o teclean sobre un terminal o parsean ficheros internos de otra herramienta. Una actualización del CLI puede romperlos. `agent-orchestrator` es el que más lo mitiga (guarda de escritura que relee el estado y se niega si hay un diálogo de permisos).
6. **La honestidad de la documentación es una señal por sí misma.** `cli-agent-orchestrator` marca partes "UNVERIFIED" y pide verificar afirmaciones contra el código; `agent-orchestrator` separa lo implementado de lo "en vuelo". En el otro extremo, la autoevaluación de `LofiSu` y las afirmaciones de escala de `gastown` ("20-30 agentes") no son contrastables con lo leído.

**Ideas aprovechables de la primera ronda** (las de la ampliación están en §10.3):

| Idea | De dónde | Para qué |
|---|---|---|
| Mandate (permiso, inmutable) separado de Subscription (contexto sin autoridad) | `dsh` | Que seguir una conversación no conceda poder |
| Cola de efectos dentro de la transacción (outbox) | `dsh` | No perder ni duplicar efectos externos |
| *At-most-once* por turno y failover del Router con sondas | `super-super-harness` | Recuperarse sin repetir acciones |
| Aceptación decidida por código sobre evidencia de herramientas | `deer-flow` | Dejar de fiarse del autoinforme del Worker |
| Efectos inciertos marcados `unknown_outcome`, con aprobación humana para reintentar | `LofiSu` | No reintentar a ciegas |

### 11.4 Huecos consolidados

| # | Hueco | Estado con los nueve | Quién se acerca |
|---|---|---|---|
| 1 | Aislamiento real y portable del Worker | **Abierto.** Hay Docker, pods y sandbox de SO, pero opt-in, con red abierta o sin Windows nativo | `cli-agent-orchestrator` (broker K8s), `agent-swarm`, `LofiSu` |
| 2 | Defensa técnica contra prompt injection | **Abierto.** `agent-orchestrator` solo sanea caracteres de control; el resto es texto o no se encontró | Nadie |
| 3 | Aceptación verificada por código en Router/Worker | **Abierto** fuera de `deer-flow` | `agent-orchestrator` (deriva de hechos del PR) |
| 4 | Control de coste por tarea | **Parcial.** `agent-swarm` aplica presupuestos diarios en USD (por agente, global y usuario) al reclamar la tarea, y `gastown` tiene `max_polecats` apagado por defecto. Nadie pone tope por tarea ni por tokens, ni corta a mitad de ejecución | `agent-swarm`, `gastown` |
| 5 | Evidencia de que la arquitectura dual compensa | **Abierto.** Los benchmarks existentes son sintéticos o de un solo ticket; los cuatro nuevos no aportan mediciones comparables | Nadie |
| 6 | Windows nativo | **Parcial.** `agent-orchestrator` (ConPTY) y, con funciones mínimas, `gastown` | `agent-orchestrator` |
| 7 | Seguridad por defecto del plano de control (autenticación, identidad del llamante) | **Nuevo.** Loopback sin auth, auth opt-in, o clave compartida que equivale a operador | `deer-flow` (JWT + CSRF), `super-super-harness` (ticket de un uso) |

### 11.5 Implicaciones para una prueba en este equipo (Windows 11)

Esto actualiza el §8:

- **Los cuatro del linaje DeepSeek siguen sin ser viables en Windows nativo** (necesitan WSL2 o un porte). Tampoco `cli-agent-orchestrator` (lanza `ImportError` en `win32`) ni `agent-swarm` (pide Linux containers; Docker Desktop sin probar).
- **`agent-orchestrator` es el único que dice correr nativo**: instalador `win32-x64`, ConPTY, sin tmux. Matices: es una app de escritorio monousuario, no un framework; su `STATUS.md` da por verificado el empaquetado de Windows x64, pero parte del tooling de desarrollo es macOS-céntrico y la lectura nuestra cubre solo el 3-4 %. **No se ha instalado ni ejecutado.**
- **Para entender**: lo más barato sigue siendo WSL2 con `cc-super-agent` (35 ficheros) o `super-super-harness` (solo stdlib), o probar `agent-orchestrator` en nativo si se acepta instalar un producto de escritorio.
- **Para construir algo propio**: los huecos 1, 3 y 7 de §11.4 son los que más pesan, y las ideas de §10.3 y §11.3 dan material para cada uno.

### 11.6 Lo que sigue sin saberse

- La cobertura de lectura de los cuatro repos de la ampliación sigue siendo baja pese a la profundización de §12. Sus watchdogs, sandboxes y rendimientos están por contrastar con ejecución.
- El recuento de contribuyentes de esos cuatro: la API devolvió 404 el 2026-11-21.
- Las citas `ruta:línea` se han verificado por muestreo (ver §0), no una a una. Quedan sin muestrear las citas largas y las del §13 de cada nota.
- Qué cubre exactamente la "Windows live policy" del CI de `deer-flow`.
- Si la arquitectura dual rinde más que un agente único con buenas herramientas: no se ha medido.

## 12. Profundización de la ampliación (opción B)

Segunda pasada de **solo lectura** sobre `agent-orchestrator`, `cli-agent-orchestrator`, `gastown` y `agent-swarm`, dirigida a lo que las notas habían dejado sin leer y a las afirmaciones más flojas. El detalle con `ruta:línea` está en el §13 de cada nota (`study/agent-orchestrator.md`, `study/cli-agent-orchestrator.md`, `study/gastown.md`). Cada pasada leyó entre ~50 y ~60 ficheros; la cobertura global sigue siendo baja (el §13 de cada nota lista lo que continúa sin leer). **Donde este apartado contradice al §10 o al §11, manda éste.** En `agent-swarm` se comprobó, además, **quién invoca** cada mecanismo fuera de tests.

### 12.1 Qué cambia: afirmaciones corregidas

| Repo | Antes decíamos | Lo que el código dice | Efecto |
|---|---|---|---|
| `gastown` | El Deacon detecta atascos con un ping y, tras 3 fallos, se mata (`stuck.go`) | `stuck.go` solo guarda estado. La lógica vive en `gt deacon health-check` y `force-kill`, y **nadie las invoca automáticamente**; `LoadStuckConfig` no se llama fuera de tests | La detección de atascos de los agentes **depende del prompt del Deacon**. Lo único automático y determinista es el daemon sobre el Deacon (nudge a 5-20 min, kill a >20 min) |
| `gastown` | Sandbox (exitbox) "en propuesta, con implementación parcial" | **No hay implementación**: solo un `exec_wrapper` genérico y un test de `sandbox-exec` de macOS | Sin aislamiento más allá del worktree |
| `gastown` | Proxy mTLS como pieza de aislamiento | Implementado y sólido, pero **nada lo configura al lanzar polecats** y escucha en `0.0.0.0:9876` por defecto | Opcional, no es una defensa por defecto |
| `gastown` | Refinery con merge queue "en curso" | El batch-then-bisect existe y está probado, pero **no está conectado**: fusiona un agente LLM | La cola de merge "inteligente" es hoy prompt |
| `gastown` | Un tope de polecats protege de los límites de la API | `scheduler.max_polecats` por defecto es **-1: sin tope** | El control de coste sigue sin existir por defecto |
| `cli-agent-orchestrator` | Sin política de reintento en código | Reenvío del prompt hasta 3 veces tras 8 s en `IDLE`; creación del terminal con 3 intentos | Hay reintento, sin backoff y con riesgo de duplicar |
| `cli-agent-orchestrator` | e2e "contra proveedores reales" en CI | Los e2e reales están **excluidos de CI** y solo corren a mano con `-m e2e`; el CI usa e2e con `mock_cli` | La señal de madurez de esos tests es más débil |
| `cli-agent-orchestrator` | El pod que ejecuta agentes no puede crear pods | Cierto para supervisor y worker; el broker puede crear Deployments, que crean pods | Matiz de privilegios; el pod del worker solo fija `fs_group` (sin `runAsNonRoot` ni capabilities caídas) y el **lease vive en memoria** |
| `agent-orchestrator` | `no_signal` avisa al humano | Es solo una **etiqueta** a los 90 s sin ningún hook; no avisa ni actúa. Pasado el primer hook, un worker colgado vuelve a verse como `idle` o `working` | **No hay detector de atasco**, confirmado |
| `agent-orchestrator` | CI y conflicto: `maxAttempts: 0` | Significa sin tope fijo. La firma de CI incluye `CommitHash` y `LogTail`: **un nudge por cada push que siga fallando, sin techo** | Posible bucle caro con un worker que empuja y falla |
| `agent-orchestrator` | Sin defensa ante inyección | El log de CI va **entre fence** markdown; los comentarios de review van **en crudo** y con la orden "Address each comment" | Matiz: delimitado en CI, no en review |
| `agent-orchestrator` | ~40 adaptadores; outbox durable | 44 harnesses seleccionables (19 con Chat, ~25 solo TUI); el outbox vigente solo cubre traspasos TUI↔Chat y **la entrega al terminal es best-effort (sin ack)** | Cifras y alcance ajustados |
| `agent-swarm` | El clasificador A/B/C remedia los atascos | A y B remedian (`supersede-resume` o `fail`); **la clase C (sesión viva, ≥30 min) solo registra** (`heartbeat.ts:617-620`). Salvo un hook de extensión, un agente vivo pero atascado no se rescata solo | El hueco "vivo pero atascado" persiste también aquí |
| `agent-swarm` | Control de coste solo con `modelTier` | Presupuestos diarios en USD por agente, global y usuario, aplicados **al reclamar** la tarea (`budget-admission.ts:64-128`) | Hueco 4 más cerca de cubierto; sigue sin tope por tarea, por tokens ni corte a mitad de ejecución |
| `agent-swarm` | RBAC activado por defecto | Solo gatea a principales de usuario (`core.ts:609-630`); la clave compartida es `operator` y `X-Agent-ID` lo declara el cliente | Confirma el aviso del propio runbook |
| `agent-swarm` | Sin defensa técnica ante inyección | Plantillas `untrusted_*` (solo prompt) y etiqueta de confianza calculada por código; sin escape ni restricción de herramientas | Matiz: hay una señal de confianza, pero no se usa para limitar |
| `agent-swarm` | Review gates "por prompt" | Confirmado: `allowMerge` solo cambia el texto del prompt y nada bloquea `gh pr merge`; `/finish` es autoinforme | Aceptación por código sigue ausente |
| `agent-swarm` | Endurecimiento Docker opt-in | Confirmado, y solo en el provider de pools (`docker.ts:71`): los workers del compose de ejemplo no llevan `cap_drop`, `security_opt` ni `mem_limit` | El aislamiento por defecto es más flojo que "Docker" a secas |

### 12.2 Lo que se confirma

- **`agent-orchestrator` corre de verdad en Windows nativo**: ConPTY real con `go-pty`, tests con `cmd.exe` real y un job `windows-latest` en CI (que ejecuta solo tests seleccionados). Es el hallazgo más útil para una prueba en este equipo.
- `cli-agent-orchestrator`: auth apagada por defecto con 123 rutas (el propio docstring dice "INERT"), `ImportError` en `win32` sin camino alternativo, y la "falsa victoria" **sin mitigación en código** (el broker la detecta a posteriori y el supervisor sigue recibiendo `success:true`).
- `cli-agent-orchestrator`: los modos de fallo de `get_status` son **opuestos entre proveedores** (Claude cae en `UNKNOWN`, Kiro en `PROCESSING`).
- `gastown`: `gt sling` es idempotente solo con target inequívoco y su rollback es best-effort.
- `agent-swarm`: el sweep del watchdog tiene llamador en producción (`http/index.ts:952-953`), igual que `queue-stall-alarm.ts`; el CAS sobre `lastUpdatedAt` y el *resume pinning* están en código. `scrubSecrets` es amplio (212 ficheros, con barrido de arranque que reescribe lo ya guardado). El gate de Slack con allowlist vacía deja entrar a todos, y el HMAC de GitHub es correcto.

### 12.3 Hallazgos nuevos que merecen vigilancia

- **En Codex, el modo `default` de `agent-orchestrator` equivale a bypass total** (`--dangerously-bypass-approvals-and-sandbox` en TUI; `never` + `danger-full-access` en Chat). Quien lance un worker Codex sin elegir modo no recibe aprobaciones.
- `gastown` tiene soporte de **psmux** (tmux para Windows) en el código, sin verificar que funcione. Matiza el "WSL para casi todo".
- `agent-swarm`: un hook de extensión (`pre.heartbeat.remediate`) puede vetar o cambiar la remediación del watchdog; con la clave compartida, `X-Agent-ID` es un header declarado por el cliente; y no hay timeout duro por tarea.
- `cli-agent-orchestrator`: un bug conocido (#735/#407) en que un `COMPLETED` de la vuelta anterior puede leerse como el de la actual, y un `DiagnosticBundle` que devuelve inputs sin redactar secretos.

### 12.4 Efecto sobre las conclusiones

1. **El Watchdog "en capas" de `gastown` es menos determinista de lo que parecía.** Parte de la cadena (Boot, Deacon, Witness) son agentes LLM, y la reacción del daemon a un polecat caído es un mail al Witness. Refuerza el hallazgo de §11.3 sobre la heterogeneidad del Watchdog, y deja a `agent-swarm` como el único de la ampliación con un clasificador de atasco **en código y con llamador** (verificado en `heartbeat.ts`). Matiz: solo remedia las clases A y B; la C solo registra.
2. **El hueco 1 (aislamiento) se agranda**: `gastown` pasa de "sandbox en propuesta" a "sin sandbox", y el proxy mTLS no se usa por defecto.
3. **El hueco 4 (coste por tarea) sigue abierto** con más evidencia: el único tope existente (`max_polecats`) viene apagado.
4. **Refuerzo de la tesis de §11.1**: en `gastown`, como en `cc-super-agent`, **más lógica de la que parece vive en el prompt**, aunque el repo sea Go y tenga mucho código. Leer los llamadores importa tanto como leer las funciones: dos piezas "implementadas" (la detección de atascos del Deacon y la cola de merge por lotes) no tienen llamador automático en producción.
5. **La confianza en las notas mejora, no se completa**: las cuatro tablas de confianza siguen en "baja" por cobertura global.

## Anexo: ficheros

- Notas por repositorio, con tabla de evidencias `ruta:línea`: `study/dsh-superagent-prototype.md`, `study/cc-super-agent.md`, `study/deepseek-super-agent.md`, `study/super-super-harness.md`, `study/deer-flow.md`, y de la ampliación `study/agent-orchestrator.md`, `study/agent-swarm.md`, `study/cli-agent-orchestrator.md`, `study/gastown.md`.
- Tablas comparativas con métricas de GitHub (estrellas, forks, commits...) y matrices de capacidades: `TABLAS.md`.
- Código descargado (no tocar): `references/`.
- Corrección: las notas de `study/super-super-harness.md` dicen "un único commit visible"; era un artefacto del clon superficial (el repo tiene 6 commits y `v0.3.0`).
