# Tablas comparativas

Complemento de `STUDY.md`. Datos de GitHub consultados el **2026-11-21** con `gh api` (los cinco primeros en la primera ronda; los cuatro de la ampliación, ese mismo día). El resto procede de las notas de `study/` (lectura de código y documentación, sin ejecutar nada).

**Aviso de confianza.** Las columnas de los cuatro repos de la ampliación (`agent-orchestrator`, `agent-swarm`, `cli-agent-orchestrator`, `gastown`) salen de una lectura del 3 al 20 % de cada repositorio. Una celda con ❓ en esas columnas suele significar "no encontrado en lo leído", no "no existe". Las columnas de los cinco primeros tienen cobertura mayor (ver `STUDY.md` §0 y §10).

Leyenda de las tablas de capacidades:
- ✅ implementado en código (visto en las notas)
- ⚠️ parcial, solo por prompt, opt-in, o con limitaciones serias
- ❌ no existe
- ❓ no verificado o no encontrado en la parte leída

Abreviaturas de columna: **dsh** = `dsh-superagent-prototype`; **cc** = `cc-super-agent`; **Lofi** = `LofiSu/dsh-super-agent`; **sSH** = `super-super-harness`; **deer** = `deer-flow`; **AO** = `agent-orchestrator`; **swarm** = `agent-swarm`; **CAO** = `cli-agent-orchestrator`; **gas** = `gastown`.

## 1. Popularidad y comunidad (GitHub)

| Repo | ⭐ Estrellas | Forks | Watchers | Issues abiertas | Contribuyentes | Commits | Releases |
|---|---:|---:|---:|---:|---:|---:|---|
| [`bytedance/deer-flow`](https://github.com/bytedance/deer-flow) | **85.571** | 12.070 | 336 | 1.128 | **681** | 4.931 | 3 (`v2.2.0`) |
| [`gastownhall/gastown`](https://github.com/gastownhall/gastown) | **18.862** | 1.704 | 105 | 532 | n/d ² | **7.783** | `v1.2.1` |
| [`OrchestratorInc/agent-orchestrator`](https://github.com/OrchestratorInc/agent-orchestrator) | **14.996** | 2.066 | 43 | 1.061 | n/d ² | 4.379 | `v0.15.4-nightly.202611202121` |
| [`deepseek-ai/dsh-superagent-prototype`](https://github.com/deepseek-ai/dsh-superagent-prototype) | **24.127** | 2.997 | 183 | 0 | 1 | **1** | ninguna |
| [`awslabs/cli-agent-orchestrator`](https://github.com/awslabs/cli-agent-orchestrator) | 1.619 | 342 | 5 | 179 | n/d ² | 797 | `v2.5.3` |
| [`desplega-ai/agent-swarm`](https://github.com/desplega-ai/agent-swarm) | 1.097 | 184 | 21 | 37 | n/d ² | 3.989 | `v2.14.2` |
| [`LofiSu/dsh-super-agent`](https://github.com/LofiSu/dsh-super-agent) ¹ | 146 | 18 | 0 | 8 | 1 | 375 | ninguna |
| [`yingpengma/cc-super-agent`](https://github.com/yingpengma/cc-super-agent) | 7 | 1 | 0 | 1 | 1 | 29 | ninguna |
| [`ketchupz1999/super-super-harness`](https://github.com/ketchupz1999/super-super-harness) | 1 | 0 | 0 | 4 | 1 | 6 | `v0.3.0` |

¹ El repositorio se llamaba `deepseek-super-agent` y GitHub lo redirige desde ese nombre. La carpeta local conserva el antiguo.

² La API de contribuyentes devolvió 404 para estos cuatro el 2026-11-21. No se ha podido contar y no se inventa la cifra. Los commits sí se obtuvieron (por el contador de páginas de `commits?per_page=1`).

### Ritmo y edad

| Repo | Creado | Último push | Días de vida | ⭐ por día | Forks / ⭐ |
|---|---|---|---:|---:|---:|
| `deer-flow` | 2025-05-07 | 2026-11-21 | 563 | 152 | 14,1 % |
| `dsh-superagent-prototype` | 2026-10-31 | 2026-11-03 | 21 | **1.149** | 12,4 % |
| `gastown` | 2025-12-16 | 2026-11-14 | 340 | 55,5 | 9,0 % |
| `agent-orchestrator` | 2026-02-13 | 2026-11-21 | 281 | 53,4 | 13,8 % |
| `cli-agent-orchestrator` | 2025-07-29 | 2026-11-21 | 480 | 3,4 | 21,1 % |
| `agent-swarm` | 2025-12-19 | 2026-11-21 | 337 | 3,3 | 16,8 % |
| `LofiSu/dsh-super-agent` | 2026-10-31 | **2026-11-21** | 21 | 7 | 12,3 % |
| `cc-super-agent` | 2026-11-01 | 2026-11-02 | 20 | 0,35 | 14,3 % |
| `super-super-harness` | 2026-11-03 | 2026-11-04 | 18 | 0,06 | 0 % |

Lecturas:
- **El prototipo oficial tiene 24k estrellas con un solo commit y 0 issues.** Es tirón de marca (DeepSeek) y de curiosidad; no indica uso real. Lleva 18 días sin tocarse.
- **Siguen vivos hoy seis de los nueve**: `deer-flow`, `agent-orchestrator`, `agent-swarm`, `cli-agent-orchestrator` y `LofiSu` con push el 21-11, y `gastown` con push hace 7 días. Los otros tres (`dsh-superagent-prototype`, `cc-super-agent`, `super-super-harness`) llevan entre 17 y 19 días parados desde su primer empujón; de `cc-super-agent` y `super-super-harness` casi nadie ha probado nada fuera de su autor.
- **Los cuatro de la ampliación son proyectos con ritmo real de desarrollo**: de 797 a 7.783 commits y releases con versión. Los cuatro del linaje DeepSeek de un solo autor tienen entre 1 y 375.
- **Estrellas y rigor visible no van a la par.** `gastown` (18,9k ⭐) se declara "experimental" en su propio `SECURITY.md:40`. `agent-swarm` (1,1k ⭐) y `cli-agent-orchestrator` (1,6k ⭐) son los de la ampliación con más señales de ingeniería en las notas (CI con muchos gates, e2e, documentación que se mantiene contra el código). Es una lectura de señales declaradas o contadas, no una medida de calidad.
- La ficha de GitHub de `agent-orchestrator` dice **TypeScript**; el daemon es **Go** (ver `STUDY.md` §10.5).

## 2. Identidad técnica

| | dsh | cc | Lofi | sSH | deer | AO | swarm | CAO | gas |
|---|---|---|---|---|---|---|---|---|---|
| Lenguaje principal | TypeScript | Bash + JS | TypeScript | Python | Python + Next.js | **Go** + Electron/React + Expo | Bun / TypeScript | Python + Rust (TUI) + React | Go |
| Licencia | MIT | MIT | MIT | MIT | MIT | Apache-2.0 | MIT | Apache-2.0 | MIT |
| Origen | Oficial (DeepSeek), prototipo | Particular | Particular | Particular | Empresa (ByteDance) | Organización, producto comercial | Organización (desplega-ai) | Empresa (AWS Labs) | Organización, un solo autor domina los commits |
| Tamaño del código | ~12k líneas sin tests | ~1.300 líneas | ~35k en `src/` | ~27k con tests | ~1.020 `.py` + frontend | 7.978 ficheros (~3.106 `.go`, ~3.319 `.ts/.tsx`) | `src/` ~2.885 ficheros | 3.033 ficheros (477 `.py` fuera de `test/`) | ~1.232 `.go` |
| Dependencias | Cordis, `@deepseek-ai/*` alpha, zod | Ninguna | Solo `@modelcontextprotocol/sdk` | Solo stdlib | LangGraph, LangChain, FastAPI | chi, Cobra, SQLite + sqlc | `bun:sqlite`, sqlite-vec, Biome, Ink | FastAPI, SQLAlchemy, FastMCP, tmux ≥ 3.3 | `beads v1.0.5`, Dolt |
| Harness de agente | DeepSeek Harness | Claude Code | Runtime ReAct propio | Claude Code (Hermes experimental) | LangGraph | 44 harnesses (19 con driver Chat; ~25 solo TUI) | Claude Code, Codex, pi, opencode, Gemini… | 13 CLIs (Kiro, Claude Code, Codex…) | Claude Code, Copilot, Codex, Gemini, Cursor, OpenCode… |
| Almacenamiento | SQLite | Markdown en disco | SQLite (`node:sqlite`) | SQLite WAL | SQL + checkpoints | SQLite (goose + sqlc), CDC por triggers | SQLite WAL + sqlite-vec | SQLite + ficheros | **Dolt** (SQL versionado) |
| Sistema probado | No indicado | Linux | macOS y Linux | macOS | Docker / Linux | Windows x64 (empaquetado verificado según `STATUS.md`); macOS/Linux pendiente | Contenedores Linux | macOS (arm64, x86_64) y Linux x86_64 | ❓ |
| Corre en Windows | ❓ | ❌ (WSL2) | ⚠️ sin sandbox | ❌ (WSL2) | ❓ | ✅ nativo (ConPTY, sin tmux) | ⚠️ Docker Desktop, sin probar | ❌ (WSL2; `ImportError` en `win32`) | ⚠️ CLI mínimo; WSL para tmux y Mayor (hay soporte de psmux en el código, sin verificar) |

## 3. Arquitectura

| Capacidad | dsh | cc | Lofi | sSH | deer | AO | swarm | CAO | gas |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Router y Worker separados | ✅ | ✅ (por prompt) | ✅ | ✅ | ❌ (lead único) | ✅ | ✅ | ✅ | ✅ |
| Gate determinista antes del LLM | ✅ | ❌ | ✅ | ✅ (reglas por fuente) | ⚠️ (middlewares) | ✅ (reacciones en Go; sin router LLM) | ✅ (elegibilidad por rol y capacidades) | ❌ | ⚠️ (`gt sling` y scheduler, tras decidir el Mayor) |
| Eventos externos / triggers | ✅ GitHub, Feishu | ❌ | ✅ varios canales | ✅ GitHub (Feishu pendiente) | ⚠️ chats IM, sin proactividad | ✅ observer de PR/CI (30 s), intake de issues, cron | ✅ Slack, GitHub, GitLab, email, Jira, Linear | ⚠️ flows programados | ❓ |
| Worker permanente con identidad | ❌ (uno por Ticket) | ❌ (uno por tarea) | ❌ | ✅ | ❌ | ❌ (una sesión por tarea) | ✅ (identidad y memoria; resume pinning) | ❌ (terminal por handoff) | ✅ (identidad persistente, sesión efímera) |
| Dependencias entre tareas | ❌ | ❌ | ⚠️ equipos | ✅ `wait_for` | ✅ `depends_on` | ❓ | ✅ `dependsOn` | ⚠️ handoff secuencial; YAML `parallel/pipeline/loop` "reservado" | ⚠️ convoys y molecules |
| Modelo distinto por rol | ✅ | ⚠️ el de Claude Code | ✅ | ✅ | ⚠️ hereda | ✅ (modelo y esfuerzo al delegar) | ✅ `modelTier` | ⚠️ `model` opcional, no en todos los proveedores | ✅ presets por runtime |
| Varios proveedores de modelo | ❌ | ❌ | ✅ OpenAI-compatible | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Memoria a largo plazo | ❌ | ⚠️ Markdown | ⚠️ embedding por hash | ⚠️ notas | ✅ | ⚠️ conversaciones durables y notas de traspaso; sin memoria semántica | ✅ híbrida con grafo y ratings | ✅ 5 ámbitos (wiki + índice SQLite) | ⚠️ ledger Dolt compartido; sin memoria semántica |
| Cola durable de efectos | ✅ | ❌ | ✅ | ✅ | ⚠️ solo lotes durables | ⚠️ outbox solo en traspasos TUI↔Chat; entrega al terminal best-effort (sin ack); dedup persistido | ❓ | ⚠️ inbox con confirmación y reconciliación | ⚠️ `sling` idempotente con rollback |
| Panel / interfaz gráfica | ✅ | ✅ | ⚠️ | ✅ | ✅ | ✅ Electron y móvil | ✅ dashboard `apps/ui` | ✅ UI React y panel de flota | ❓ (CLI `gt`) |
| Habilidades (skills) | ⚠️ | ❌ | ✅ | ❌ | ✅ (22 públicas) | ⚠️ "Cues" (prompts reutilizables) | ❓ | ✅ `skills` en perfil, `load_skill` | ❓ |

## 4. Fiabilidad y supervisión

| Capacidad | dsh | cc | Lofi | sSH | deer | AO | swarm | CAO | gas |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Watchdog explícito | ❌ | ✅ LLM por cron, solo avisa | ✅ en código | ⚠️ repartido | ⚠️ repartido | ⚠️ Reaper (5 s), solo de vida | ✅ sweep ~90 s con clasificador | ⚠️ regex sobre la pantalla del TUI | ✅ daemon (código) más Boot, Deacon y Witness (agentes LLM) |
| Detector de Worker colgado | ❌ | ✅ (juicio del LLM, 15 min) | ✅ nudge y luego fail | ✅ 30 min | ✅ bucles y heartbeats | ❌ (`no_signal` es solo una etiqueta a los 90 s sin ningún hook; no avisa ni actúa) | ✅ clasificador A/B/C con CAS; A y B remedian, C (sesión viva) solo registra | ⚠️ solo timeout de recogida de delegación | ⚠️ automático solo el daemon sobre el Deacon (nudge 5-20 min, kill >20 min); el resto depende del prompt del Deacon |
| Reintentos con backoff | ✅ | ✅ solo errores transitorios | ✅ | ✅ | ✅ | ⚠️ sin backoff: re-poll de 30 s y dedup; tope de 3 en review; CI sin tope (un nudge por push fallido) | ✅ backoff 30/90/180 s en pools | ⚠️ reenvío del prompt hasta 3 veces tras 8 s en idle (sin backoff); 3 intentos al crear el terminal | ✅ backoff al reiniciar Dolt (5 por ventana, hasta 5 min) y agentes (30 s → 10 min) |
| Recuperación tras reinicio | ✅ | ⚠️ PID obsoleto posible | ✅ | ✅ | ⚠️ prometida, no completa | ✅ reconciliación de arranque | ✅ reboot sweep | ⚠️ limpieza fail-closed; workflows reanudables | ✅ el estado vive en hooks y beads |
| Failover del Router | ❌ | ⚠️ reinicio del bucle | ❌ | ✅ con sondas | ❌ | ❓ | ❓ | ❓ | ❓ |
| Efectos *at-most-once* | ✅ outbox | ❌ | ✅ | ✅ | ❌ | ⚠️ dedup por firma persistida | ❓ | ❌ (handoff no idempotente) | ⚠️ `sling` idempotente |
| Aceptación del resultado verificada | ❌ | ❌ | ⚠️ regex heurístico | ⚠️ verifica origen, no resultado | ✅ por código | ⚠️ estado del PR derivado de hechos de SCM; reviewer agent | ⚠️ el worker se autoinforma en `/finish`; `allowMerge` solo cambia el texto del prompt | ❌ ("falsa victoria" reconocida) | ⚠️ batch-then-bisect implementado pero sin llamador; fusiona un agente LLM |

## 5. Seguridad

| Capacidad | dsh | cc | Lofi | sSH | deer | AO | swarm | CAO | gas |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Sandbox real del Worker | ❌ `danger-full-access` | ❌ | ⚠️ macOS y Linux | ❌ permisos totales | ⚠️ `local` flojo; `aio` monta el socket de Docker | ❌ solo worktree | ⚠️ Docker; `DOCKER_HARDENED` apagado por defecto y solo en el provider de pools; los workers del compose no llevan `cap_drop`; red abierta | ⚠️ nada local; pods desechables en K8s (ejemplo) | ❌ solo worktree; proxy mTLS implementado pero sin uso por defecto; sandbox no implementado |
| Allowlist de quién origina trabajo | ✅ | ❌ | ✅ | ✅ por fuente | ⚠️ auth por usuario | ⚠️ monousuario local | ⚠️ opt-in (Slack vacío = abierto); GitHub con HMAC | ❌ sin auth por defecto | ❓ |
| Defensa contra prompt injection | ⚠️ textual | ⚠️ solo en el Watchdog | ⚠️ textual + validación | ⚠️ escapado + etiquetas | ⚠️ rechaza `system` del cliente | ⚠️ caracteres de control; log de CI entre fence; comentarios de review en crudo | ⚠️ solo de prompt: plantillas `untrusted_*` y etiqueta de confianza por código; sin escape ni restricción de herramientas | ⚠️ textual (`Security Constraints`) | ❓ no encontrada |
| Aprobación humana de acciones | ⚠️ por prompt | ⚠️ permisos de Claude Code | ✅ por CLI | ⚠️ `permission` no se aplica | ✅ para red | ⚠️ `PermissionMode`; en Codex `default` equivale a bypass total; guarda que no responde diálogos por el humano | ⚠️ `approval requests`; merge por prompt y `allowMerge` | ⚠️ allowlist; Claude Code con `--dangerously-skip-permissions` | ⚠️ escalado al Overseer; `--dangerously-skip-permissions` |
| Gestión de credenciales | ✅ la más cuidada | ❌ | ✅ Vault y redacción | ✅ ficheros 0600 | ✅ limpia el entorno | ✅ "nunca lee un token"; usa la config de cada CLI | ⚠️ `scrubSecrets` amplio (212 ficheros; Slack, BD y logs; barrido de arranque), pero secretos en el entorno del worker | ⚠️ cada CLI con su auth; Pod Identity en K8s | ❓ |
| Autenticación de la web / API | ✅ loopback + same-origin | ⚠️ token en URL, HTTP | ✅ token | ✅ ticket de un uso + cookie firmada | ✅ JWT + CSRF | ⚠️ loopback sin auth; LAN opt-in con bearer y lockout | ⚠️ RBAC solo para principales de usuario; la clave compartida es operador y `X-Agent-ID` lo declara el cliente | ❌ por defecto ("superficie de ejecución sin autenticar") | ❓ |
| Política de seguridad publicada | ⚠️ en docs | ❌ | ✅ `docs/SECURITY.md` | ⚠️ en docs | ⚠️ `SECURITY.md` mínimo | ❓ | ⚠️ `SECURITY.md` solo con un correo | ✅ `SECURITY.md` extenso | ⚠️ `SECURITY.md`: experimental, recomienda entornos aislados |

## 6. Madurez

| | dsh | cc | Lofi | sSH | deer | AO | swarm | CAO | gas |
|---|---|---|---|---|---|---|---|---|---|
| Estado declarado | "Research prototype" | "Experimental" | v0.1.0, un solo propietario | v0.3.0, uso personal | Sin release oficial en `SECURITY.md` | Producto comercial: releases firmadas, auto-update | v2.14.2; algunas partes BETA/alpha | v2.5.3 (el `pyproject.toml` del clon dice 2.5.4) | "Experimental" (`SECURITY.md:40`); GitHub publica `v1.2.1` mientras `SECURITY.md` soporta "0.1.x" |
| Tests | ~794 casos | **ninguno** | 148 ficheros | 866 `def test_` | ~1.377 ficheros | ~1.633 `_test.go` y ~1.313 `*.test.ts(x)` | 1.132 `*.test.ts`; 198 escenarios e2e | 907 ficheros `.py` en `test/` | 569 `_test.go` |
| Tests con modelo real | ❌ host simulado | — | ⚠️ 15 saltados | ❌ harness falso | ⚠️ opt-in | ❓ | ❓ | ⚠️ e2e reales solo manuales (`-m e2e`); el CI usa e2e con `mock_cli` y tests de proveedor | ❓ |
| Documentación | 9 docs numerados | README y prompts | ~45 docs, algunos desfasados | 5 docs | Muy extensa, desigual | Muy detallada; `documentation-map.md` fija qué doc manda | Runbooks mantenidos contra el código | ~40 docs + Docusaurus; pide verificar afirmaciones contra código | Extensa; muchos diseños en estado "Proposal"; citas internas desfasadas |
| Escrito en gran parte por agentes | ✅ lo declara | ❓ | ✅ (`CLAUDE.md` orquesta) | ❓ | ❓ | ❓ | ❓ | ❓ | ❓ |
| Verificación de nuestras notas | Alta | Alta | Media (muestreo) | Alta | Media-baja (muestreo) | **Baja** (3-4 %; ampliada, ver `STUDY.md` §12) | **Baja** (~15 %; ampliada §12) | **Baja** (~12 %; ampliada §12) | **Baja** (15-20 %; ampliada §12) |

Los tests se han contado por ficheros, no ejecutados. Una cifra de ficheros de test no es cobertura.

## 7. Qué ofrece cada uno (una línea)

| Repo | Lo más valioso | Lo más débil |
|---|---|---|
| `dsh-superagent-prototype` | Dominio muy pensado: Mandate frente a Subscription y gestión de credenciales | Sin sandbox, sin Watchdog, atado a DeepSeek Harness alpha |
| `cc-super-agent` | Mínimo, legible y sobre Claude Code sin parches | Casi todo es prompt, sin tests, solo Linux |
| `LofiSu/dsh-super-agent` | El más completo en código: puertos por rol, Watchdog, aprobaciones | Un solo autor, docs desfasados, autoevaluación sin contrastar |
| `super-super-harness` | Coordinación robusta (failover, at-most-once, dependencias) y desacople de modelos | Sin sandbox, probado solo en macOS |
| `deer-flow` | Ecosistema grande, aceptación por código, skills | No es Router/Worker, sandbox por defecto flojo, enorme |
| `agent-orchestrator` | Windows nativo (ConPTY); nudges con dedup persistido; estado derivado en lectura; guarda que no responde diálogos de permisos por el humano | Sin sandbox; sin detector de worker atascado; ninguna defensa semántica ante un comentario de review malicioso que se pega como instrucción |
| `agent-swarm` | API única dueña de la base de datos; clasificador de atasco con CAS y *resume pinning*; RBAC con cobertura comprobada en CI | Docker con endurecimiento opt-in y red abierta; credenciales en el entorno; clave compartida = operador; la clase C del watchdog solo registra; sin tope de coste por tarea |
| `cli-agent-orchestrator` | Broker K8s de privilegios mínimos con workers desechables; techo `delegateTools`; callback garantizado | Sin autenticación por defecto; estado leído por regex del TUI ("falsa victoria" a los 3,1 s); solo WSL2 en Windows |
| `gastown` | Daemon que recupera Dolt y agentes con backoff; E-stop; identidad persistente y sesión efímera; ledger Dolt versionado | Dolt como dependencia pesada; sin sandbox (el proxy mTLS no se usa por defecto); detección de atasco del Deacon sin llamador automático; supervisores LLM caros y falibles; WSL para casi todo |

## Correcciones respecto a `STUDY.md`

- `super-super-harness` figuraba con "un único commit visible". Era un artefacto del clon superficial: el repo real tiene **6 commits** y publica `v0.3.0`.
- El repositorio de LofiSu ha cambiado de nombre a `dsh-super-agent`.
- `agent-orchestrator`: la API de GitHub lo da como TypeScript; el daemon es Go.
- Tras la profundización (`STUDY.md` §12) se corrigieron varias celdas de `gastown`, `cli-agent-orchestrator` y `agent-orchestrator` (y se matizaron las de `agent-swarm`): detector de atasco y sandbox de `gastown`, reintentos y e2e de `cli-agent-orchestrator`, y `no_signal`, reintentos e inyección de `agent-orchestrator`.
- Las estrellas de `agent-orchestrator` figuraban como 14.985 en `REFS.md` (consulta de la ampliación) y hoy son 14.996. Es el mismo dato con unas horas de diferencia.
