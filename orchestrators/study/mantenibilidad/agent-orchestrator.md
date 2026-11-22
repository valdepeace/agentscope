# agent-orchestrator: estudio de MANTENIBILIDAD

Repo: `OrchestratorInc/agent-orchestrator` (clon superficial en `references/agent-orchestrator/`). Rutas relativas a la raíz del repo. DATO = contado o leído; INTERPRETACIÓN = juicio mío. GitHub solo con GET, el 2026-11-21.

## 1. Resumen y veredicto

- DATO: ~1,65 M de líneas de código y tests en Go y TS, 23 workflows de CI, ~14.995 estrellas, releases nightly a diario y estables cada pocos días.
- DATO: proporción tests/código cercana a 1:1 en casi todos los módulos de backend, y CI con `-race`, golangci-lint v2, gofmt, vet y detección de deriva de código generado.
- INTERPRETACIÓN: proyecto vivo y bien vigilado por CI, pero con ficheros gigantes (8382 líneas en `session_manager/manager.go`) y reglas de arquitectura documentadas que el código ya incumple.
- INTERPRETACIÓN: el riesgo principal es el ritmo (987 PRs fusionadas en 30 días, 592 abiertas) frente a un grupo pequeño de mantenedores y a la deuda de migraciones SQLite.
- Veredicto: mantenibilidad media-alta por proceso y tests; media-baja por tamaño de módulos clave y coste de añadir piezas.

## 2. Tamaño y forma

Comandos (bash, desde la raíz): `git ls-files | wc -l` = 7950; para cada categoría `git ls-files 'backend/**/*.go' | grep -v _test.go | xargs cat | wc -l` y equivalentes con `_test.go`, `'*.ts' '*.tsx'` (filtrando `.test.|.spec.`) y `'*.md' '*.mdx'`.

| Categoría | Ficheros | Líneas |
|---|---|---|
| Go fuente (incluye generado: `storage/sqlite/gen` 23475 líneas y `protocol.gen.go` 5365) | 1471 | 401.058 |
| Go tests (`*_test.go`) | 1633 | 470.738 |
| TS/TSX fuente (incluye `schema.ts` generados: 29.200 + 11.363) | 1931 | 419.353 |
| TS/TSX tests (`.test.` 1312 y `.spec.` 72) | 1384 | 325.680 |
| Markdown/MDX | 213 (`docs/` 178 ficheros en total) | 29.913 |
| SQL | 305 (263 migraciones) | n/d |

DATO: Go tests tienen más líneas que Go fuente (1,17x). Aviso: otra medición mía por clasificación de rutas (`/e2e/`, `/test/` como test) da cifras algo distintas; usar las de la tabla.

**Los 10 ficheros más grandes no generados** (`wc -l`):

| Líneas | Fichero | Qué hace |
|---|---|---|
| 8382 | `backend/internal/session_manager/manager.go` | spawn, kill, restore, reconcile, envío de sesiones |
| 7908 | `frontend/src/renderer/components/chat/ChatWorkspace.tsx` | pantalla de chat |
| 6778 | `frontend/src/main/browser-view-host.ts` | navegador embebido (Electron) |
| 5681 | `backend/internal/storage/sqlite/store/conversation_store.go` | store de conversaciones |
| 5101 | `frontend/src/main.ts` | proceso principal de Electron |
| 4826 | `backend/internal/httpd/controllers/dto.go` | DTOs HTTP |
| 4589 | `backend/internal/service/chat/controller.go` | controlador de chat |
| 4542 | `backend/internal/httpd/apispec/specgen/build.go` | generador de OpenAPI |
| 4379 | `frontend/src/renderer/components/chat/ChatTimelineItems.tsx` | línea de tiempo del chat |
| 3848 | `frontend/src/renderer/components/Sidebar.tsx` | barra lateral |

Generados más grandes: `frontend/src/api/schema.ts` 29.200; `packages/cloud-client/src/schema.ts` 11.363; `storage/sqlite/gen/conversations.sql.go` 5849.

Complejidad evidente (DATO, leído):
- `daemon/daemon.go:174-1724`: `Run` es una función de ~1550 líneas.
- `session_manager/manager.go:1110-1840`: `Spawn`, ~730 líneas con ~35 puntos de error etiquetados.
- `lifecycle/manager.go:567-1170`: `ApplyActivitySignal`, ~600 líneas.
- `observe/scm/observer.go:454-843`: `Poll`, ~390 líneas.
- `storage/sqlite/db.go` 2621 líneas con 22 funciones `repair*/prepare*` y un parche de `sqlite_master` por harness (`db.go:2303-2557`).

INTERPRETACIÓN: los módulos centrales son difíciles de revisar por diff; cualquier cambio en spawn/restore pasa por el mismo fichero.

## 3. Pruebas

Proporción líneas test/fuente por módulo (fuente sin generado; cifras de `wc -l` agrupadas por directorio):

| Módulo | Fuente | Tests | Ratio |
|---|---|---|---|
| `lifecycle` | 4288 | 9051 | 2,11 |
| `session_manager` | 20567 | 39667 | 1,93 |
| `storage` | 22919 | 37164 | 1,62 |
| `observe` | 10785 | 16762 | 1,55 |
| `service` | 94707 | 125302 | 1,32 |
| `adapters/chatdriver` | 26509 | 37455 | 1,41 |
| `adapters/agent` | 55429 | 59357 | 1,07 |
| `httpd` | 31054 | 33098 | 1,07 |
| `cli` | 16934 | 21839 | 1,29 |
| `daemon` | 6150 | 5591 | 0,91 |
| `domain` | 9135 | 2450 | 0,27 |
| `ports` | 5875 | 187 | 0,03 (interfaces) |
| `frontend/src/renderer` | 216588 | 202822 | 0,94 |
| `frontend/src/main` | 36555 | 40441 | 1,11 |
| `packages/mobile` | 90268 | 59132 | 0,66 |
| `packages/product-ui` | 6741 | 3773 | 0,56 |
| `frontend/src/docs` | 2126 | 152 | 0,07 |

Sin ningún test (DATO): `frontend/src/main.ts` (5101 líneas), `frontend/src/preload.ts` (1157; hay `preload.test.ts` de 604 en `frontend/src`, así que el ratio real no es 0), `annotate-preload.ts` (1743; hay `annotate-preload.test.ts` 689), `backend/cmd` (1124), `packages/shared` (1403), `adapters/cuadriver` (151), `reqid`, `daemonmeta`. Aviso de método: el agrupador por directorio no empareja `x.ts` con `x.test.ts` en la raíz de `frontend/src`; `main.ts` sí no tiene `main.test.ts` visible.

Tipos de test:
- Unit: la gran mayoría, con fakes (`newFakeStore`, etc.).
- Integración backend: `backend/internal/integration/` (6 ficheros, 1553 líneas): SQLite, observer SCM, kill, workspace.
- Migraciones: decenas de `storage/sqlite/migrate_*_test.go`.
- E2E con daemon real: `backend/e2e/` (16 ficheros, 4380 líneas) arranca el binario `ao daemon` y "habla con un agente real"; protegido por `AO_CHAT_E2E=1` porque cuesta llamadas reales a modelos (`backend/e2e/harness_test.go:1-17`). `git grep AO_CHAT_E2E -- .github` no devuelve nada: **no corre en CI** (DATO).
- E2E frontend: `frontend/e2e/` (90 ficheros, 12.306 líneas, Playwright) y `test/e2e-pod/` (Electron real en pod Daytona, `boot-real.sh`).
- Conformidad con TUIs publicadas: `tau-conformance.yml`, `open-interpreter-conformance.yml`.
- Modelo real o simulado: unitarios simulados (harness `fake`, referenciado en 9 ficheros); modelo real solo en el e2e opt-in y en los conformance.
- Skips: 455 `t.Skip*` en Go y 10 `skip/todo` en TS (DATO). Etiquetas de build en tests: 59 `!windows`, 22 `windows`, 2 `e2e`.

## 4. Integración continua y calidad

Workflows (23 en `.github/workflows/`):

| Workflow | Qué comprueba | Disparador |
|---|---|---|
| `ci.yml` | elige carriles con `scripts/ci-lanes.mjs` y llama a go/cli-e2e/frontend/mobile/config; `ci-ok` agrega y exige success o skipped | push a main, PR |
| `go.yml` | `backend-core` (shards) y `storage-race`, `windows-workspace`, `lint` (gofmt, build, vet, golangci-lint v2), `codegen-drift` (sqlc, OpenAPI, `schema.ts`) | reutilizable |
| `frontend.yml` | typecheck, vitest en shards, docs, boundary, postcondition, cloud, product-ui, renderer-smoke, helpers mac | reutilizable |
| `mobile.yml` | `npm run typecheck`, `npm test` | reutilizable |
| `cli-e2e.yml` | `go test -tags e2e ./internal/cli/...` y smoke en Docker | reutilizable |
| `gitleaks.yml` | secretos | push/PR |
| `public-boundary.yml` | `scripts/check-public-boundary.sh` (frontera con el código privado) | push/PR |
| `cloud-contract-compat.yml` | oasdiff del contrato cloud | PR |
| `version-floor-guard.yml`, `release-latest-guard.yml` | protegen el "suelo" de versión de escritorio y el último release | PR / cron |
| `pricing-catalog-validate.yml` | catálogo de precios | PR |
| `e2e-gate-tests.yml`, `optional-private-submodule.yml` | pruebas de scripts y del submódulo opcional | PR |
| `tau-conformance.yml`, `open-interpreter-conformance.yml`, `harness-release-feeds.yml` | TUIs reales y feeds de releases de harness | PR/cron |
| `build-artifacts.yml`, `mac-update-e2e.yml`, `harness-proof-build.yml`, `deploy-docs.yml` | empaquetado, actualización macOS, docs | manual/main |
| `contributor-project.yml`, `pr-review-leaderboard.yml`, `ci-retarget.yml` | gestión de contribuciones | PR/cron |

- Gate: `ci-ok` (`.github/workflows/ci.yml:75-91`). Windows completo solo si `full == 'true'` (`ci.yml:41-45`).
- Go: `go test -race` en los jobs (`go.yml`); lint `backend/.golangci.yml` con ~26 linters (errcheck, staticcheck, unused, gosec, dupl, revive, gocritic, forbidigo...), `max-issues-per-linter: 0`. Lint local: `package.json:5` (fija `golangci-lint@v2.14.0`).
- TS: `strict: true`, `noUnusedLocals/Parameters`, `noImplicitReturns` en `frontend/tsconfig.json`; typecheck con `tsgo` (`@typescript/native-preview`, versión dev, `frontend/package.json`).
- Formateo/lint de TS: **no hay** ESLint, Prettier, Biome ni EditorConfig versionados (`git ls-files | grep -iE '(eslint|prettier|biome|oxlint|editorconfig)'` vacío). Pre-commit/husky/lefthook: ninguno encontrado en lo leído.
- Cobertura declarada: ninguna; `grep -i coverage` en workflows solo da comentarios. No hay umbral de cobertura (DATO).
- INTERPRETACIÓN: la calidad se apoya en tests, lint de Go y typecheck; el lado TS depende de typecheck y revisión.

## 5. Documentación

- DATO: `README.md` 21 KB, `AGENTS.md` 167 líneas, `CLAUDE.md` 41, `CONTEXT.md` 48, `DESIGN.md` 461, `docs/` 178 ficheros (41 capturas, 26 en `superpowers/`, 16 `research/`, 7 ADR), `docs/architecture.md` ~1400 líneas, `docs/backend-code-structure.md` 978, `docs/STATUS.md`, `docs/scm-observer.md`. Manual público en `frontend/src/docs/content/` (MDX).
- Quién manda: `docs/documentation-map.md:12-20` declara que gana el contrato generado (openapi, sqlc, CI drift) sobre la prosa, y que la prosa "puede ir por detrás".
- Desfases comprobados (doc vs código):
  1. Doc: "Adapters are leaves... no core imports" (`docs/backend-code-structure.md:73`, `docs/architecture.md:1364`). Código: `backend/internal/adapters/scm/multi/provider.go:14` importa `observe/scm`; `adapters/telemetry/localsqlite.go:13` importa `storage/sqlite/store`.
  2. Doc: "Storage forbidden to import HTTP" (`backend-code-structure.md:108`) se cumple; pero `storage/sqlite/store/shell_terminal_store.go:10` importa `service/shellterm`, que sí es capa superior según `:103-107`.
  3. Comentario de código: añadir un harness es "a single edit to Constructors" (`backend/internal/adapters/agent/registry/registry.go:1-5,107-109`). Código: 57 ficheros mencionan `mastra-code`, incluyendo `domain/harness.go:46`, `storage/sqlite/db.go:2330`, `httpd/controllers/dto.go:696`.
  4. Coincide (sin desfase): cadencia del observer 30 s (`docs/scm-observer.md:99` vs `observe/scm/observer.go:32`).
- INTERPRETACIÓN: documentación abundante y con jerarquía de verdad explícita; los desfases están en reglas de arquitectura, no en cifras operativas.

## 6. Dependencias

- Gestor/lockfiles (DATO): Go modules con `backend/go.sum` y `go.work(.sum)`; npm con 8 `package-lock.json` (raíz, `frontend`, `frontend/acp-runtime`, `frontend/src/docs`, `packages/{cloud-client,mobile,product-ui}`, `scripts`); `flake.lock` (Nix). Sin workspaces npm unificados.
- Go: 31 directas + ~35 indirectas (`backend/go.mod`); `go 1.27.4` (`go.mod:3`). Pesadas o de riesgo: `modernc.org/sqlite`, `github.com/pressly/goose/v3`, `github.com/getsentry/sentry-go`, `github.com/coder/acp-go-sdk`, `github.com/aymanbagabas/go-pty`, `github.com/unreallabsai/unreal-agent v0.1.1` (dependencia de terceros para un harness concreto), `github.com/unixshells/vt-go v0.2.0`.
- Frontend: 73 deps + 34 dev + 2 opcionales (`frontend/package.json`); fijadas exactas 18 de 107, caret 88. `@xterm/*` en versiones **beta** fijas, `@pierre/trees@1.0.0-beta.6`, `@typescript/native-preview@7.0.0-dev...`, `electron@^44.7.0`, `better-sqlite3` (nativa), `posthog-js`, `@sentry/electron`.
- Móvil: 54 deps + 12 dev; 34 con tilde, 17 con caret, 12 exactas.
- Herramientas fijadas por `go run`: sqlc v1.31.1 y golangci-lint v2.14.0 (`package.json:5,9`); `oasdiff v1.34.1` en CI.
- INTERPRETACIÓN: superficie de dependencias grande (Electron + Expo + Go); betas fijadas en terminal embebido son riesgo de regresión silenciosa.

## 7. Convenciones y onboarding

- `CONTRIBUTING.md` (112+ líneas): empezar por bugs etiquetados, comentar para reclamar, features con aprobación previa, `Fixes #N`, llamada semanal y Discord; apartados "CI lanes" (`:77`) y "Native CI coverage" (`:94`); sección Code of Conduct (`:112`) pero GitHub no detecta `code_of_conduct` (`community/profile`: null).
- `AGENTS.md` es la guía operativa y `CLAUDE.md` la remite; reglas duras como no editar migraciones fusionadas, no editar `gen/`, crear migración con `npm run migration:create` y registrar su ID en `shippedMigrations` (`AGENTS.md:118-120`).
- Skills de agente en `.agents/skills/` y `.claude/skills/` (p. ej. `bug-triage`, `ao-desktop-dev`). Plantillas: `bug_report.yml`, `feature_request.yml`, `pull_request_template.md` (exige conteos de líneas y Antes/Después).
- `CODEOWNERS` mínimo: solo protege `desktop-version-floor.json` (`.github/CODEOWNERS:1-3`).
- Cómo se añade una pieza: hay docs por harness en `docs/harnesses/` (11 ficheros) pero no una lista única de ficheros a tocar; en la práctica son ~35 (ver arquitectura §6). No encontrado en lo leído un checklist ejecutable de "alta de harness".
- Zona sensible: DESIGN.md manda para decisiones visuales (`CLAUDE.md:25-31`).

## 8. Salud del proyecto vía GitHub (GET)

- Repo: creado 2026-02-13, último push 2026-11-21, 14.995 estrellas, 2066 forks, Apache-2.0, no archivado.
- Últimas 5 releases estables: v0.15.3 (2026-11-16), v0.15.2 (2026-11-15), v0.15.1 (2026-11-14), v0.15.0 (2026-11-13), v0.14.10 (2026-11-06). Previas: v0.14.9 (11-04), v0.14.8 (10-30), v0.14.7 (10-28). Además nightlies (prerelease) casi diarias: `v0.15.4-nightly.202611212010` el 2026-11-21.
- Issues (search API, tipo issue): 468 abiertas, 2193 cerradas; en los últimos 30 días 336 abiertas y 367 cerradas. Abiertas con >90 días: 29.
- PRs: 592 abiertas, 3303 fusionadas, 2219 cerradas sin fusionar; 987 fusionadas en 30 días; 146 abiertas con >30 días.
- Respuesta, muestra de 10 issues cerradas recientes (#8801, #8769, #8762, #8743, #8700, #8699, #8665, #8656, #8645, #8504): tiempo hasta cierre 1,4 h a 86,6 h, mediana ~21 h; 6 de 10 sin ningún comentario. En 4 issues abiertas de externos (#8811, #8781...), primera respuesta de la cuenta `OrcInc` a los 18 y 77 minutos (#8811, #8781). Muestra pequeña y sesgada a casos cerrados.
- Muestra de 10 PRs cerradas recientes (#8827...#8783): 9 fusionadas, cierre entre 0,1 h y 17,6 h; 0 h hasta primer comentario en todas (casi seguro bot; no distinguí bot de humano), 7 de 10 con 1-3 reviews. #8783 cerrada sin fusionar.
- Ficheros de comunidad (`community/profile`, health 62 %): README, LICENSE, CONTRIBUTING, plantilla de PR presentes; `code_of_conduct` null; `issue_template` null en la API aunque `.github/ISSUE_TEMPLATE/` tiene 2 formularios y `config.yml` (desfase de detección, no del repo).

## 9. Factor de bus y deuda

- Contribuyentes (API, total histórico): `illegalcall` 490, `ronishrohan` 408, `AgentWrapper` 341, `harshitsinghbhandari` 245, `nikhilachale` 182, `Prasad-D-Ware` 182, ... (15 con >=83). Últimos 100 commits en main: `ronishrohan` 24, `Prasad-D-Ware` 19, `illegalcall` 17 (60 % entre tres).
- Marcas: `git grep -wE 'TODO|FIXME|HACK|XXX'` en `.go/.ts/.tsx/.mjs/.sh/.py` da 23 aciertos, 0 FIXME/HACK/XXX; solo **2** son deuda real: `backend/internal/adapters/agent/crush/activity.go:51` y `backend/internal/service/session/claim_pr.go:160` ("implement workspace branch checkout"). El resto son cadenas de test o texto de UI.
- Código sin llamador: `(*Manager).launchRuntimeEnv` (`backend/internal/session_manager/manager.go:7304`) solo se invoca desde un test (`provision_test.go:138`). Método: recuento de referencias de ~3500 funciones de `session_manager`, `lifecycle`, `observe/scm` y `service/session`; los nombres con una sola referencia en fuente y no-test fueron 2, solo ese era función de producción. El linter `unused` de CI explica la baja cifra.
- `legacy` aparece en 745 líneas de Go no test (`git grep -nIi legacy`); hay paquetes `legacyimport` y `devimport` y rutas de compatibilidad (`legacyScratch`, `legacyInterruptedClientSetup`).
- Flags experimentales: 92 menciones de "experimental" en código no test; `docs/STATUS.md:98-132` marca como experimentales el reviewer contenido y el adaptador `fx`. Chat hibernation arranca deshabilitada (`docs/architecture.md:80`).

## 10. Riesgos de mantenimiento ordenados

1. Ficheros monolíticos en el camino crítico: `manager.go` 8382, `daemon.Run` ~1550, `ApplyActivitySignal` ~600. Evidencia: §2.
2. Esquema SQLite: 263 migraciones, 22 reparaciones de historial y reescritura manual del CHECK por cada harness (`db.go:2303-2557`). Cada harness nuevo añade una rama. Contradice `AGENTS.md:118`.
3. Coste de alta de harness (~35 ficheros de producción) con un catálogo de 44 y creciendo: el comentario `registry.go:1-5` da una falsa expectativa.
4. Ritmo de cambio (987 PRs fusionadas/30 días, 592 abiertas, 146 con >30 días) con tres personas aportando el 60 % de los últimos commits.
5. Reglas de capas documentadas incumplidas (§5.1-2), lo que erosiona la guía para agentes y humanos.
6. Sin lint/formato de TS ni cobertura medida, con ~745 k líneas de TS (fuente+tests).
7. Dependencias beta y dev fijadas en terminal y tipado (`@xterm/*`, `@typescript/native-preview`), y `go 1.27.4` en go.mod: exige toolchains recientes.
8. E2E con agente real sin CI (`AO_CHAT_E2E`), así que regresiones de integración real dependen de ejecución manual o de los conformance de Tau/Open Interpreter.
9. Submódulo privado `private/ao-cloud` y `public-boundary.yml`: parte del producto no es auditable desde el repo público.

## 11. Cobertura y confianza

- Cobertura por conteo: todo el repo (7950 ficheros) con `git ls-files`, `wc -l` y `git grep`; las cifras de tamaño, TODO, dependencias y workflows son completas, con la salvedad del agrupador por directorio indicado en §3.
- Lectura de código: ~45 ficheros abiertos (~0,6 %), ~2500 líneas (<0,2 %); las citas de código son de rangos efectivamente abiertos (ver arquitectura §9). Workflows: leídos `ci.yml` completo y encabezados/pasos `run:` de los 23; el detalle de `frontend.yml` (457 líneas) y `go.yml` (231) es parcial.
- GitHub: contadores por search API; muestras de 10 issues y 10 PRs, pequeñas y sesgadas; no distinguí bots de humanos en los primeros comentarios. La paginación de releases se leyó con `--jq` sobre una página de 100 (alcanza hasta 2026-10-09).
- Confianza: alta en tamaños, CI, dependencias y salud; media en pruebas (ratios por agrupación de rutas); baja en deuda de código muerto (heurística de referencias limitada a 4 directorios).
- Pendiente: tests más grandes por fichero, cobertura real por ejecución (no permitida), contenido de `frontend.yml` y `mobile`.
