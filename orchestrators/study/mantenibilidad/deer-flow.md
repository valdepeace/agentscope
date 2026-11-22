# DeerFlow (bytedance/deer-flow) - eje MANTENIBILIDAD

Solo senales contables o leidas. DATO = medido o citado; INTERPRETACION = lectura mia. Rutas relativas a la raiz del repo; recuentos sobre git ls-files del clon de 1 commit.

## 1. Resumen y veredicto
- DATO: ~275.000 lineas de Python fuente (harness 204.336, app 68.192, extension-api 2.177), 610.254 lineas de tests Python (1.418 ficheros, 1.378 test_*.py) y 109.133 lineas de TS/TSX fuente frente a 109.398 de tests frontend.
- DATO: CI amplio (21 workflows en el repo; unit en 6 shards con Postgres y Redis, gate de I/O bloqueante, e2e, replay, lint con ruff y eslint/tsc), pre-commit y uv.lock; sin mypy/pyright ni cobertura declarada.
- DATO: proyecto muy activo en GitHub (3 releases: v2.0.0, v2.1.0, v2.2.0; 530 PRs y 597 issues abiertas; 617 PRs mergeadas en 30 dias) y concentrado en pocos mantenedores (top 4 con 610/285/258/203 commits).
- INTERPRETACION: mantenibilidad buena en proceso (tests densos, guias AGENTS.md por directorio verificadas en CI) y mala en forma (ficheros de 3.000-4.700 lineas, mucho acoplamiento interno, documentacion enorme).
- Veredicto: mantenible por un equipo que ya lo conoce; caro de asimilar para un tercero. Riesgo principal: volumen y velocidad de cambio (913 PRs abiertas en 30 dias).

## 2. Tamano y forma
Comandos: git ls-files 'backend/packages/harness/*.py' | xargs -d '\n' cat | wc -l (analogo para app, extension-api, tests, frontend con filtro por extension); wc -l ordenado para los mayores.

| Categoria | Ficheros | Lineas |
|---|---|---|
| Python fuente harness | 829 | 204.336 |
| Python fuente app | 172 | 68.192 |
| Python extension-api | 19 | 2.177 |
| Python tests backend | 1.418 (1.378 test_*.py) | 610.254 |
| TS/TSX fuente frontend | 645 | 109.133 |
| TS/TSX tests frontend (unit+e2e) | 497 | 109.398 |
| Docs backend/docs | 50 md | 18.788 |
| Docs docs/ | 130 | 32.083 |
| Skills (skills/) | 107 | no medido |
| Repo total | 4.770 ficheros | - |

Otros datos: CHANGELOG.md pesa 1,0 MB y hay README en 7 idiomas; ficheros no test con mas de 1.000 lineas: 72 (git ls-files | grep .py/.ts/.tsx | wc -l | awk '$1>1000', incluye ficheros i18n).

Los 10 ficheros fuente mas grandes (wc -l):
1. frontend/src/core/threads/hooks.ts 4.749 - hooks de hilos y streaming de la UI.
2. backend/app/channels/manager.py 4.418 - gestor de canales IM.
3. backend/packages/harness/deerflow/runtime/runs/worker.py 4.332 - run_agent y finalizacion.
4. frontend/src/components/workspace/input-box.tsx 3.921 - caja de entrada.
5. backend/packages/harness/deerflow/sandbox/tools.py 3.773 - tools de sandbox.
6. backend/packages/harness/deerflow/integrations/lark_cli.py 3.716 - integracion lark.
7. backend/app/gateway/services.py 3.716 - admision y arranque de runs (start_run :2478-3285).
8. backend/packages/harness/deerflow/runtime/runs/manager.py 3.706 - RunManager, leases.
9. backend/packages/harness/deerflow/community/e2b_sandbox/e2b_sandbox_provider.py 3.189.
10. backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py 3.083.
(Siguientes: i18n en-US.ts 2.905, mock-api.ts de e2e 2.829, subagents/executor.py 2.731.)

Complejidad evidente (leida): start_run de ~800 lineas con decenas de ramas (services.py:2478-3285); run_agent de ~1.600 lineas (worker.py:1045-2650); execute_async con manejo de carreras de cancelacion en 140 lineas de comentarios y ramas (subagents/executor.py:2401-2540); deps.py:830 langgraph_runtime. INTERPRETACION: hot spots de cambio; cada arreglo toca estas funciones.

## 3. Pruebas
- DATO: ratio lineas test/codigo Python = 610.254 / 274.705 = 2,2; frontend = 1,0.
- Tipos: unit offline (mayoria, backend/tests/), estrictos de I/O bloqueante (backend/tests/blocking_io, gate con Blockbuster), integracion con Postgres/Redis (marcador integration, 4 ficheros), live con APIs reales (marcador live, 12 ficheros; solo con DEER_FLOW_RUN_LIVE_TESTS=1, backend/Makefile:32-33), e2e Playwright (109 ficheros en frontend/tests/e2e mas e2e-auth, e2e-real-backend, e2e-record), replay golden de LLM (backend/tests/replay_provider.py), tests de skills en tests/skills y de ejemplos de extension (marcador bundled_extension), benchmarks (marcador benchmark) (backend/pyproject.toml, tool.pytest.ini_options.markers).
- Modelo real o simulado: la regla escrita es sustituir solo la frontera exterior (modelo, transporte, reloj) (backend/tests/AGENTS.md, "What a test must exercise"); 67 ficheros de test usan dobles de modelo (grep de Fake*Model, replay_provider); los tests live son los unicos con modelo real y son opt-in (Makefile:32).
- Cobertura por modulo (DATO, metodo: ficheros de test que importan el modulo con grep; no es cobertura de lineas): config 542, app.gateway 322, agents 307, runtime 286, persistence 274, community 125, sandbox 105, app.channels 104, tools 101, skills 90, extensions 86, subagents 73, mcp 62, models 57, authz 39, tui 23, uploads 20, projects 19, knowledge 18, guardrails 13, openai_decisions 1, tracing 3, approvals 3.
- Modulos sin tests: ninguno de los subpaquetes del harness sale con 0; los mas flojos por importacion son openai_decisions (1), tracing (3) y approvals (3). El frontend src/ no tiene test por directorio medido; no lo verifique modulo a modulo.
- Suite por shards con duraciones guardadas (backend/.test_durations, backend/Makefile:49-52).

## 4. Integracion continua y calidad
Workflows del repo (.github/workflows, 21 ficheros): Unit Tests (6 shards, Postgres 17 + Redis 7, make test-shard; unit.yml completo en backend-unit-tests.yml), default-install-collection (pytest --collect-only con solo --group dev), Backend Blocking IO, Lint Check (agent-guidance, frontend-scope-reads, label-declarations, lint-backend con uv lock --check + make lint, lint-frontend con format, lint, typecheck y build), Frontend Unit Tests, E2E Tests, Replay E2E (front-back contract), Bundled Extension Tests, Extension Package Check, Jev Plugin Package, Skill Review CI, Backend Benchmark Tests, Windows Live Test Policy, Sandbox Image Smoke, Triage, Label Sync, Nightly Build, Publish Containers/Helm Chart/lark-cli/Sandbox Network Proxy, Verify Versions (versiones de tag coherentes con pyproject, package.json, Chart.yaml y uv.lock).
- Alcance por cambios: scripts/ci_changed_scopes.py decide backend/frontend (backend-unit-tests.yml jobs changes); falla abierto en push.
- Estado en main (GET actions/runs): ultimo push 2026-11-21 con Lint, Unit, Blocking IO, Replay, Frontend y Bundled en success; varias ejecuciones previas canceladas por concurrencia (cancel-in-progress). 33 workflows activos segun GitHub, mas de los 21 ficheros del clon (incluye Copilot, Dependabot, CodeQL y otros nombres que no vi como fichero: n/d el origen).
- Linter: ruff con reglas E,F,I,UP, line-length 240, py312 (backend/ruff.toml:1-8); make lint = ruff check + ruff format --check (backend/Makefile:74-76). Frontend: eslint, prettier --check, tsc --noEmit (frontend/package.json scripts).
- Tipado estricto: no encontre mypy ni pyright en toml/yml/Makefile (grep). TS con tsc --noEmit.
- Pre-commit: ruff, ruff-format, uv lock --check, eslint, prettier (.pre-commit-config.yaml).
- Cobertura declarada: ninguna (sin pytest-cov ni --cov en pyproject, Makefile o workflows).
- Gates propios notables: check_agent_guidance.py (presupuesto de tamano de AGENTS.md), test_ci_uv_version_pin.py (uv 0.11.1 igual que Dockerfile).

## 5. Documentacion
- DATO: AGENTS.md raiz y por directorio (guias cercanas al codigo), CLAUDE.md como shim con @AGENTS.md (CLAUDE.md:1-5; backend/CLAUDE.md), backend/docs (50 md), docs/ (130), docs/guide, README en 7 idiomas, config.example.yaml de 197 KB, CONTRIBUTING.md 20 KB.
- Quien manda: el codigo; las guias declaran "tools, parameters and limits are in the code" (tools/AGENTS.md cabecera; agents/middlewares/AGENTS.md: "Read those two functions for the current list").
- Links relativos: 502 enlaces en AGENTS/CONTRIBUTING/README/docs comprobados con script; los 10 que fallan son falsos positivos (texto entre corchetes en blob-storage.md:24 y CODE_CHANGE_SUMMARY_BY_FILE.md). INTERPRETACION: la red de docs esta bien cuidada.
- Desfases comprobados:
  1. SECURITY.md:5 dice "doesn't provide an official release yet"; GitHub (GET releases) lista v2.0.0 (2026-06-25), v2.1.0 (2026-09-24), v2.2.0 (2026-11-04).
  2. backend/packages/harness/deerflow/tools/tools.py:275 comenta que el Gateway "runs in a separate process"; backend/AGENTS.md:3-7 afirma que el runtime del agente corre dentro del Gateway en todo despliegue.
  3. backend/app/gateway/deps.py:814 y :840 remiten a backend/CLAUDE.md "Config Hot-Reload Boundary"; backend/CLAUDE.md es un shim de 5 lineas y el grep de ese titulo en *.md de backend no lo encuentra (no encontrado en lo leido).
  4. Timeout de subagente: dataclass 900 (subagents/config.py:55) vs global 1800 aplicado a builtins (config/subagents_config.py:185-188); config.example.yaml:2077-2078 lo documenta, pero un lector del dataclass se equivoca.
- Legibilidad: docs/guide/subagents.md:174 es un parrafo de una sola linea de mas de 3.000 caracteres.

## 6. Dependencias
- Gestor: uv (workspace con harness y extension-api, backend/pyproject.toml tool.uv.workspace) con backend/uv.lock (258 paquetes [[package]]); frontend pnpm 10.26.2 con pnpm-lock.yaml (~1.256 entradas en packages:), 80 dependencies + 22 devDependencies.
- Python directas: harness 44, app 20 (conteo de lineas entre comillas del bloque dependencies). Casi todas con cota inferior ">=" y pocas con techo (langgraph>=1.2.9,<1.3; sqlalchemy<3; starlette>=1.3.1,<2; websockets forzado ==16.0 via override-dependencies, backend/pyproject.toml tool.uv). El lock fija versiones reales.
- Pins exactos deliberados: deerflow-extension-api==0.2.13, langchain-openviking==0.1.0 (harness pyproject).
- Pesadas o de riesgo: kubernetes, markitdown[all,xlsx], duckdb, langgraph-api/-cli, langfuse, firecrawl, e2b-code-interpreter (tambien en app), ~10 SDK de proveedores y de IM (lark-oapi, slack-sdk, python-telegram-bot, dingtalk-stream, wecom-aibot). Dependencia de API privada de starlette (starlette._utils.get_route_path) declarada y acotada con comentario (backend/pyproject.toml). Override de websockets por un pin de langgraph-sdk que el propio comentario declara infundado.
- Extras opcionales para proveedores raros (tenki, boxlite, opensandbox, boxd, browser) para mantener el nucleo ligero.
- Dependabot activo (116 commits de dependabot[bot] en la lista de contribuidores).

## 7. Convenciones y onboarding
- CONTRIBUTING.md (20 KB), CODE_OF_CONDUCT.md, plantilla de PR y de issues (bug-report.yml, feature-request.yml), RELEASING.md, Install.md. GitHub community/profile: health_percentage 87; issue_template aparece null en la API aunque existe .github/ISSUE_TEMPLATE (n/d la causa).
- AGENTS.md raiz: leer todos los AGENTS.md desde el directorio hasta la raiz antes de editar; "Evaluate an extension first"; comandos make config/install/dev (AGENTS.md raiz).
- Como anadir una pieza: proveedor/tool/sandbox por config use: modulo:clase (0 cambios de core); canal: editar _CHANNEL_REGISTRY (app/channels/service.py:33-44); migracion: make migrate-rev MSG=... (backend/AGENTS.md Rules); guia por area en la tabla de backend/AGENTS.md.
- Estilo: ruff format (240 columnas) y regla de pruebas explicita (backend/tests/AGENTS.md).
- INTERPRETACION: onboarding bueno para agentes de codigo y para quien lee las guias; abrumador para humanos por volumen.

## 8. Salud del proyecto via GitHub (solo GET, 2026-11-21)
- Repo: 85.582 estrellas, 12.072 forks, creado 2025-05-07, ultimo push 2026-11-21T12:47Z, licencia MIT, no archivado. Lenguajes: Python 39,9 MB, TypeScript 7,1 MB.
- Releases (solo hay 3, la API no devolvio mas): v2.2.0 2026-11-04, v2.1.0 2026-09-24, v2.0.0 2026-06-25. Cadencia ~1-2 meses. Version en main: 2.3.0-dev (backend/pyproject.toml). Las "ultimas 5" no existen: n/d.
- Abiertas ahora: 530 PRs y 597 issues (search API); en los ultimos 30 dias: 289 issues abiertas, 221 cerradas, 913 PRs abiertas, 617 mergeadas.
- Muestra A (10 issues mas recientes, creadas 20-21 nov): 6 sin ningun comentario, 3 con primer comentario humano a 1,5 h, 1,6 h y 4,6 h; una cerrada a 11,2 h. Sesgo: son muy recientes.
- Muestra B (10 issues del 20-27 oct): 7 cerradas, 3 abiertas; tiempo hasta cierre 4,0 h, 26,6 h, 46,7 h, 53,6 h, 94,7 h, 148,1 h, 402,8 h; solo 2 con comentario humano previo (54,4 h y 4,1 h).
- Muestra B de PRs (10, del 20-27 oct): 8 mergeadas (1,5 h, 2,9 h, 4,0 h, 13,2 h, 52,2 h, 53,5 h, 91,8 h y 93,9 h), 1 cerrada sin merge (81,8 h), 1 abierta. Primer comentario humano en solo 1 de 10 (0,7 h).
- Ficheros de comunidad: CODE_OF_CONDUCT, CONTRIBUTING, PR template, licencia, README (profile API), SECURITY.md en el repo.
- Bus factor (GET contributors, 10 primeros): MagicCube 610, WillemJiang 285, hetaoBackend 258, henry-byted 203, hyeonsang010716 169, AnnaSuSu 132, dependabot[bot] 116, ggnnggez 103, LofiSu 100, DaoyuanLi2816 85. Ultimos 100 commits (15-21 nov): 12 de hyeonsang010716, 8 de fancyboi999, resto repartido entre >10 autores. INTERPRETACION: base de contribuidores amplia; el nucleo historico es de 4-5 personas.

## 9. Factor de bus y deuda
- TODO/FIXME/HACK/XXX: 58 coincidencias con \b(TODO|FIXME|HACK|XXX)\b (git ls-files | grep -v CHANGELOG/lock/imagenes | grep -InE). 28 estan en backend/tests/test_sandbox_search_tools.py (datos de prueba, no deuda), 7 en skills/public/skill-creator/scripts/init_skill.py (plantilla). Deuda marcada real: <20. INTERPRETACION: la deuda se gestiona en issues y docs, no con marcas.
- Codigo sin importador (heuristica propia sobre 712 modulos del harness: busqueda de la ruta de modulo en fuentes, config y pyproject): 139 sin importador estatico en fuente, 73 solo referenciados por tests y 66 en ningun sitio; los 66 son en su mayoria migraciones Alembic (cargadas por nombre de fichero) mas unos pocos ficheros (agents/memory/backends/noop/config.py, agents/memory/backends/openviking/session.py, community/aio_sandbox/ownership/base.py, community/e2b_sandbox/capacity/redis.py, extensions/model_schema_worker.py). No lo valide: la carga dinamica por reflexion (use: modulo:clase) hace que sean candidatos, no codigo muerto confirmado.
- Flags experimentales / opt-in: skill_evolution.enabled (tools/tools.py: skill_manage_tool), live tests opt-in, providers comunitarios por extras, plan mode, browser (extra playwright); backend/docs/plans y docs/experiments con trabajo sin consolidar (nota previa; no reverifique).
- Version de desarrollo 2.3.0-dev (backend/pyproject.toml); ramas mantenidas main (2.x) y main-1.x (SECURITY.md:6-7).

## 10. Riesgos de mantenimiento (orden de gravedad)
1. Ritmo de cambio vs capacidad de revision: 913 PRs abiertas en 30 dias y 530 abiertas ahora; en muestra de 10 PRs solo 1 tuvo comentario humano. Evidencia: busquedas search/issues de la seccion 8.
2. Ficheros monoliticos y funciones de 800-1.600 lineas en el camino critico (services.py:2478-3285, worker.py:1045-2650, 4.700 lineas en hooks.ts). Cada cambio de run toca los mismos puntos.
3. Acoplamiento circular entre subpaquetes del harness (23 pares, ver arquitectura seccion 3) que contradice la guia de capas (extensions/AGENTS.md "Composition").
4. Sin tipado estatico Python ni cobertura declarada (ruff E,F,I,UP solamente, backend/ruff.toml:5). Mitigado por tests densos y gates especificos.
5. Volumen documental (180 ficheros md, 50 KB+ en config.example.yaml) con desfases puntuales (seccion 5) y parrafos de miles de caracteres.
6. Dependencias: ~65 directas Python con cotas laxas, pins exactos de contrato, un override de websockets, API privada de starlette. Lock mitiga.
7. Releases escasos (3) frente al volumen de cambios; SECURITY.md desactualizado.
8. Nucleo de mantenedores concentrado (4 con 1.356 de los commits listados).

## 11. Cobertura y confianza
- Alta confianza (comando exacto): recuentos de ficheros y lineas, listas de workflows, ruff.toml, pyproject, pre-commit, TODOs, metadatos GitHub (GET).
- Confianza media: ratios de test por modulo (por importacion, no por lineas ejecutadas); heuristica de codigo sin importador; ciclos (texto de imports, incluye TYPE_CHECKING); tiempos de respuesta (muestras de 10, sesgo temporal).
- No evaluado: ejecucion real de CI y tests (no se ejecuto nada), cobertura efectiva, seguridad de dependencias, rendimiento. Lectura de codigo fuente ~1-2 % de lineas; guias y config mas.
- n/d: ultimas 5 releases (solo existen 3); causa de issue_template null en community/profile.
