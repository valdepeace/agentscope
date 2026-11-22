# Mantenibilidad: LofiSu/dsh-super-agent (paquete `ds-agent`)

Solo señales contables o legibles. DATO = medido o citado; INTERPRETACIÓN = lectura mía. Clon superficial en `references/deepseek-super-agent/`; rutas relativas a su raíz. Nada del repo se ejecutó. Metadatos de GitHub solo por GET (`gh api`), consultados el 2026-11-21.

## 1. Resumen y veredicto

- Código: 35 281 líneas en 172 ficheros .ts de `src`; tests 43 079 líneas en 167 ficheros; docs 16 513 líneas en 60 ficheros. Hay más líneas de test que de código (1,22 a 1).
- CI exigente en lo declarativo (lint de invariantes, typecheck estricto, tests offline, cobertura sin umbral) pero sin formateador ni linter de estilo.
- Un solo autor (375 de 375 commits de LofiSu) y un workflow que cierra automáticamente cualquier issue o PR ajeno: factor de bus 1 por política.
- Documentación abundante y con desfases comprobables (`docs/BROWSER_ERRANDS.md`); 0 marcas TODO/FIXME/HACK en código.
- Veredicto: mantenible por su autor (tests densos, tipado estricto, CI verde); frágil para terceros (sin CONTRIBUTING, ficheros dios de 3231 y 1159 líneas, 68 variables de entorno, ninguna release publicada).

## 2. Tamaño y forma

Comandos exactos (desde la raíz del repo):
- `git ls-files | awk -F/ '{print $1"/"$2}' | sort | uniq -c | sort -rn`
- `for d in src/*/; do echo "$d $(git ls-files $d | wc -l) $(git ls-files $d | xargs cat | wc -l)"; done`
- `git ls-files src | xargs wc -l | sort -rn | head -16`
- `git ls-files test | wc -l; git ls-files test | xargs cat | wc -l` y lo mismo con `docs`.

DATO:

| Área | Ficheros | Líneas |
|---|---|---|
| `src` (.ts) | 172 | 35 281 |
| `test` (todo, 148 `*.test.ts`) | 167 | 43 079 |
| `docs` | 60 | 16 513 |
| `README.md` / `PLAN.md` / `docs/MODULES.md` / `docs/DEVELOPMENT_STATUS.md` | - | 485 / 1112 / 538 / 147 |
| `scripts/` | 14 visibles (listado recortado) | no contadas |
| Total ficheros versionados | 437 | - |

Diez ficheros más grandes de `src`:

| Líneas | Fichero | Qué hace |
|---|---|---|
| 3231 | `src/runtime.ts` | Cableado global, ingress, recuperación, mesh remoto |
| 1159 | `src/harness/loop.ts` | `Harness`, cola de mesh, bucle ReAct |
| 1074 | `src/cli/main.ts` | CLI `ds-agent` |
| 1068 | `src/harness/remote-node.ts` | Nodos remotos A2A |
| 1056 | `src/env/browser-errands.ts` | Adaptador de navegador |
| 875 | `src/harness/tools.ts` | Registro y definición de herramientas |
| 773 | `src/channels/imessage.ts` | Canal iMessage |
| 759 | `src/harness/proactive.ts` | Auto-despertar proactivo |
| 668 | `src/memory/long-term.ts` | Memoria a largo plazo |
| 645 | `src/store/backup.ts` | Backup y restauración |

Complejidad evidente (DATO): `Runtime.create` va de `src/runtime.ts:533` a la l.2097 donde empieza el siguiente método (~1560 líneas en un método estático); `Runtime` completa ocupa l.395-3165. `src/harness/loop.ts` mezcla tres responsabilidades (`Harness` l.134, mesh l.356-537, bucle ReAct l.1029). INTERPRETACIÓN: `runtime.ts` es el punto de mayor coste de cambio; cualquier característica nueva lo toca (importa de 12 directorios).

## 3. Pruebas

DATO (método: regex `from '../src/<mod>'` sobre cada test; un test que importa varios módulos cuenta en cada uno, así que las líneas de test se solapan y son una cota, no una proporción exacta):

| Módulo src (líneas) | Ficheros de test que lo importan |
|---|---|
| shared (598) | 63 |
| harness (7061) | 62 |
| channels (3006) | 52 |
| root: runtime/index (3293) | 50 |
| store (1419) | 35 |
| evolution (3770) | 24 |
| core (2952) | 23 |
| env (2560) | 21 |
| ingress (498) | 13 |
| gateway (2108) | 13 |
| memory (1234) | 12 |
| metrics (640) | 11 |
| cli (4400) | 11 |
| watchdog (402) | 8 |
| speculate (117) | 5 |
| measurement (474) | 3 |
| ui (749) | 1 |

- Módulos sin ningún test que los importe: ninguno. Con muy poco contacto: `src/ui` (1 fichero) y `src/measurement` (3). `src/cli` (4400 líneas) solo en 11 ficheros, aunque hay varios `test/cli-*.test.ts` que pueden invocarlo por proceso (no leídos).
- Tipos: unitarios y de integración sobre Runtime (`test/runtime-integration.test.ts`, `test/runtime-wiring.test.ts`); recuperación con fallo de SO (`test/restart-recovery-os-fault.test.ts`, `test/external-effects-os-fault.test.ts`, `test/delegate-agent-os-fault.test.ts`); procesos remotos (`test/remote-mesh-os-process.test.ts`); sondas de sandbox (`test/sandbox-enforcement-probe.test.ts`); e2e con modelo real (`test/live-demo.test.ts`, `live-e2e`, `live-modules`, `team-live`, `demos-real-channels`). Solo nombres; no abrí ninguno.
- Modelo real o simulado: `pnpm test` rellena por defecto un gateway local (`DS_AGENT_API_BASE=http://127.0.0.1:3425/v1`, clave `magpie`; `package.json:28`) y `vitest.config.ts:7-9` carga `test/setup-live.ts` con timeouts de 180 s. En CI se ejecuta con `DS_AGENT_ALLOW_MOCK_FIXTURE=1` y las claves vaciadas (`.github/workflows/ci.yml:73-93`): modelo simulado. Existe `test/scripted-model.ts`.
- Saltos: 9 ficheros usan `skip`/`skipIf` (`git grep -lE "skipIf|\.skip\(|describe.skip|it.skip" -- test`): demos-real-channels, evolution-isolate-eval-env, gateway, live-demo, live-e2e, live-modules, runtime-remote-mesh-stale-timeout, sandbox-read-isolation, team-live; 7 ocurrencias en total por línea. El "15 saltados" de la documentación no se reverificó.
- INTERPRETACIÓN: la suite offline de CI usa un modelo guionado, así que valida la mecánica, no el comportamiento con un modelo real; ese lo cubre solo el workflow manual `Live acceptance`.

## 4. Integración continua y calidad

Cuatro workflows (`gh api .../actions/workflows` y lectura de ficheros):

| Workflow | Disparo | Qué comprueba |
|---|---|---|
| `CI` (`ci.yml`) | push a main y PR a main | job `check` (timeout 10 min): instala bubblewrap, `pnpm lint`, `pnpm format:check`, `pnpm typecheck`, `pnpm test` offline, `pnpm test:coverage` (sin umbral, l.95-97); job `demo` solo si hay clave de modelo (l.130-176) |
| `Release check` (`release.yml`) | manual | lo anterior más `pnpm build`, `scripts/release-check.ts`, `npm pack` y sube el tarball; no publica (l.3-4) |
| `Live acceptance` (`acceptance.yml`) | manual | `pnpm test` con modelo real y `pnpm demo` contra GitHub real (crea issues); 45 min |
| `Owner-only submissions` (`owner-only.yml`) | issues, PR, comentarios, revisiones, discusiones | cierra, bloquea o borra lo de cualquiera que no sea `LofiSu` ni un bot (l.35) |

- Gates: `CLAUDE.md:94` dice que `typecheck and test` es check requerido en main con `enforce_admins`; no verifiqué la protección de rama (no consulté ese endpoint).
- Lint: `scripts/lint.ts` (11,1 KB) no es un linter de código; comprueba invariantes del repositorio (tabla de ROADMAP, nombres prohibidos; `CLAUDE.md:23-25`, `scripts/lint.ts:76-110`). Lo leí solo en parte.
- Formato: `format:check` es `git diff --check` (`package.json:33`), es decir solo espacios en blanco. No existe ESLint, Prettier, Biome, `.editorconfig`, pre-commit ni husky (`ls` negativo sobre esos nombres).
- Tipado: `strict`, `noUnusedLocals`, `noUnusedParameters` (`tsconfig.json:6,12-13`); `tsc --noEmit` abarca src, test y scripts. Sin `noUncheckedIndexedAccess`. Supresiones o `any` en src y test: 11 líneas (`git grep -cE "@ts-ignore|@ts-expect-error|as any|: any\b" -- src test`, suma).
- Cobertura: se mide (`@vitest/coverage-v8`, `vitest.config.ts:11-20`) pero sin umbral de fallo; la cifra base se dice registrada en `docs/ROADMAP.md` P2-5 (no abierto).
- Historial reciente de CI (últimas 20 ejecuciones, GET): casi todas `success`; 1 `failure` el 2026-11-20T10:49Z en una PR; las de `Owner-only` salen `skipped` (actor propietario).
- INTERPRETACIÓN: el job `demo` se llama "demo fixture smoke (offline; not real-channel acceptance)" (`ci.yml:131`) pero exige clave real y aborta si cae a `MockModel` (l.137, 159-176); el nombre induce a error.

## 5. Documentación

DATO: `README.md` (485 líneas) y `README.zh-CN.md`; `CLAUDE.md` (instrucciones para agentes); `PLAN.md` (1112 líneas); `docs/` con 60 ficheros: arquitectura, módulos (`MODULES.md`, 538 líneas), seguridad, aprobaciones, recuperación, evolución controlada, contenedor, release, upgrade, roadmap, revisiones de código fechadas (`LIVE_CODE_REVIEW_2026-11-01.md`, `HARDENING_2026-11-01.md`), registros de PR (`PR_REVIEW_LOG.md`, `AGENT_LOG.md`), 7 planes/specs en `docs/superpowers/` y traducción china parcial (`docs/zh/`, 4 ficheros). Hay también `.pr-body-controlled-evolution.md` en la raíz.
- Quién manda: el código, con los docs como registro. `CLAUDE.md:23-25` y `scripts/lint.ts` imponen reglas de nombres y de tablero de ROADMAP sobre los docs, lo que indica que los docs se validan mecánicamente en parte.
- Desfase comprobado 1: `docs/BROWSER_ERRANDS.md:3` dice "Status: **PR-only** on `feat/instinct-browser-errands`, not merged to `main`", pero `src/harness/tools.ts:105` registra `browserErrandTools` si hay `deps.browserErrands`, `src/runtime.ts:127` declara la opción y GitHub lista la PR #172 "feat: add owner-gated browser errand adapter" con `merged_at` 2026-11-19T12:00:52Z.
- Desfase comprobado 2 (nombre): `.github/workflows/acceptance.yml:21` usa por defecto `LofiSu/deepseek-super-agent` mientras el repo consultado es `LofiSu/dsh-super-agent` y el paquete es `ds-agent`. INTERPRETACIÓN: renombrado del repositorio sin actualizar ese valor; no comprobé si GitHub redirige.
- INTERPRETACIÓN: volumen documental alto (16,5 mil líneas, casi la mitad del código) con parte de historia y registros; el coste de mantenerlo al día es real y ya hay un desfase.

## 6. Dependencias

DATO (`package.json:46-55`): 1 dependencia de ejecución, `@modelcontextprotocol/sdk` fijada exacta en `1.32.2`; 5 de desarrollo con caret (`@types/node`, `@vitest/coverage-v8`, `tsx`, `typescript`, `vitest`). Gestor pnpm 10 (`ci.yml:29-32`), `pnpm-lock.yaml` de 70,7 KB, instalación con `--frozen-lockfile` (`ci.yml:41`). `engines.node >=22` (l.19-21). `pnpm.onlyBuiltDependencies: ["esbuild"]` (l.41-45). Sin Dependabot ni Renovate: `git ls-files` solo muestra 4 ficheros bajo `.github`, todos en `workflows/`.
- Riesgo: la persistencia usa `node:sqlite` (`src/store/sqlite.ts:3`), módulo del núcleo de Node 22. Todos los scripts de `package.json` prefijan `NODE_NO_WARNINGS=1` (l.23-38). INTERPRETACIÓN: se silencian los avisos de la API experimental de SQLite; no lo comprobé en la documentación de Node.
- Riesgo: ejecución obligatoria en macOS o Linux con `bwrap`/`sandbox-exec` para el Worker (`src/env/seccomp.ts:160-190`); en Windows lanza error.
- El SDK MCP es la única superficie externa de terceros en producción; el resto de red usa `fetch`/`gh` (no verificado a fondo).

## 7. Convenciones y onboarding

- Existe: `CLAUDE.md` (comandos, reglas de nombres M0-M11, reglas de PR: una preocupación por PR, nunca commit directo a main), `.claude/agents` (architect, code-reviewer, implementer, qa-engineer, security-reviewer), skill `autonomous-engineering` y hook `super-agent-stop-guard.py`. `docs/CODE_REVIEW_METHOD.md` y `docs/MODULES.md` describen el método y los módulos.
- No existe: `CONTRIBUTING.md`, guía de estilo, plantillas de issue/PR, código de conducta (perfil de comunidad GET: solo `license` y `readme` presentes; salud 57 %). `CLAUDE.md:4` declara el repositorio "owner-only" y `owner-only.yml` lo hace cumplir.
- Cómo se añade una pieza: no hay guía; se deduce del código (herramienta = un objeto en `tools.ts`; proveedor = una entrada en `providers.ts:39`; canal = >= 3 ficheros). Ver el estudio de arquitectura, sección 6.
- Onboarding automatizable: `pnpm demo:local` y `pnpm ds-agent` (`package.json:25-34`); los tests por defecto necesitan un gateway local de modelo en el puerto 3425 (INTERPRETACIÓN: barrera de entrada para un colaborador nuevo, que debe usar `pnpm test:coverage` con `DS_AGENT_ALLOW_MOCK_FIXTURE=1` para correr offline).

## 8. Salud del proyecto vía GitHub (solo GET)

DATO:
- Repo creado 2026-10-31; último push 2026-11-21T21:26Z; 146 estrellas; 19 forks; no archivado; rama por defecto `main`; 94 ramas.
- Releases: ninguna (`releases` devolvió vacío); etiquetas: ninguna (`tags` vacío). Fechas de las últimas 5 releases: n/d, no existen. `package.json` está en `0.1.0`; `release.yml` no publica.
- Commits: 375 (cabecera `Link` con `per_page=1`, última página 375); los 100 últimos se reparten entre 2026-11-05 y 2026-11-19 y todos son de `LofiSu`; contributors: `LofiSu` 375.
- Issues: 11 en total (números 23-28, 47-50, 52), todas creadas por `LofiSu` y con títulos `[ds-agent demo ...]`: son artefactos de la demo con GitHub real. 1 abierta (#28), 10 cerradas en segundos o minutos. Ninguna issue de usuario externo.
- PR: 171 en total (182 números menos 11 issues). Sobre las 100 más recientes: 91 fusionadas, 7 abiertas, 2 cerradas sin fusionar. Tiempo de apertura a fusión de esas 91: mediana 0,23 h, percentil 90 36,6 h, máximo 261,8 h (cálculo propio sobre `created_at`/`merged_at`).
- PR abiertas (7): #182, #180, #177, #175, #174, #173, #167. La más antigua (#167) abierta desde 2026-11-07.
- Muestra de 10 PR (las 10 más recientes): las 10 tienen comentario de `coderabbitai[bot]` a los 8-22 s de abrirse; revisiones formales de CodeRabbit en 6 de ellas (varias en #174, #180); revisiones humanas: ninguna. Tiempo de respuesta humana: n/d.
- Ficheros de comunidad: licencia MIT y README sí; CONTRIBUTING, código de conducta, plantillas de issue y de PR no.
- INTERPRETACIÓN: cadencia de integración muy alta (decenas de PR por semana, la mayoría cerradas en menos de una hora), pero es una sola persona más un revisor automático, y varias PR de funciones grandes (#173, #174, #175, #177) llevan 1-2 semanas abiertas.

## 9. Factor de bus y deuda

- Factor de bus: 1 (DATO: 375/375 commits; workflow que rechaza a cualquier otro actor, `owner-only.yml:31-35`; `CLAUDE.md:4`). El proyecto se construye con agentes Claude Code orquestados (`CLAUDE.md:37-99`), lo que reparte trabajo pero no conocimiento humano.
- Marcas `TODO`/`FIXME`/`HACK`/`XXX` en `src`, `test`, `scripts`: 0 (`git grep -nE "TODO|FIXME|HACK|XXX" -- src test scripts`). INTERPRETACIÓN: la deuda se escribe en docs (`ROADMAP.md`, `DEVELOPMENT_STATUS.md`), no en el código.
- Supresiones de tipo o `any`: 11 líneas en src y test.
- Flags experimentales: 68 variables `DS_AGENT_*` distintas en `src` (`git grep -ohE "DS_AGENT_[A-Z0-9_]+" -- src | sort -u | wc -l`); no conté cuántas son interruptores experimentales (el conteo de la palabra "experimental" falló por un error de herramienta y no lo repetí).
- Código muerto o sin llamador: no se hizo detección sistemática. Observación puntual: `sandboxEnabledFromEnv` ignora su argumento y devuelve `true` (`src/env/seccomp.ts:194-196`); su parámetro `_env` está sin uso real (no busqué sus llamadores).
- Deuda declarada (heredada, no recomprobada): ver notas anteriores sobre embeddings hash, mesh remoto sin listener por defecto y contenedor solo como receta.

## 10. Riesgos de mantenimiento, ordenados

1. Factor de bus 1 y contribución externa bloqueada por automatización (`owner-only.yml:31-35`; contributors: 1).
2. Módulo dios `src/runtime.ts`: 3231 líneas, un método de ~1560 líneas (`:533-2097`), importa de 12 directorios; todo cambio de cableado pasa por él.
3. Ningún artefacto publicado: 0 releases, 0 tags, versión 0.1.0; no hay política de compatibilidad verificable más allá de `docs/UPGRADE.md` (no abierto) y las migraciones SQLite v1-v4.
4. Sin formateador ni linter de código (`format:check` es `git diff --check`); el estilo depende de convención y de revisión automática.
5. Suite por defecto acoplada a un gateway local de modelo (`package.json:28`) y 9 ficheros con saltos; la CI valida con modelo guionado.
6. Ciclos entre módulos (core <-> channels, core -> speculate -> evolution) que dificultan extraer piezas; ver arquitectura, sección 3.
7. Desfase documental ya presente (`docs/BROWSER_ERRANDS.md:3`) y volumen alto de docs (60 ficheros) mantenidos a mano.
8. 7 PR abiertas de funciones grandes sin fusionar con hasta 14 días de antigüedad: riesgo de divergencia con `main`, que cambia a diario.
9. Dependencia de `node:sqlite` y de binarios de sandbox solo macOS/Linux.
10. 68 variables de entorno como superficie de configuración.

## 11. Cobertura y confianza

- Leído en detalle: `package.json`, `tsconfig.json`, `vitest.config.ts`, los 4 workflows, `CLAUDE.md`, y los tramos de código listados en el estudio de arquitectura (unas 1830 líneas de src, ~5 %). Parcial: `scripts/lint.ts` (~100 de unas 300 líneas), `docs/BROWSER_ERRANDS.md` (6 líneas).
- Medido sin leer: tamaños, conteos de ficheros y líneas, grafo de importaciones, relación test-módulo (regex), TODO, variables de entorno, listado de docs y tests (solo nombres).
- GitHub: `gh api` solo GET. Las listas de issues y PR se pidieron con `per_page=100`; las estadísticas de PR son sobre las 100 más recientes de 171. Los tiempos de respuesta se miden sobre fusión (no hay revisiones humanas que medir) y sobre comentarios de bot.
- Confianza: alta en tamaños, workflows, dependencias, salud GitHub y ausencia de ficheros de comunidad; media en proporciones de test (solapamiento por módulo); baja en calidad interna de los tests, protección de rama, cobertura real y código muerto (no medidos).
- Correcciones respecto a la nota anterior: ver sección 9 del estudio de arquitectura (BROWSER_ERRANDS desfasado y confirmado con PR #172; 167 ficheros de test, no 153; 60 ficheros de docs, no ~45).
