# Mantenibilidad: dsh-superagent-prototype

Solo lectura sobre `references/dsh-superagent-prototype` (clon superficial de 1 commit). DATO = contado o leído con cita; INTERPRETACIÓN = juicio mío. Comandos ejecutados desde la raíz del repo con `git ls-files`, `wc` y `rg`; GitHub solo con `gh api` (GET).

## 1. Resumen y veredicto

- DATO: 216 ficheros; código fuente 15587 l. (src), tests 14960 l. (59 specs), docs 1973 l. Proporción tests/código ~0,96.
- DATO: no hay CI en el repo (sin `.github`), ni linter, ni formateador, ni pre-commit, ni CONTRIBUTING, ni AGENTS.md/CLAUDE.md; solo hay un workflow dinámico CodeQL en GitHub.
- DATO: 1 TODO en todo el repo; typecheck estricto (`strict`, `noUncheckedIndexedAccess`) y script `check` local.
- DATO: sin releases ni tags, issues y PRs desactivados, 1 commit; el propio README lo declara prototipo no listo para producción (README.md:3-5, según nota previa; releído en esa sesión).
- INTERPRETACIÓN: base de código muy cuidada en tests y docs, pero mantenible solo por su equipo original: sin canal de contribución, sin CI visible, dependencias alpha de un framework no incluido, y una capa `app` con ciclos. Riesgo de mantenimiento externo alto; calidad interna alta.

## 2. Tamaño y forma

Comandos: `git ls-files | wc -l` (216); `git ls-files src | grep -v '^src/client' | xargs cat | wc -l` (11168, 84 ficheros); `git ls-files src/client | xargs cat | wc -l` (4419, 27 ficheros, incluye client.css); `git ls-files tests | xargs cat | wc -l` (14960, 65 ficheros); `git ls-files docs | xargs cat | wc -l` (1973, 9 ficheros). Prompts: 15 ficheros; skills: 2; script de servicio: scripts/dsh-superagent (250 l.). Lenguajes según GitHub: TypeScript 2 048 810 B, CSS 40 558 B, Python 12 223 B.

Los 10 mayores ficheros de src no-client (`wc -l`):

| Fichero | Líneas | Qué hace |
| --- | --- | --- |
| src/executor/dsh.ts | 673 | Executor: slots, launch, entrega, reintentos, background |
| src/app/migration.ts | 616 | Migración esquema 10→11 y de settings |
| src/sources/github.ts | 459 | Kind GitHub y bucles de polling |
| src/core/tickets.ts | 423 | Ciclo de vida de Tickets y suscripciones |
| src/core/model.ts | 353 | Tipos y helpers del dominio |
| src/app/sessions.ts | 352 | Sesiones DSH, prompts, tools, guardas |
| src/core/evals.ts | 340 | Evals y Decisions |
| src/app/resources.ts | 308 | Recursos que lee el modelo |
| src/app/tools/main.ts | 301 | Tools del router |
| src/app/store.ts | 291 | Persistencia SQLite |

Mayores tests: runtime.spec.ts 1796, core-engine.spec.ts 1621, sources.spec.ts 1407, runtime-concurrency.spec.ts 943, migration.spec.ts 539. El mayor del client no lo medí (el mayor sin test directo es prompts.tsx, 296 l.).
Complejidad evidente (leída): src/core/tickets.ts con casos límite encadenados (`letGo`, `unfinished`, `readBefore`, src/core/tickets.ts:294-379); src/executor/dsh.ts `workerIdle` (601-671) con varios mapas de estado en memoria (211-224); src/policy.ts `revalidate` (213-240) y `gate` (191-211) con ramas condicionales densas. Estilo general: líneas muy largas y expresiones compactas (INTERPRETACIÓN: dificulta revisión).

## 3. Pruebas

- DATO: 59 specs (`git ls-files tests | grep -c '\.spec\.'`), 6 ficheros de apoyo en tests/fixtures; 11 specs son .tsx (panel). Casos `it/test` ~830 por regex (`rg -c "\b(it|test)..."`), 75 `describe`; sin `.skip/.todo/.only` (rg vacío).
- Tipos: unit (core-engine, policy, store, effects), integración con host DSH simulado en memoria (tests/fixtures/host.ts:1-4; runtime.spec, runtime-concurrency), panel con jsdom (`panel-*.spec.tsx`), script de servicio con binarios falsos y servidor local (tests/service-script.spec.ts:5,52-63), coherencia de docs (docs.spec, class-docs.spec, model-surface.spec). Sin e2e contra DSH real ni contra GitHub/Feishu reales (las URLs de api.github.com en tests son datos de fixture, p. ej. tests/sources.spec.ts:73). Algunos tests usan git y el sistema de ficheros reales (tests/environment.spec.ts:82-105, tmpdir en 14 ficheros).
- Modelo: ningún test llama a un LLM; el host y las sesiones son mocks (`vi.fn` en runtime.spec 58 usos, host.ts 26). INTERPRETACIÓN: la calidad de las decisiones del router no tiene cobertura automática.
- Proporción por módulo (líneas src / ficheros importados directamente por algún test; análisis por script de imports, aproximado porque core se importa vía índice y tools vía runtime):

| Módulo | Ficheros | Líneas | Con test que lo importa directo |
| --- | --- | --- | --- |
| sources | 16 | 2075 | 16 |
| infra | 5 | 352 | 5 |
| executor | 2 | 840 | 2 |
| app | 28 | 4258 | 25 |
| app/views | 7 | 628 | 6 |
| app/tools | 3 | 444 | 0 directo (hay tests/tools.spec.ts, 491 l., que no leí) |
| core | 15 | 2103 | 1 directo (core-engine.spec.ts 1621 l. y otros lo ejercitan; no leí cómo importan) |
| client | 26 | 4063 | 13 |

Sin test directo en client: eval, execution, overview, prompts, provider, rows, settings, source, thread, ticket, trigger, widgets (~1760 l.), aunque hay specs de panel que podrían cubrirlos por ruta indirecta (no verificado). INTERPRETACIÓN: cobertura funcional probablemente alta en dominio y fuentes; el dato de "sin test" de core y tools es artefacto del método.

## 4. Integración continua y calidad

- DATO: no hay `.github` en el árbol (`ls -a`). GitHub lista 1 workflow: "CodeQL", ruta `dynamic/github-code-scanning/codeql`, activo, creado 2026-10-31 (`gh api .../actions/workflows`): es el escaneo por defecto de GitHub, sin definición en el repo.
- DATO: scripts de package.json: `build` (tsdown), `typecheck` (`tsc --noEmit`), `test` (`vitest run`), `check` = typecheck && test && build (package.json:scripts). Ningún gate automático lo ejecuta (no encontrado en lo leído: no hay CI).
- DATO: tsconfig `strict`, `noUncheckedIndexedAccess`, `verbatimModuleSyntax` (tsconfig.json). Sin ESLint/Prettier/Biome/editorconfig (`git ls-files` no los lista), sin pre-commit/husky, sin cobertura declarada ni umbral (vitest.config.ts:3 no define `coverage`; `coverage/` solo está en .gitignore).
- INTERPRETACIÓN: la calidad depende de disciplina local (`pnpm check`); el estilo de código es coherente pese a no haber formateador, probablemente por generación con agentes (README declara que gran parte de la ingeniería la hicieron agentes).

## 5. Documentación

- DATO: README.md (2.5 KB), SETUP.md (7.8 KB), docs/000-009: filosofía, dominio, autorización, ejecución, prompts, panel, despliegue, arquitectura (617 l.), cookbook (425 l.); comentarios JSDoc abundantes en casi todas las clases.
- Quién manda: el código lo respalda con tests de docs (`tests/docs.spec.ts`, `class-docs.spec.ts`, 68 y 19 l.; no leídos en detalle), lo que apunta a doc vigilada por tests.
- Desfases comprobados (doc/comentario frente a código): (a) src/app/sessions.ts:55-60 dice que el worker no recibe herramientas que inicien o envíen mensajes a otras sesiones, pero `WORK_TOOLS` incluye `send_message`, `list_agents`, `interrupt_agent` (sessions.ts:61-62); (b) comentario "Revalidation rules follow docs/002" correcto en el sentido, pero policy.ts:213-240 sobrepasa el rango citado por la nota previa (213-235): cita imprecisa de una nota, no del repo.
- Coincidencias comprobadas: docs/007-architecture.md:258 ("core importa nada fuera de sí", "sources no importa de app") coincide con el análisis de imports; docs/002-authorization.md:151 (2 reintentos y a la 3.ª unavailable) coincide con src/core/evals.ts:19,61-62; docs/003-execution.md:146 (reinicio) coincide con src/core/assignments.ts:21,83-92.
- Idioma: docs en inglés; textos de UI/descripciones zh+en (src/sources/github.ts:41; cordis.patch.yml:24-25).

## 6. Dependencias

- DATO: runtime: 2 (`proper-lockfile 4.1.2`, `zod 4.4.3`), exactas. Peer: 24 paquetes `@deepseek-ai/*` con `^0.2.1-alpha.2` más `cordis 4.0.5-alpha.1` y `schemastery 3.18.5-alpha.1` (package.json). Dev: ~35 (typescript 6.0.3, vitest 4.1.8, tsdown 0.22.14, react 18.3.1, jsdom 29.1.1...), exactas.
- Gestor: pnpm (`packageManager: pnpm@11.7.0`); lockfile `pnpm-lock.yaml` (171 KB) versionado. Node `^22.19.0 || >=24.0.0`.
- Riesgos: todo el framework (DSH/Cordis) en versión alpha, no incluido ni verificable desde el repo; uso de `node:sqlite` (src/app/store.ts:2), módulo de Node de estabilidad limitada (INTERPRETACIÓN: no verifiqué su estado en Node 22.19); dependencia de CLIs externos `gh` y `lark-cli` (src/infra/environment.ts:99-101); paquete `private: true`, versión 0.1.0.
- No ejecuté auditoría de vulnerabilidades.

## 7. Convenciones y onboarding

- DATO: sin CONTRIBUTING, CODE_OF_CONDUCT, plantillas de issue/PR (community/profile: todos null), sin AGENTS.md/CLAUDE.md.
- Lo que sí hay: convenciones de diseño escritas en docs/007-architecture.md:255-272 ("Dependency rules": interfaces estrechas, un escritor por estado, verbos imperativos para peticiones y pasado para hechos) y docs/008-cookbook.md (cómo añadir un Source; no leído). SETUP.md y docs/006 guían el despliegue. 
- Añadir una pieza nueva: un Source requiere un `SourceKind` + registro + entrada de build (src/sources/github.ts:110-122; tsdown.config.ts:11; cordis.patch.yml:4-12), unos 5-8 ficheros; una tool, 1-3 ficheros (src/app/tools/*, src/app/sessions.ts:61).
- INTERPRETACIÓN: onboarding por lectura de docs, buena para quien las lea; sin guía de estilo formal ni gates para nuevos colaboradores.

## 8. Salud del proyecto vía GitHub (solo GET)

- Repo: creado 2026-10-31, último push 2026-11-03, rama `main`, no archivado, MIT; estrellas 24 156, forks 3 005, suscriptores 184, tamaño 37 KB (`gh api repos/deepseek-ai/dsh-superagent-prototype`).
- Releases: lista vacía; tags: vacío. Cadencia de las últimas 5 releases: n/d (no existen).
- Commits: 1 (8c32a9d, 2026-10-31, "feat: add the dsh-superagent research prototype", autor "DeepSeek Harness Team"); contributors: lista vacía en la API.
- Issues: `has_issues: false`; listados abiertos/cerrados vacíos. PRs: `has_pull_requests: false`; el endpoint `/pulls` devuelve 404. Tiempo de respuesta en muestra de 10 issues/PRs: n/d (no hay).
- Archivos de comunidad: health 37 %; README y LICENSE sí; CONTRIBUTING, CoC, plantillas no.
- INTERPRETACIÓN: es un volcado de referencia publicado, no un proyecto abierto a contribuciones; ninguna señal de cadencia o respuesta puede medirse.

## 9. Factor de bus y deuda

- Factor de bus: n/d con historial git (un solo commit); la firma única es "DeepSeek Harness Team" (INTERPRETACIÓN: 1 equipo, sin comunidad).
- TODO/FIXME/HACK: 1 en todo el repo (`rg 'TODO|FIXME|HACK' src tests scripts prompts docs`): src/index.ts:17 (esperar a que el servicio de settings de DSH relea una entrada reactivada).
- Código heredado/compatibilidad: inputs `reopen` solo históricos (src/executor/dsh.ts:154), marcadores de nombres antiguos `dsh-tag`, `dsh-go`, `dsh-freelancer` (src/infra/environment.ts:16), Evals sin `primary` (src/core/tickets.ts:40), migración desde esquema 10 (src/app/migration.ts:20-25). INTERPRETACIÓN: ya arrastra deuda de compatibilidad con un historial interno previo.
- Código muerto: no busqué de forma sistemática; no encontrado en lo leído.
- Flags experimentales: `eval.mode` off/observe/auto con `observe` por defecto (src/policy.ts:84); `observeIssues/Pulls/Comments/Ci` apagados por defecto (src/sources/github.ts:27-28). No hay flags de compilación.
- Deuda declarada: sin purga de datos (docs/007-architecture.md:484); preguntas abiertas de confianza y aislamiento (docs/007-architecture.md:614-617).

## 10. Riesgos de mantenimiento (ordenados)

1. Dependencia de DSH/Cordis alpha, no incluidos ni verificables (package.json peerDependencies; 24 paquetes alpha). Un cambio de API rompe el plugin sin aviso.
2. Sin CI ni gates: `pnpm check` solo se corre a mano (package.json scripts; sin .github).
3. Sin canal de contribución ni respuesta: issues y PRs desactivados, sin releases (API de GitHub).
4. Ciclo de imports de valores en la capa `app` y ficheros densos (src/executor/dsh.ts:18 ↔ src/app/notices.ts:4; tickets.ts 423 l.): los cambios locales tienen radio amplio.
5. Falta de pruebas con modelo real: la lógica del router/assessor solo se prueba con mocks (tests/fixtures/host.ts).
6. Sin aislamiento de workers: riesgo operativo permanente (src/app/sessions.ts:152, docs/006-deployment.md:19).
7. Crecimiento sin purga de la BD (docs/007-architecture.md:484) y migración limitada a 10→11 (src/app/migration.ts:20-25).
8. Sin linter/formateador y líneas largas: deriva de estilo con más autores (ausencia de configuración).
9. Desfase comentario/código en las tools del worker (src/app/sessions.ts:55-62).

## 11. Cobertura y confianza

- Medido con comandos (alta confianza): conteos de ficheros y líneas, casos de test, TODO, existencia de CI/linters, metadatos de GitHub.
- Leído (confianza media-alta): package.json, tsconfig, tsdown, vitest.config, cordis.patch.yml, ~35 ficheros de src (≈45 % de las líneas no-client; ≈32 % de todo src), tests/fixtures/host.ts (60 l.), extractos de docs 002/003/006/007.
- No leído: casi todos los tests (<2 %), src/client, docs 000/001/004/005/008/009, prompts, scripts/dsh-superagent, migración (cuerpo), feishu*, settings, http (salvo grep).
- Limitaciones: la proporción por módulo usa un análisis de imports con regex (ignora imports multilínea); sin historial git no hay factor de bus ni churn; no ejecuté nada del repo, así que no sé si los tests pasan. Ningún dato de GitHub sobre tiempos de respuesta (n/d).
