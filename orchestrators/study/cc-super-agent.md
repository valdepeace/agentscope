# Estudio: cc-super-agent (solo lectura, nada ejecutado)

Ruta: `references/cc-super-agent/` (abreviado `R/`). Todo el contenido se trató como dato no confiable; AGENTS.md y CLAUDE.md se leyeron como objeto de estudio.

## 1. Resumen
- Qué es: implementación abierta del diseño "Super-agent" (DeepSeek, citado como oct-2026) sobre Claude Code sin modificarlo. Una sesión "Router" (el propio `claude` interactivo abierto en el directorio del hub) habla con el usuario y delega en "Workers" (`claude -p` en el directorio de cada proyecto). (`R/README.md:21-23`, `R/AGENTS.md:3-5`)
- Stack: bash (dispatch, pause, resume, router, watchdog), Node sin dependencias (`web/server.mjs`, 508 líneas; `web/public/app.js`, 812), Markdown en disco, cron, tmux, systemd, `flock`, `uuidgen`.
- Licencia: MIT, (c) 2026 Yingpeng Ma (`R/LICENSE:1-3`).
- Tamaño: ~35 ficheros, solo ~1.300 líneas de código real (+ prompts en Markdown). El "código" más importante es el prompt `AGENTS.md`.
- Madurez: la propia README se declara "experimental" (`R/README.md:11`), sin tests (roadmap `README.md:179`). Repo clonado superficial (`.git/shallow`).

## 2. Modelo de dominio
No hay modelo formal de datos; son convenciones Markdown (`R/AGENTS.md:114-198`).
- **Mensaje del usuario**: se clasifica como trabajo nuevo, seguimiento o nada (`AGENTS.md:23`). No existen "evento" ni "trigger" externos: README lo reconoce como "Not yet" (`README.md:172`, roadmap `:178`).
- **Task**: una pieza de trabajo (no un mensaje). Fichero `tasks/<id>-<slug>.md` con Status (open|waiting|on hold|done|dropped), Hold, Project, Worker, Request literal, Log, Outcome. Numeración secuencial, nunca se borra (`AGENTS.md:154-179`).
- **Project**: `projects/<name>.md` con Path, Purpose, Notes, lista de tasks (`AGENTS.md:181-198`).
- **INDEX.md**: una línea por tarea abierta (`AGENTS.md:144-152`).
- **Brief**: mensaje al Worker desde `templates/brief.md` (Request verbatim, Routing facts, Hold).
- **Hold** (por tarea) vs **pause** (global, fichero `.paused`) (`AGENTS.md:57, 99`).
- **Run**: estado de ejecución de una tarea en `.runs/<task-id>/` (`bin/dispatch:26-29`).
- **Thread**: no existe como concepto; la continuidad de conversación es el session-id fijo por tarea (task 1:1 session-id 1:1 Worker).
- **Twin**: modelo del juicio del usuario (ver 9).
Relaciones: Project 1-N Task; Task 1-1 sesión Claude (session-id) 1-N rondas; cada ronda consume N briefs encolados.

## 3. Router
- Es la sesión interactiva de Claude Code en el directorio del hub; `CLAUDE.md` solo contiene `@AGENTS.md` (`R/CLAUDE.md:1`). Todo su comportamiento está en prompt, no en código.
- Decide: nuevo/seguimiento/nada; vincular a tarea existente solo si el mensaje apunta claramente (`AGENTS.md:50`); registrar antes de actuar (`:213`); confirmar en una frase (`:103`); reescribir informes en vez de reenviarlos (`:107`); mantener holds (`:57`).
- Modelo: no fijado; usa el de Claude Code del usuario (no encontrado ninguna selección de modelo). Router no hace el trabajo ni lee código ajeno (`AGENTS.md:27`).
- Herramientas: las de Claude Code (Bash para `bin/dispatch` en background, `SendMessage` a sesiones vivas, Read/Write de los .md). No hay filtrado de eventos en código: el "filtrado" es la política de comunicación del prompt (hablar solo por resultado, decisión o fallo, `AGENTS.md:104-108`).
- Persistencia de la sesión: `.router/session-id` y `claude --resume` (`R/router/loop.sh:15-23`); reinicio tras 5 s si sale (`:19-27`). Evita `--continue` a propósito (`AGENTS.md:65`).
- Corre en tmux (socket `ccsa`, sesión `ccsa-router`) bajo systemd (`router/start.sh:7`, `router/install.sh:18-36`).

## 4. Worker
- Aislamiento: directorio del proyecto (`cd "$project" && claude -p ...`, `bin/dispatch:139`) y sesión propia con UUID fijo. Sin contenedor ni sandbox; mismo usuario y máquina ("Remote execution: Not yet", `README.md:173`). Carga el CLAUDE.md y memoria del proyecto.
- Arranque: primera llamada `--session-id`, siguientes `--resume` según exista el transcript `~/.claude/projects/*/<sid>.jsonl` (`dispatch:97,136`). Se añade `templates/worker-rules.md` con `--append-system-prompt` en cada ronda (`:94-95,139`), salida `--output-format json` a `.runs/<id>/result.json`.
- Reporte: último mensaje de cada turno con formato `Status: ready|blocked|failed|dropped / Result / Question` (`templates/worker-rules.md:14-18`). El Router se entera cuando el comando background termina. El formato lo impone solo el prompt; nada lo valida en código.
- Ciclo de vida: `queued -> running -> retrying n -> exited <code>` o `paused` (`dispatch:29`). pid, started, ended en `.runs/<id>/`.
- Cola de mensajes: si el Worker está vivo (pid con `kill -0`), el brief se copia a `pending/<ns>.md` y se sale; el dispatch en ejecución, al acabar la ronda, junta todos los pendientes en un solo prompt y reanuda (`dispatch:63-66,106-131`). Exclusión con `flock` sobre `run/lock` (fd 8).
- Límite de concurrencia: `CCSA_MAX_WORKERS` (3) con slots por `flock -n` sobre `.runs/.slots/N`, sondeo cada 10 s (`dispatch:48,84-91`).
- Reintentos: hasta 3, backoff 30*2^(n-1) s, solo si stderr/result.json casan una regex de errores transitorios (429/5xx/overloaded/network...) (`dispatch:98-101,144-150`); en el reintento sustituye el prompt por "comprueba qué hiciste ya" (`:148-150`).
- Alternativa: sesión viva del usuario vía `SendMessage` (`AGENTS.md:42`).

## 5. Estado y memoria
- Todo en disco, Markdown + ficheros planos de una línea: `INDEX.md`, `tasks/`, `projects/`, `.runs/<id>/{session-id,project,pid,status,started,ended,result.json,stderr.log,briefs.log,pending/,lock,transcript}`, `.router/{session-id,restarts.log,transcript,tmux-alive}`, `.watchdog/{lock,logs/}`, `.webui/token`, `.paused`.
- Tres capas de memoria (`AGENTS.md:114-122`): `~/.claude/CLAUDE.md` global (reglas para todos los Workers), memoria de Claude Code del directorio (preferencias; Workers no la leen, `:128`), registros de trabajo.
- "Chain of recall": el Router empieza por INDEX.md y sigue enlaces relativos un salto cada vez (`AGENTS.md:142`).
- Regla "registrar primero, luego actuar" para sobrevivir a compactación/reinicio (`AGENTS.md:211-213`). Crear/modificar memoria requiere mostrar el texto y aprobación explícita (`:130`).
- Separación framework/datos personales con `.gitignore` en lista blanca (`R/.gitignore:1-18`).
- Sin base de datos ni locking sobre los .md (los escribe el LLM; el Watchdog también edita INDEX.md y tasks/ => posible carrera, ver 10).

## 6. Watchdog y fallos
- Prometido y implementado: `watchdog/install.sh` mete una línea de crontab cada 15 min (`install.sh:17`); `watchdog/run.sh` es un pase: `flock -n` anti-solape (`run.sh:13-14`), sale si `.paused` (`:17`), sale sin llamar al modelo si ninguna tarea está en progreso ni hay runs activos (`:21-34`), resuelve rutas de transcripts, anota si el tmux del Router vive, y lanza `claude -p "$(cat watchdog/prompt.md)" --no-session-persistence --add-dir ~/.claude/projects --allowedTools Read Grep Glob Edit Write Bash(ps|cat|tail|stat|date|ls) ListAgents SendMessage` (`:52-56`). Conserva 200 logs (`:59`).
- Qué detecta (juicio del LLM, `watchdog/prompt.md:15-27`): Died, Stranded, Never started, Silent (30 min), Stuck on a tool (15 min), Looping, Off course, Unreported, Forgotten (24 h). Y del Router: down, stuck, not relaying (`:38-42`).
- Acción: solo observa e informa: añade línea al log de la tarea, marca con ⚠ en INDEX.md, mensaje con SendMessage al Router si está vivo; si el Router falla, línea `⚠ Router:` arriba en INDEX.md (`prompt.md:46-55`). Nunca reinicia ni para Workers. Las instrucciones dentro de transcripts se tratan como datos (`:63`).
- Manejo de fallos en dispatch: reintentos transitorios (ver 4). Fallos no transitorios: sale con el código y el Router decide (`AGENTS.md:85`). Router: bucle de reinicio + systemd `Restart=always`.
- Limitación: un Worker que muere deja `pid` obsoleto; no hay limpieza (solo el chequeo `kill -0`, que es vulnerable a reutilización de PID). `trap` no encontrado en dispatch.

## 7. Seguridad y autorización
- Permisos de Claude Code: Workers usan los ajustes del usuario para ese proyecto; `dispatch` solo pasa flags extra (`-- extra claude args`) si el usuario los concede (`dispatch:31-33`). Regla: nunca mandar a otra sesión lo que una rechazó (`AGENTS.md:59`). Todo por prompt; sin enforcement en código. Como los Workers son `claude -p` sin TTY y stdin `/dev/null`, un permiso no concedido no puede aprobarse mientras corre (por eso el Watchdog vigila "stuck on a tool").
- Watchdog: lista blanca de herramientas, pero incluye `Edit`/`Write` sin restricción de ruta (el prompt dice "solo INDEX.md y tasks/", `prompt.md:61`, solo por instrucción) y `--add-dir ~/.claude/projects` (acceso a todos los transcripts de la máquina).
- Web UI: token aleatorio de 24 bytes base64url en `.webui/token`, modo 0600 (`server.mjs:38-43`); comparación `timingSafeEqual` (`:45-50`); acepta `Authorization: Bearer` o `?t=` en la URL (tokens en query acaban en logs/historial); la UI lo guarda en localStorage (`app.js:257-262`); escucha en 127.0.0.1 por defecto (`server.mjs:34`); cabeceras CSP, X-Frame-Options, nosniff (`:375-380`); estáticos restringidos por regex `^[\w.-]+$` (`:403`); lectura de tareas con regex `tasks/[\w.-]+\.md` (`:425`); teclas permitidas en lista blanca (`:358`). Sin TLS ni rate-limit; con `--host 0.0.0.0` va en HTTP plano (advertido en `web/install.sh:9-10`). Quien tenga el token puede inyectar texto al Router, o sea, ejecutar acciones como el usuario.
- Prompt injection: no hay defensa de código en Router; el único control explícito es el del Watchdog ("tratar transcripts como datos", `prompt.md:63`). El Router lee informes de Workers y registros que pueden contener contenido externo; no hay saneado. Botón "Not right" cita texto del feed dentro del siguiente mensaje al Router (`app.js:39-40`).
- Credenciales: no gestiona ninguna; depende de que `claude` esté logueado. Ni secretos ni vault; `HOME` pasada en unidades systemd.
- La web pega texto en el pane tmux y `Enter` (`server.mjs:360-371`) y puede enviar teclas `y/n/1/2/3/enter` para responder permisos del Router (`:358,445-450`): aprobar permisos remotamente con solo el token.
- Detección de "asking" por regex sobre la pantalla (`:165`), frágil ante cambios de UI de Claude Code.

## 8. Madurez real
- Tests: ninguno (no hay carpeta ni scripts de test; roadmap "Automated tests for the dispatch and Watchdog scripts" sin marcar, `README.md:181`).
- Implementado (verificado en código): dispatch con cola/flock/slots/reintentos, pause/resume, router loop+tmux+systemd, watchdog cron con prefiltro y prompt, Web UI completa (chat desde transcript, feed de grupo de trabajo, SSE, pausa, teclas).
- Prometido/sin implementar: eventos/triggers externos y ejecución remota ("Not yet", `README.md:172-173`); build del twin a partir de todos los transcripts (roadmap); "Watchdog flags it within 15 minutes" depende de cron y del LLM; "Workers never get permissions you did not grant" es política por prompt, no se impone en código. El twin es solo un método (`twin/README.md`): no hay herramienta; `twin/model.md`, `review.md`, `sources.md` no existen en el repo (por diseño, personales).
- Benchmark (p95 15,7 vs 29,5 min) es de DeepSeek, no del repo (`README.md:150-161`). Versión probada: Claude Code 2.1.316 (`README.md:90`).
- Parser de transcripts depende de formato interno no documentado de Claude Code (`origin.kind`, `isSidechain`, `task-notification`, `server.mjs:92-124`): frágil.

## 9. Ideas originales / distintivo
- **Twin** (`twin/README.md`): método para modelar el juicio del usuario a partir de desalineaciones (correcciones/rechazos/overrides) en sus transcripts, no de lo que acepta ("ok" no se registra, `:24`). Clasifica causas (juicio, conocimiento privado, límite humano, ya cubierto, propósito pasajero, `:29-35`); el twin guarda "la mente, no las citas", en primera persona, con evidencia aparte en `review.md` (`:43-54`); "límite humano" no se hereda al twin; decide solo lo claro y pregunta solo lo dudoso (`:56-60`); marca puntos "ask the user". Es prompt-only.
- Router puro que no trabaja, para estar siempre libre; briefs verbatim sin planes del Router (`AGENTS.md:50`); reglas fijas en system prompt de Worker (no en cada brief).
- Un único Markdown wiki de 3 capas con navegación de un salto.
- Watchdog externo con LLM (cron + `claude -p`), con prefiltro barato sin modelo.
- Web UI sin API propia: usa tmux paste-buffer + lectura del JSONL de Claude Code; terminal y navegador son la misma conversación. "Work group" con briefs/respuestas de Workers extraídos de sus transcripts, y botón de flag que cita al Router.
- Retry que avisa al Worker de revisar lo ya hecho (idempotencia por prompt).

## 10. Carencias y huecos
- Sin tests; sin CI; todo orquestado por prompts (no verificable ni determinista).
- Solo Linux (flock, uuidgen, systemd, cron, tmux). Sin Windows.
- Estado en .md editado por LLM y por el Watchdog a la vez: sin locking ni validación de esquema; INDEX.md puede corromperse.
- PID stale y reutilización de PID; sin `trap` para limpiar `pid` al matarse dispatch; `grep` con `status` como cadena libre.
- Un único Router = cuello de botella y punto único de fallo; si la sesión se compacta pierde contexto (mitigado por registros).
- Sin aislamiento de Workers (mismo usuario/FS); sin sandbox ni límites de coste/tokens (solo nº de workers).
- Sin eventos/triggers ni ejecución remota. Sin métricas ni observabilidad más allá de logs.
- Seguridad: token en URL/localStorage, HTTP, `Edit/Write` amplios en watchdog, aprobación remota de permisos por teclas, sin defensa explícita contra prompt injection en Router/Workers.
- Dependencia de detalles internos de Claude Code (formato de transcript, textos de prompts de permiso).
- Cuestión de nombres: Router dispara `bin/dispatch` en background y espera la notificación de fin de tarea; si el Router muere, la notificación se pierde y solo el Watchdog (15 min) lo detecta.

## 11. Qué habría que cambiar para que funcione en Windows 11
Todo el código es bash/POSIX; nada funciona nativo en PowerShell/cmd.
1. **WSL2 (opción más barata)**: ejecutar todo dentro de Ubuntu en WSL2 (con systemd habilitado en `/etc/wsl.conf`, tmux, cron, util-linux). `claude` y Node instalados dentro de WSL. Cambios casi nulos; las rutas de proyecto serían `/mnt/e/...` (lentas) o clonar dentro del FS de WSL. Problema: el cron/systemd solo corre mientras WSL esté vivo; hay que arrancarlo al iniciar sesión (tarea programada que lanza `wsl`).
2. **Nativo (reescritura)**:
   - `bin/dispatch`, `pause`, `resume`, `router/*`, `watchdog/*`, `web/*.sh`: reescribir en Node o PowerShell. Sustituir `flock` (usar ficheros de bloqueo atómicos `mkdir`/`open wx`, o `proper-lockfile`), `uuidgen` (`crypto.randomUUID`), `kill -0` (`process.kill(pid,0)`/`Get-Process`), `date -Is`, `sed`, `ls`, `xargs`, `seq`, `shopt`, `$(( ))`, `${//}`, redirecciones y subshell `( cd ... && claude )` (usar `cwd` en `spawn`).
   - **tmux**: no existe en Windows. El Router y el pegado de texto/teclas de la web (`server.mjs:158-171,360-371,445-450`) dependen de `tmux load-buffer/paste-buffer/send-keys/capture-pane`. Alternativas: ejecutar el Router vía Agent SDK/`claude -p --input-format stream-json` con proceso hijo y tubería (más robusto), o ConPTY (`node-pty`) para mantener el TUI. La detección de permisos por regex del pane se perdería.
   - **systemd** -> Servicio de Windows (NSSM/WinSW) o Tarea Programada "al iniciar sesión" con reinicio automático. **cron** -> Programador de tareas (`schtasks`) cada 15 min.
   - **Rutas**: `HOME` -> `USERPROFILE` (`server.mjs:26` usa `process.env.HOME || '/root'`); `~/.claude/projects/*/<sid>.jsonl` -> `%USERPROFILE%\.claude\projects\...` (los nombres de carpeta codifican la ruta con `E--workspace-...`); glob con `ls` -> `fs.readdirSync`. Separadores `\` y unidades `E:`.
   - Watchdog: lista de herramientas `Bash(ps:*)`, `stat`, `tail`... -> en Windows usaría `PowerShell(...)` o herramientas Node propias; `ps -p` -> `Get-Process -Id`.
   - Permisos de fichero: `mode: 0o600` no aplica; proteger `.webui/token` con ACL (`icacls`).
   - Finales de línea: asegurar LF en scripts `.sh` (git `autocrlf`) si se usa WSL/Git Bash.
   - Git Bash como atajo parcial: aporta bash, `uuidgen`? (no siempre), pero no `flock` ni tmux ni systemd; no recomendable.
3. Recomendación: para el PoC, WSL2 primero; si se quiere nativo, portar `dispatch` + watchdog a Node y sustituir tmux por proceso hijo con stream-json.

---
## Evidencias
| Afirmación clave | Ruta:línea |
|---|---|
| Router = sesión interactiva del hub; no hace el trabajo | `AGENTS.md:17-27` |
| CLAUDE.md solo importa AGENTS.md | `CLAUDE.md:1` |
| Worker = `claude -p` en el dir del proyecto con session-id fijo / resume | `bin/dispatch:136-140` |
| Cola de briefs con flock y pending/ | `bin/dispatch:54-66,106-131` |
| Límite de workers por slots flock | `bin/dispatch:48,84-91` |
| Reintentos transitorios con backoff | `bin/dispatch:98-101,144-150` |
| Formato de informe Status/Result/Question solo por prompt | `templates/worker-rules.md:14-18` |
| Reglas del Worker añadidas con --append-system-prompt | `bin/dispatch:94-95,139` |
| Pausa global vía `.paused` | `bin/pause:14`, `bin/dispatch:72-76` |
| Router reiniciado en bucle con session-id fijo | `router/loop.sh:15-27` |
| Router en tmux + systemd | `router/start.sh:7`, `router/install.sh:18-36` |
| Watchdog cron cada 15 min | `watchdog/install.sh:17` |
| Prefiltro sin modelo y herramientas permitidas | `watchdog/run.sh:21-34,52-56` |
| Hallazgos del Watchdog (Died, Silent, Looping...) | `watchdog/prompt.md:15-27` |
| Watchdog nunca interviene; marca ⚠ | `watchdog/prompt.md:3,46-55` |
| Token web aleatorio 0600 y timingSafeEqual | `web/server.mjs:38-50` |
| Bind 127.0.0.1 por defecto | `web/server.mjs:34` |
| Web pega en tmux y envía teclas | `web/server.mjs:360-371,445-450` |
| Detección "asking" por regex de pantalla | `web/server.mjs:165` |
| Parseo de transcript JSONL interno de Claude | `web/server.mjs:92-124` |
| Token en localStorage | `web/public/app.js:257-262` |
| Eventos y remoto = no implementados | `README.md:172-173,178` |
| Sin tests (roadmap) | `README.md:181` |
| Estado experimental, Linux requerido | `README.md:11,90` |
| Método del twin (desalineaciones, causas, mente no citas) | `twin/README.md:22-60` |
| Datos personales fuera de git (whitelist) | `.gitignore:1-18` |
| Memoria en 3 capas; wiki de 1 salto | `AGENTS.md:114-142` |
| Registrar antes de actuar | `AGENTS.md:211-213` |
| Licencia MIT | `LICENSE:1-3` |
