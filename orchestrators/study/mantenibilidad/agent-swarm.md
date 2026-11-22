# agent-swarm: estudio de mantenibilidad

Repo `desplega-ai/agent-swarm` v2.14.2, clon superficial en `references/agent-swarm/` (sin historial git). Solo lectura. DATO = cifra o texto citado; INTERPRETACION = juicio mío. Rutas relativas a la raíz del repo; metadatos de GitHub vía `gh api` (solo GET) a fecha 2026-11-21.

## 1. Resumen y veredicto

- DATO: proyecto muy activo y casi automatizado: 3.016 PRs fusionadas en total, 1.050 en los últimos 30 días, 12 releases en 12 días (v2.10.0 a v2.14.2), y los dos mayores contribuidores son bots (`desplega-bot` 2.184 commits, autor humano principal 1.158, `Worker Agent` 290).
- DATO: disciplina de calidad inusualmente alta en CI: 26 jobs en `merge-gate.yml`, 8 shards de tests, e2e de contrato estricto en 4 shards, Biome + oxlint + `tsc` estricto, hooks pre-push, gitleaks, comprobaciones de deriva de generadores.
- DATO: tests 1,2 veces el código fuente en líneas (435.404 vs 362.665 en `src/`), pero los e2e usan LLM simulado, no modelo real.
- INTERPRETACION: la mantenibilidad depende de que los gates sigan siendo obligatorios y de un equipo pequeño asistido por agentes; el riesgo está en el tamaño del núcleo (`db.ts` 14.753 líneas, `runner.ts` 8.334) y en la cantidad de ficheros que hay que tocar por cambio.
- INTERPRETACION: veredicto: mantenible mientras la maquinaria de CI siga viva; frágil para un recién llegado por volumen (7.498 ficheros versionados, 261 migraciones, 580 documentos de "thoughts").

## 2. Tamaño y forma

Comandos exactos (desde la raíz del repo, bash):
- Ficheros versionados: `git ls-files | wc -l` -> 7.498.
- Fuente `src/` sin tests: `git ls-files 'src/*.ts' 'src/*.tsx' | grep -v '^src/tests/' | grep -v '\.test\.ts$' | xargs cat | wc -l` -> 362.665 líneas; los mismos filtros con `wc -l` de la lista -> 1.252 ficheros `.ts`.
- Tests: `git ls-files | grep -E '\.(test|spec)\.tsx?$' | wc -l` -> 1.632 en todo el repo (src 1.150: 1.139 en `src/tests` + 11 colocados; apps/ui 186; apps/evals 84; packages 7). Líneas de los 1.150 tests de `src`: 435.404 (script propio que lee cada fichero).
- Docs: `git ls-files <dir> | grep -E '\.(md|mdx)$' | xargs cat | wc -l` -> docs-site 247 ficheros / 33.949 líneas; runbooks 27 / 7.892; thoughts 576 / 188.367; specs 6 / 1.735. Aparte `CHANGELOG.md` 369 KB, `MCP.md` 188 KB, `DEPLOYMENT.md` 51 KB, `LOCAL_TESTING.md` 43,7 KB (`ls -la`).
- Por tipo (`git ls-files | sed -n 's/.*\.//p' | sort | uniq -c`): 3.857 `.ts`, 976 `.tsx`, 915 `.md`, 399 `.json`, 267 `.sql`, 245 `.mdx`.
- Por directorio raíz (`git ls-files | awk -F/ '{print $1}'`): src 2.885, apps 2.019, thoughts 872, scripts 413, templates 353, docs-site 344, packages 105.

**10 ficheros más grandes** (no-test; `git ls-files ... | xargs wc -l | sort -rn`), con función por nombre y las lecturas que hice:
| # | Fichero | Líneas | Qué hace |
|---|---|---|---|
| 1 | `src/be/db.ts` | 14.753 | fachada y lógica de acceso a datos; 455 líneas `export` (leído parcialmente) |
| 2 | `src/commands/runner.ts` | 8.334 | runner de worker/lead; `runAgent` ~1.960 líneas, `spawnProviderProcess` ~1.080 (leído parcialmente) |
| 3 | `apps/ui/src/api/types.ts` | 4.903 | tipos de la API para el dashboard (no leído) |
| 4 | `src/types.ts` | 4.849 | schemas zod del dominio (`src/types.ts:519` ProviderNameSchema, etc.) |
| 5 | `apps/ui/src/api/client.ts` | 4.814 | cliente HTTP del dashboard (no leído) |
| 6 | `apps/ui/src/lib/json-render/components.tsx` | 4.431 | componentes de render JSON (por nombre) |
| 7 | `apps/ui/src/components/shared/session-log-viewer.tsx` | 3.341 | visor de logs de sesión (por nombre) |
| 8 | `apps/ui/src/pages/connections/page.tsx` | 3.290 | página de conexiones (por nombre) |
| 9 | `src/telemetry-contract/types.gen.ts` | 3.120 | generado (excluido de Biome, `biome.json` includes) |
| 10 | `src/slack/render-v2.ts` | 2.810 | render de mensajes Slack (por nombre) |
Siguen `src/http/tasks.ts` 2.631, `src/providers/codex-adapter.ts` 2.612, `src/http/script-connections.ts` 2.599, `src/apps/page-validator.ts` 2.486, `src/providers/pi-mono-adapter.ts` 2.366, `src/heartbeat/heartbeat.ts` 2.229.

**Complejidad evidente (DATO):** `runAgent` desde `src/commands/runner.ts:6374` hasta el final del fichero; `spawnProviderProcess` `runner.ts:4611-5690`; `src/heartbeat/heartbeat.ts` mezcla clasificación de fallos, reboot sweep y checklist del lead en un fichero de 2.229 líneas; `src/be/db-client.ts` (807 líneas) implementa lock FIFO, SAVEPOINT y reintentos BUSY a mano (`db-client.ts:198-236, 509-548`).
**INTERPRETACION:** los dos primeros ficheros concentran riesgo de conflictos de merge y de revisión; con 35 PRs/día fusionadas (1.050 en 30 días) eso se agrava.

## 3. Pruebas

**Tipos (DATO).**
- Unit/integración con `bun test`: 1.150 ficheros en `src`; 259 usan `Bun.serve`/`listenOnFreePort` y 267 llaman `fetch(` (regex sobre el contenido, aproximado), es decir, muchos arrancan servidores reales en proceso. Puertos libres obligatorios (`CLAUDE.md:66`). Sin reintento global; los inestables lo piden explícitamente (`bunfig.toml`, comentario).
- Política: los tests unitarios nuevos solo se aceptan en las rutas de `scripts/unit-test-allowlist.txt` (worker, tooling, reglas, migraciones, dashboard); lo demás se prueba por e2e (`CLAUDE.md:63`).
- E2E de caja negra: 198 escenarios en `scripts/e2e/scenarios/` (`ls | wc -l`). Arranca la API real con DB nueva, un Slack simulado y un **host de inferencia simulado**; "No Docker, no model" (`scripts/e2e/README.md:3-5`); el agente es un simulador `agent-sim.ts` que hace poll y cierra tareas por HTTP; `mocks/llm.ts` imita OpenAI (README, tabla Layout). Modo `--strict`: falla si una ruta, tool MCP, tipo de nodo de workflow o comando CLI no tiene escenario ni exclusión razonada (`README.md:8-11`). Reintenta fallos una vez en SUT nuevo y distingue `flaky` de `fail` (README, `rerun.ts`).
- UI E2E con Playwright: `packages/ui-e2e` (66 ficheros) y workflows `ui-e2e*.yml`.
- Evals de agentes: `apps/evals` (358 ficheros; no leí si llaman a modelos reales, n/d). `CLAUDE.md:210` dice que el swarm desplegado propone evals pero no corre E2B.
- Dependencia de modelo real: ninguna en tests unit/e2e leídos; 81 tests mencionan `spawn`/`Bun.spawn` y 14 nombran `mockProvider/fakeAdapter` (regex aproximado). Probar un harness real queda como procedimiento manual de QA (`runbooks/harness-providers.md:956-960`).

**Proporción por módulo (DATO, cota inferior).** Número de ficheros de test de `src` que importan el módulo (imports relativos; el alias `@/` no está resuelto, 257 ficheros fuente lo usan): be 722, http 270, utils 236, tools 134, providers 111, commands 101, workflows 95, slack 75, tasks 74, scripts-runtime 45, apps 38, prompts 34, extensions 33, integrations 29, heartbeat 26, scheduler 18, realtime 17, hooks 16, github 15, oauth 15, runtime-providers 13, fs 12, jira 12, linear 8, bitbucket 6, azure-devops 5, gitlab 5, agentmail 5, pages 5, comb 3, ranking 3, x402 3, rbac 2, memory 2, mcp-client 2, vcs 2, tracker 1.
**Sin ningún test que los importe (según el escaneo):** `src/x` (1 fichero, 304 líneas), `src/metrics` (29 líneas), `src/event-waits` (165 líneas). INTERPRETACION: pueden estar cubiertos por e2e o por import con alias; no lo verifiqué. `rbac` (1.112 líneas, 2 tests) y `memory` (121 líneas, 2 tests) tienen poca cobertura directa, aunque `CLAUDE.md:205` exige correr "los cuatro ficheros de test de memoria".
**Relación líneas tests/código (DATO):** 435.404 / 362.665 = 1,20 en `src`.
**Cobertura declarada:** existe un workflow nocturno que ejecuta `bun scripts/run-root-coverage.ts`, sube `lcov` y guarda una línea base en la API de desplega; se lanza por `workflow_dispatch` desde un planificador externo (`.github/workflows/nightly-coverage.yml:3-5, 24-30, 46-67`). No vi umbral mínimo de cobertura en ese fichero (n/d si existe en otro). `bunfig.coverage.toml` activa `coverage=true`.
**Tests no tipados:** `tsconfig.json` excluye `src/tests`, `scripts`, `apps` y `packages` del `tsc` raíz; el e2e tiene su propio `e2e:tsc` (`package.json` scripts). No encontré en lo leído un typecheck de `src/tests`.

## 4. Integración continua y calidad

**Workflows en `.github/workflows/` (17 ficheros, 5.516 líneas):** `merge-gate.yml` (1.805), `docker-and-deploy.yml` (1.229), `ci.yml` (543, no leído), `nightly-e2e.yml` (300), `ui-e2e-report.yml` (292), `slack-visuals-publish.yml` (227), `deploy.yml`, `pr-body*.yml`, `helm-publish.yml`, `migration-conflict-check.yml`, `nightly-coverage.yml`, `gitleaks-cache.yml`, `publish-mcp-registry.yml`, `slack-visuals.yml`, `ui-e2e*.yml`.
**Jobs de `merge-gate.yml` (nombre: líneas):** Detect Changes (21), Lint and Type Check (239), Restore test timings (681), E2E Contract en 4 shards (715) y agregado (781), E2E Tracker Report, E2E Coverage Comment, Run Tests en 8 shards (969), Save test timings, Pi-Skills Freshness (1192), Seeded Skills (1223), Script SDK Types Freshness (1264), OpenAPI Spec Freshness (1288), Vendored OpenAPI (1332), Docker Build Test (1352), Dashboard Image, UI E2E, UI Lint and Type Check (1543), Validate Workflow Templates, Operator Skill Check, Migration Conflict Check (1709), Secret Scan (1731) y `Merge Gate` (1781) como agregador.
**Gates previos al PR (`CLAUDE.md:73-88`):** `check:local`, `lint`, `lint:slop`, `tsc:check:tsc` (tsc 5 como autoridad), tests con `--parallel=4`, `e2e`, `e2e:ui`, `check:bun-version`, `check-db-boundary`, `check-test-spawn-sync`, `check-audit-columns`, `check:dep-graph`. Además `check:rbac-coverage`, `check:openapi-response-coverage`, `check-async-db-seam`, `check-floating-promises`, `check-api-key-boundary`, `check-agents-md-size` y comprobación del cuerpo de PR (plantilla con Why/Risk/Urgency; `CLAUDE.md:16-23`).
**Linters y formato (DATO):** Biome con `recommended` y `noExplicitAny`, `noUnusedImports`, `noFloatingPromises` como error (`biome.json`); formateo 100 columnas; oxlint solo con 6 reglas "anti-slop" propias (`oxlint.config.ts`); `konsistent` (`lint:structure`); dependency-cruiser con 3 reglas (`.dependency-cruiser.cjs`).
**Tipado (DATO):** `strict`, `noUncheckedIndexedAccess`, `noImplicitOverride` (`tsconfig.json`); `noUnusedLocals/Parameters` en `false`.
**Pre-commit/push:** `prek.toml` define hooks de pre-push filtrados por rutas (dedupe del lockfile, Biome, oxlint, typecheck, tests afectados); `PRE_PUSH_FULL=1` para la suite completa (`CLAUDE.md:91`). No hay pre-commit propiamente.
**Seguridad en CI:** Secret Scan (gitleaks, `.gitleaks.toml`), acciones fijadas por SHA (`nightly-coverage.yml:18, 22`), Dependabot semanal con "cooldown" de 7 días y sin majors automáticos (`.github/dependabot.yml`).
**Señal de higiene a revisar (DATO):** la API de GitHub lista workflows activos `Bisect shard-1 SIGTERM (temporary)` y `tmp codex chain probe`, y otros (`Publish SDK`, `Refresh star history chart`, `Vercel UI Install`) que no existen en `.github/workflows/` del clon. INTERPRETACION: pueden venir de otras ramas; indica que los workflows temporales no siempre se retiran.

## 5. Documentación

**Qué hay (DATO):** `README.md` (12 KB), `CLAUDE.md` (23.724 bytes; presupuesto 24.000 comprobado por `check:agents-md-size`, `CLAUDE.md:27`), `AGENTS.md` de 9 bytes que apunta a `CLAUDE.md`, `CONTRIBUTING.md`, `DESIGN.md`, `DEPLOYMENT.md`, `LOCAL_TESTING.md`, `MCP.md`, `BUSINESS_USE.md`, 27 runbooks, 247 ficheros en docs-site (Fumadocs), 576 en `thoughts/` (notas de trabajo), 6 specs.
**Quién manda:** los runbooks se declaran vinculantes con el código: el de heartbeat "solo contiene el comportamiento actual" y hay que actualizarlo en el mismo PR (`CLAUDE.md:208`); `CLAUDE.md` es un índice de condiciones `<important if=...>` que apunta a runbooks. Hay generadores con `--check` (OpenAPI, telemetría, MCP, skills, docs-index) que hacen fallar CI si el doc derivado no está al día (scripts `check:*` de `package.json`).
**Desfases comprobados:**
1. `CLAUDE.md:9` lista como worker-side `commands, hooks, providers, prompts, cli.tsx, claude.ts`; `scripts/check-db-boundary.sh` (cabecera y `WORKER_PATHS`) vigila además `runtime-providers`, `scripts-runtime` y `utils`, igual que `.dependency-cruiser.cjs:14-15`. El doc está atrasado respecto al guardia.
2. `src/heartbeat/heartbeat.ts:129-135` afirma que todos los umbrales se leen dinámicamente "a getter, not a module-load const", pero `heartbeat.ts:176`, `179`, `189` y `250` son constantes de módulo que leen `process.env` al cargar.
3. `CLAUDE.md:149` dice que el runner "silenciosamente se salta" una ordinal de migración duplicada; el código compara checksum y avisa por consola si difiere (`src/be/migrations/runner.ts:280-288`) y, si la versión no estaba aplicada, intentaría aplicarla y chocaría con la clave primaria (razonamiento mío sobre `runner.ts:296-303`; no probado).
4. Coherentes (sin desfase): runbook de heartbeat vs código en intervalo 90 s, umbrales 5/15/30 min, reconciliación 30 s, gracia 45 s y reintento 300 s (`runbooks/heartbeat-crash-recovery.md:15, 31, 35, 37` vs `heartbeat.ts:165-179`, `runner.ts:5680, 5687, 1879`).
**INTERPRETACION:** la documentación está pensada para agentes (el `CLAUDE.md` es denso y condicional) más que para humanos nuevos; `thoughts/` (188 k líneas) es volumen de contexto, no documentación mantenida.

## 6. Dependencias

- Gestor: Bun (`packageManager: bun@1.4.4`, `engines.bun >= 1.3.12`), workspaces `apps/ui`, `apps/templates-ui`, `apps/evals`, `packages/*` (`package.json`). Lockfile `bun.lock` (571 KB) versionado; también `requirements.lock` y `pyproject.toml` en raíz.
- Raíz: 62 `dependencies` (27 con versión exacta, 34 con `^`, 1 por URL) y 17 `devDependencies` (6 exactas, 8 con `^`, resto `workspace:*`/`latest`); `@types/bun` en `latest`. `overrides` para react, lodash-es y socket-mode.
- `bunfig.toml` fija `linker = "hoisted"` y deja dependencias fantasma sin declarar, con comentario que lo reconoce como deuda a corregir (`bunfig.toml`, comentario).
- Pesadas o con riesgo (INTERPRETACION sobre DATO): SDKs de agentes fijados (`@anthropic-ai/claude-agent-sdk 0.3.330`, `@openai/codex 0.171.0`, `@cursor/sdk`, `@earendil-works/pi-*` con `^1.6.0`); `xlsx` se instala desde un tarball en `cdn.sheetjs.com` (fuera del registro npm); stack cripto/pagos (`viem`, `@x402/*`, `@openfort/openfort-node`); `@ironcalc/wasm`, `quickjs` (sandbox de scripts), `e2b`, `@boxd-sh/sdk`, `@aws-sdk/client-bedrock`; `@chat-adapter/*` para 5 canales. Muchas piezas de terceros recientes o de nicho.
- Política de actualización (DATO): Dependabot semanal, grupo minor-patch, majors a mano y 7 días de espera; los pines de los harness los sube un job aparte (`.github/dependabot.yml`, comentario).
- Licencia propia: MIT (`LICENSE`; GitHub la detecta como MIT).

## 7. Convenciones y onboarding

- `CONTRIBUTING.md` (8,6 KB): requisitos, instalación, ejecución MCP/CLI/Docker, portless, calidad de código (índice en `CONTRIBUTING.md:1-146`); `CODE_OF_CONDUCT.md`; plantilla de PR obligatoria con comprobación automática (`CLAUDE.md:16-23`); formularios de issue `bug_report.yml`, `feature_request.yml`, `question.yml`, `community-template.yml` en `.github/ISSUE_TEMPLATE/`.
- `CLAUDE.md` actúa de guía de estilo: reglas de rutas (`route()`), tools (`toolOk/toolErr`), migraciones, secretos (`scrubSecrets`), prompts por registro (`CLAUDE.md:48-212`). `apps/ui/CLAUDE.md` aparte para el dashboard (`CLAUDE.md:29`).
- Cómo se añade una pieza: proveedor, ruta, herramienta y evento de extensión tienen checklist escrita y tests que fallan si se olvida un punto (`runbooks/harness-providers.md:903-915`, `CLAUDE.md:115-140, 200`). Coste: un proveedor nuevo toca 12 o más ficheros; una ruta nueva, 5 a 7.
- INTERPRETACION: onboarding bueno para un agente que sigue `CLAUDE.md`; pesado para un humano (muchos generadores, `check:*`, hooks, e2e estricto, requisito de Node 22 para `e2e:ui`).

## 8. Salud del proyecto vía GitHub (solo GET)

- Repo: creado 2025-12-19, último push 2026-11-21T18:22Z, no archivado, MIT, 1.097 estrellas, 185 forks (`gh api repos/desplega-ai/agent-swarm`).
- **Últimas 5 releases:** v2.14.2 (2026-11-21), v2.14.1 (2026-11-20), v2.14.0 (2026-11-19), v2.13.5 (2026-11-18), v2.13.4 (2026-11-17). Todas a las ~03:00 UTC; 12 releases entre el 10 y el 21 de nov. (una al día). Fusionar el bump de versión en `main` publica Docker, npm, E2B y la release (`CLAUDE.md:173`).
- **Issues:** 21 abiertas, 242 cerradas (búsqueda GitHub); 107 cerradas y 101 creadas en los últimos 30 días.
- **PRs:** 16 abiertas (la más antigua #2846, 2026-11-07; 11 de ellas de 2026-11-16 o posteriores), 3.016 fusionadas, 231 cerradas sin fusionar; 1.050 fusionadas en 30 días.
- **Tiempo de respuesta, muestra (no 10 issues: la API solo devolvió 6 issues entre las últimas 100 entradas, el resto son PRs):** #3530 y #3528 (abiertas por un usuario externo): primer comentario del bot ~6 h después, cerradas a las ~12,7 h; #3457, #3456, #3455 (creadas por el bot): cerradas en 2,5 h, 2,5 h y ~10 h; #3509 (bot externo) abierta. 
- **Muestra de PRs externas cerradas recientes (8):** #3540 y #3539 (autor 4) fusionadas en ~2,4 h; #3514 (Seyamalam) en 16 min; #3502 (Seyamalam) en 3,6 h; #3520 y #3510 cerradas sin fusionar tras ~2,5 y ~4 h; dependabot #3501 fusionada en ~8 h; #3481 cerrada sin fusionar. INTERPRETACION: respuesta de horas, muy rápida, probablemente por el propio enjambre.
- **Comunidad (`community/profile`):** health 87 %; README, LICENSE, CONTRIBUTING, CODE_OF_CONDUCT y plantilla de PR presentes; `issue_template` figura `null` en la API aunque `.github/ISSUE_TEMPLATE/` contiene 4 formularios y un `config.yml` (DATO contradictorio, no resuelto). `SECURITY.md` existe (1,1 KB) y según la nota previa solo da un correo.
- **Contribuidores:** 72 (con anónimos). Top: `desplega-bot` 2.184, autor humano principal 1.158, `Worker Agent` 290, autor 4 77, autor 5 47, `capchase-bot` 36, `dependabot[bot]` 26.

## 9. Factor de bus y deuda

- **Factor de bus (DATO):** dos identidades concentran 3.342 de las contribuciones visibles de los 8 primeros; una es un bot operado por el propio proyecto (`desplega-bot`) y la otra una persona (autor humano principal, autor de `package.json` como `Taras`). INTERPRETACION: factor de bus efectivo cercano a 1 humano más una flota de agentes; si el humano o el servicio del enjambre (`api.desplega.agent-swarm.dev`, usado por CI para métricas y cobertura, `nightly-coverage.yml:46-67`, `merge-gate.yml:1128, 1164`) se detienen, parte de CI (informes) falla en silencio por `continue-on-error`.
- **Marcas de deuda (DATO):** `grep` de comentarios `TODO|FIXME|HACK` (con marcador de comentario y límite de palabra): src 0, apps/ui 0, apps/evals 0, packages 0, scripts 1. Un barrido más laxo (sin marcador) dio 18 coincidencias en 15 ficheros, incluidos falsos positivos. INTERPRETACION: el proyecto no usa marcas en el código (la deuda se guarda en issues, runbooks y "thoughts"); cifra cero no equivale a cero deuda.
- **Código muerto o sin llamador:** no medí (no hay herramienta de análisis en lo ejecutado; solo lectura). n/d. Señales indirectas: `repairRenumberedModelTiers` (parche puntual, `runner.ts:51-83`); un camino "legacy" de pausa para despliegues parciales en el shutdown (`runner.ts:2742-2746`).
- **Flags experimentales/beta (DATO):** runtime pools (`RUNTIME_POOLS_ENABLED`, por defecto off; nota previa y `runtime-pools.mdx`), multi-runtime (`isMultiRuntimeEnabled`, `src/http/active-sessions.ts:243`), `CHAT_SDK_ENABLED` (e2e README, `sut.ts`), `MCP_USER_EVENTS_ENABLED` (`src/http/index.ts:914`), `HEARTBEAT_DISABLE`, `OAUTH_KEEPALIVE_DISABLE`, `OAUTH_REFRESH_SWEEP_DISABLE` (`index.ts:951, 960, 968`), capabilities opcionales en `src/server.ts` (p. ej. `hasCapability("scheduling")`, `http/index.ts:937`). Hay catálogo de variables en `apps/ui/src/lib/configuration-catalog.ts` (2.252 líneas).

## 10. Riesgos de mantenimiento, ordenados

1. **Núcleo monolítico** (alto). `src/be/db.ts` 14.753 líneas/455 exports; `runner.ts` 8.334 con funciones de ~1.000 a ~2.000 líneas. Cada cambio de tareas o runner colisiona con ~35 PRs/día.
2. **Dependencia de un solo humano y de un servicio propio** (alto). Sección 9: contribuciones concentradas y CI que reporta a `api.desplega.agent-swarm.dev`.
3. **Superficie de cambio ancha** (medio-alto). Un proveedor exige 12+ ficheros; una ruta, 5-7 más e2e y OpenAPI regenerado (`harness-providers.md:903-952`, `CLAUDE.md:115-140`). La mitigación son tests de registro y e2e estricto.
4. **Acoplamiento y ciclos entre capas** (medio). Ciclos be<->workflows, be<->slack, be<->tasks; regla de dependencias transitiva solo en `warn` (`.dependency-cruiser.cjs:29-34`); `db.ts` usa imports dinámicos hacia workflows (`db.ts:3165`).
5. **Fiabilidad de reglas por convención** (medio). Parámetros del heartbeat leídos al cargar el módulo contradicen el comentario del propio fichero (`heartbeat.ts:176, 179, 189, 250` vs `129-135`); checksum de migración solo avisa (`runner.ts:282-288`).
6. **Pruebas sin modelo real** (medio). Los 198 escenarios e2e usan LLM y agente simulados (`scripts/e2e/README.md:3-5`); el comportamiento con harness reales se valida por QA manual (`harness-providers.md:956-960`). `src/tests` fuera del typecheck raíz (`tsconfig.json`, `exclude`).
7. **Dependencias de nicho y pines de SDKs** (medio). `xlsx` por tarball externo, SDKs de agentes de alta rotación, `@types/bun: latest`; `hoisted` con dependencias fantasma (`bunfig.toml`).
8. **Volumen documental** (bajo-medio). `thoughts/` 576 ficheros y `CHANGELOG.md` 369 KB; `CLAUDE.md` al 99 % de su presupuesto (23.724 de 24.000 bytes), lo que fuerza compresión y puede dejarlo atrasado respecto a los guardias (desfase de la sección 5, punto 1).
9. **Workflows temporales visibles** (bajo). Dos workflows "temporary/tmp" activos en GitHub (sección 4).

## 11. Cobertura y confianza

- **Cuantitativo (alta confianza):** recuentos de ficheros y líneas, tests, workflows, dependencias y escenarios e2e (comandos arriba), y metadatos de GitHub (`gh api` GET). Cifras de GitHub son de la fecha de lectura y pueden cambiar.
- **Medio:** contenidos de tests clasificados por regex (`Bun.serve`, `spawn`, `mock`), importaciones relativas (alias `@/` sin resolver), comparación por nombres de módulo.
- **Lectura directa:** `CLAUDE.md`, `.dependency-cruiser.cjs`, `package.json`, `biome.json`, `tsconfig.json`, `bunfig*.toml`, parte de `prek.toml`, `oxlint.config.ts`, `dependabot.yml`, `nightly-coverage.yml`, esqueleto de `merge-gate.yml` (nombres de jobs y 1100-1168), `scripts/e2e/README.md` (50 líneas), cabecera de `CONTRIBUTING.md` y de `check-db-boundary.sh`, y las secciones de código citadas en el informe de arquitectura. Menos de 1 % de las líneas de `src/` leídas línea a línea.
- **No visto o no medido:** contenido de `ci.yml`, `docker-and-deploy.yml` y de los jobs de `merge-gate.yml` salvo nombres; umbrales de cobertura; calidad de `apps/evals`; código muerto; tiempo de respuesta con una muestra de 10 issues (solo 6 disponibles; PRs 8+10 usadas como complemento); `SECURITY.md` releído solo por la nota previa.
