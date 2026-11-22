# Mantenibilidad: awslabs/cli-agent-orchestrator (CAO)

Solo lectura. Rutas relativas a `references/cli-agent-orchestrator/`. DATO = contado o leído con cita; INTERPRETACION = lectura mía. Clon superficial: no hay historial git; las métricas de salud salen de `gh api` (solo GET), fecha de consulta 2026-11-21.

## 1. Resumen y veredicto

- DATO: 317 ficheros .py de código (166.505 líneas) frente a 907 de test (399.459 líneas): ratio 2,4 líneas de test por línea de código; CI con 16 workflows y puertas de formato, dead-code, tipado con "ratchet", cobertura y seguridad de dependencias.
- DATO: proyecto muy activo y reciente (repo creado 2025-07-29; 235 PRs fusionadas y 131 issues abiertas en los últimos 30 días; mediana de fusión de PR ~18 h en una muestra de 10).
- DATO: concentración de tamaño: `api/main.py` 12.562 líneas, `terminal_service.py` 7.370, `memory_service.py` 5.251, `database.py` 4.661.
- INTERPRETACION: la disciplina de proceso (tests, CI, documentación verificada por tests) es muy alta; el riesgo de mantenimiento viene del volumen y del ritmo (167k líneas de código con 794 contribuciones a main y 43,7 % en 3 personas), de los ficheros gigantes y de la dependencia de detección por pantalla del TUI.
- Veredicto: mantenible hoy gracias a la red de tests y CI; caro de entender y de cambiar sin ella. Para un POC que quiera copiar patrones: leer, no heredar.

## 2. Tamaño y forma

Comandos exactos (desde la raíz del repo):
- `git ls-files | wc -l` -> 3.004 ficheros versionados.
- `git ls-files 'src/*.py' | xargs cat | wc -l` -> 317 ficheros, 166.505 líneas.
- `git ls-files 'test/*.py' | xargs cat | wc -l` -> 907 ficheros, 399.459 líneas.
- `git ls-files 'examples/*.py' | xargs cat | wc -l` -> 136 ficheros, 40.008 líneas.
- `git ls-files '*.rs' | xargs cat | wc -l` -> 18 ficheros, 27.454 líneas (`tui/`).
- `git ls-files 'web/*.ts' 'web/*.tsx' | xargs cat | wc -l` -> 33.200 líneas en 122 ficheros (`web/`).
- `git ls-files '*.md' | wc -l` -> 370; `git ls-files 'docs/*.md'` -> 78 ficheros, 30.787 líneas.
- Extensiones: `git ls-files | sed 's/.*\.//' | sort | uniq -c | sort -rn` -> py 1.384, md 370, json 345, txt 216, tsx 142, ts 121.
- Por carpeta de primer nivel (`cut -d/ -f1`): test 1.282, examples 721, src 385, web 142, docs 106, docusaurus 94, cao_mcp_apps 68.

| Parte | Ficheros | Líneas |
|---|---|---|
| Código Python (`src/`) | 317 | 166.505 |
| Tests Python (`test/`) | 907 | 399.459 |
| Ejemplos Python | 136 | 40.008 |
| Rust TUI | 18 | 27.454 |
| Web TS/TSX | 122 | 33.200 |
| Docs `docs/*.md` | 78 | 30.787 |

Los 10 ficheros de código más grandes (`wc -l` sobre `git ls-files 'src/*.py'`):
1. `pkg/api/main.py` 12.562: toda la superficie HTTP, SSE, AG-UI y el lifespan (~123 rutas; `lifespan` en 1743-~2100).
2. `pkg/services/terminal_service.py` 7.370: creación, input, borrado de terminales (142+ definiciones).
3. `pkg/services/memory_service.py` 5.251: memoria wiki/índice.
4. `pkg/clients/database.py` 4.661: 22 tablas ORM y 24 migraciones.
5. `pkg/providers/kimi_cli.py` 3.742.
6. `pkg/mcp_server/server.py` 3.686: herramientas MCP.
7. `pkg/utils/orchestration.py` 3.448: `_handoff_impl`, `_assign_impl`.
8. `pkg/providers/codex.py` 3.149.
9. `pkg/clients/tmux.py` 2.965.
10. `pkg/services/install_service.py` 2.533.
(pkg = `src/cli_agent_orchestrator`). Los dos mayores de test: `test/providers/test_codex_provider_unit.py` 7.348 y `test/services/test_terminal_service_full.py` 7.237.

Complejidad evidente (leída): `run_step` en `api/main.py:6216` mezcla replay, fences de generación, recolectores y mapeo de errores; el docstring de `create_terminal` supera 200 líneas (`terminal_service.py:2004-2200`); `lifespan` arranca ~15 tareas de fondo (`api/main.py:1888-1965`); 29 `# type: ignore` y 234 `noqa` en `src` (`grep -n`). Comentarios: 19.305 líneas que empiezan por `#` en `src` (11,6 % de las líneas), más docstrings largos.
INTERPRETACION: el tamaño por fichero es el principal factor de coste; la documentación inline compensa pero se desactualiza con los cambios.

## 3. Pruebas

Líneas de test / líneas de código por carpeta (medido con `wc -l` de `.py` por directorio):

| Módulo | Código | Test | Ratio |
|---|---|---|---|
| `api` | 12.563 | 39.976 | 3,2 |
| `mcp_server` | 4.497 | 18.950 | 4,2 |
| `security` | 882 | 4.817 | 5,5 |
| `providers` | 27.568 | 57.961 | 2,1 |
| `services` (+ agui, vault) | 67.651 | 137.160 | 2,0 |
| `cli` | 8.277 | 19.338 | 2,3 |
| `decisions` | 4.092 | 11.512 | 2,8 |
| `agent_plugins` | 6.316 | 19.663 | 3,1 |
| `backends` | 3.781 | 7.555 | 2,0 |
| `clients` | 7.904 | 10.910 | 1,4 |
| `utils` | 12.526 | 15.063 | 1,2 |
| `models` | 2.627 | 2.522 | 1,0 |
| `plugins` | 1.549 | 3.491 | 2,3 |
| `graph` | 1.427 | 2.787 | 2,0 |
| `ops_mcp_server` | 1.299 | 2.886 | 2,2 |
| `ext_apps` | 584 | 480 | 0,8 |
| `telemetry` | 454 | 657 | 1,4 |

- Sin directorio de test propio: `skills/` (625 líneas, un script) y `cao_workflow` tiene `test/cao_workflow` con 1.198 líneas para 7 ficheros. No encontré `test/skills`: puede cubrirlo `test/scripts` (4.377 líneas), no comprobado. Mínimos: `ext_apps` (0,8) y `models` (1,0), `utils` (1,2).
- Tipos: unitarios (la gran mayoría), `integration`, `e2e`, `slow` y `live_provider` como marcadores (`pyproject.toml`, `[tool.pytest.ini_options] markers`); e2e en `test/e2e/` (19 ficheros, 5.946 líneas). Tests "de documentación y política" en la raíz de `test/` (p. ej. `test_command_catalog_matches_click.py`, `test_workflows_doc_surface_is_not_stale.py`, `test_cao_contributing_skill_accuracy.py` de 86,8 KB). Frontend: vitest en `cao_mcp_apps` (`ci.yml:309`).
- Modelo real vs simulado: los e2e exigen el CLI del proveedor instalado y autenticado y tmux (`test/e2e/conftest.py:1-12`); los unitarios de proveedores usan fixtures capturadas de salida de CLIs reales (`CONTRIBUTING.md:95-96`) y mocks de tmux (SKILL.md paso 7); 45 ficheros de test usan `mock_cli`/`MockCli` (`grep -rl`). Los pruebas con modelo real se ejecutan en 4 workflows dedicados (claude, codex, kiro, antigravity). `live_provider` solo con `CAO_RUN_LIVE_PROVIDER_TESTS=1` (`pyproject.toml`, markers).
- Skips/xfail: 224 apariciones en `test/` (`grep -n 'pytest.mark.skip\|pytest.skip\|xfail'`).
- INTERPRETACION: base de test muy grande; los tests a nivel de comportamiento del TUI dependen de fixtures de versiones concretas, por lo que envejecen cuando el CLI cambia.

## 4. Integración continua y calidad

Workflows (`git ls-files .github/workflows`, 16): `ci.yml`, `codeql.yml`, `cargo-deny.yml`, `dependency-security.yml`, `gh-pages.yml`, `nightly-python-3-14.yml`, `publish-to-pypi.yml`, `release.yml`, `robotics-factory.yml`, `robotics-transport.yml`, `secret-scan.yml`, `test-antigravity-cli-provider.yml`, `test-claude-code-provider.yml`, `test-codex-provider.yml`, `test-kiro-cli-provider.yml`, `zizmor.yml`.

`ci.yml` (jobs vistos por `name:`): Unit Tests (`ci.yml:32-38`, `--cov-branch` y `--cov-fail-under=80` en `:81-84`, 60 min); Web UI Build; AI-DLC Portfolio Example; CAO MCP Apps (vitest con cobertura); Coverage ratchet (`needs: [test, cao-mcp-apps]`, `:333-335`); CAO MCP Apps E2E (Playwright); Code Quality (`:403-458`: `black --check`, `isort --check-only`, `ruff check src/ test/ examples/` como puerta de código muerto, `python scripts/mypy_ratchet.py`); varios jobs de demos AG-UI; Rust TUI (fmt y tests, con timeout 20 min); Security Scan; Dependency Security; Dependency Review (solo en PR); CodeQL.
- Tipado: no estricto. `mypy.ini` fija `disallow_untyped_defs = False` y silencia códigos de error en `clients/database.py` y `mcp_server/server.py`; la puerta es un ratchet contra `mypy-baseline.json` (2,6 KB; errores existentes pasan, los que suben fallan, `ci.yml:444-458`). Hay además una sección `[tool.mypy]` en `pyproject.toml` (línea 487) junto a `mypy.ini`: dos configuraciones, no comprobé cuál prevalece.
- Cobertura: CI exige 80 % (`ci.yml:84`); `.coverage-baseline.json` fija suelo de 87,0 % de líneas (`python.floor_pct`) y 90,0 % frontend; el de ramas está en `null` (solo registro).
- Pre-commit: opcional (`.pre-commit-config.yaml:1`); gitleaks y dos hooks de trazas "CAO Dojo". No incluye black/isort/ruff.
- Seguridad: CodeQL, zizmor, secret-scan (gitleaks), cargo-deny, Trivy con política en `.github/dependency-security/policy.json`, dependabot (`.github/dependabot.yml`).
- Python en CI: 3.11 y 3.12 según `DEVELOPMENT.md:363`; nightly en 3.14 (`nightly-python-3-14.yml`); clasificadores hasta 3.14 (`pyproject.toml`).
- INTERPRETACION: puertas serias y escalonadas; la cadena de gates es difícil de reproducir en local (muchos jobs, Playwright, Rust, Trivy).

## 5. Documentación

- DATO: README (9,1 KB), README.zh-CN (26,3 KB), `CODEBASE.md` (8,6 KB), `CONTRIBUTING.md` (11 KB), `DEVELOPMENT.md` (18,8 KB), `SECURITY.md` (45,8 KB), `CHANGELOG.md` (194 KB, 3.007 líneas), 78 documentos en `docs/`, Docusaurus (94 ficheros), 16 skills en `skills/` y `pkg/skills/`.
- DATO: quién manda. `CODEBASE.md` ("Documentation maintenance") dice que los documentos focalizados son canónicos y que cualquier cambio de comandos, proveedores, campos de perfil, rutas o encabezados debe actualizar el doc y sus enlaces "en el mismo cambio". Esto se hace cumplir con tests de la raíz de `test/` (`test_command_catalog_counts_are_not_stale.py`, `test_workflows_doc_surface_is_not_stale.py`, tabla de enforcement en `test_enforcement_tables.py` según SKILL.md paso 5). INTERPRETACION: la documentación se trata como código verificable; es una fortaleza poco común.
- Desfases comprobados (doc vs código):
  1. `pkg/skills/cao-provider/SKILL.md` paso 3 ("Add the import and elif branch" en `create_provider`) vs `providers/manager.py:44-59` (diccionario `provider_class` además de la cadena de `create_provider`, `:114`): la guía omite el segundo registro. Es incompleto, no contradictorio.
  2. `DEVELOPMENT.md:363` ("Python 3.11, 3.12 matrix") vs clasificadores 3.13 y 3.14 en `pyproject.toml` y un workflow nocturno 3.14: el doc no menciona 3.13/3.14. Menor.
  3. `pyproject.toml` lista `tomli>=2.0.0; python_version < '3.11'` (en runtime y en `dev`) con `requires-python = ">=3.11"`: marcador muerto, el comentario habla de Python 3.10.
  4. `pyproject.toml` version `2.5.4` vs última release en GitHub `v2.5.3` (2026-10-08): no es desfase, es versión sin publicar.
- No encontré contradicciones entre `CODEBASE.md` y el mapa de paquetes real (`src/` contiene los directorios que lista, incluidos `agent_store/`, `templates/`, `schemas/` sin .py).

## 6. Dependencias

- Runtime Python (`pyproject.toml:[project].dependencies`): 27 entradas (una con marcador muerto, ver arriba): dnspython, fastapi, fastmcp, mcp, pydantic, apscheduler, sqlalchemy, uvicorn[standard], websockets, libtmux, click, python-dotenv, python-frontmatter, requests, rank-bm25, pyte, wcwidth, pyjwt[crypto], cryptography, anyio, jsonschema, jinja2, pyyaml, markdown-it-py, tomli, psutil, tomlkit. Extras: `agui` (1), `otel` (3). Grupo dev: 16.
- Fijación: la mayoría con cota inferior (`>=`); cotas superiores en `fastmcp<5` (monkey-patch de un interno de FastMCP, `ext_apps/sep2133`), `libtmux>=0.51.0,<0.62.1` (rotura documentada en 0.53.1+), `tomlkit<1`, `ag-ui-protocol<1.1`. Varias cotas inferiores existen solo por CVEs transitivos (`cryptography>=50`, `anyio>=4.14.2`, `dnspython>=2.9.2`).
- Lockfile: `uv.lock` con 124 paquetes (`grep -c '^\[\[package\]\]'`), `tui/Cargo.lock`, `package-lock.json` en `web/`, `cao_mcp_apps/`, `docusaurus/` y varios ejemplos, más `bun.lock` y `uv.lock` en ejemplos. Gestor: uv (`uv run` en CI y Makefile), build con hatchling y hook custom para el binario Rust.
- Pesadas o de riesgo: `libtmux` (API interna, ya rota una vez según los comentarios de `pyproject.toml`); `fastmcp` (monkey-patch); cadena de CVEs en `cryptography`; toolchain Rust + cibuildwheel solo para el TUI (macOS arm64 y Linux x86_64; Windows e Intel mac no se construyen por `fcntl` y `cryptography`, comentarios de `pyproject.toml` en `[tool.cibuildwheel]`).
- INTERPRETACION: dependencias sin fijar versión exacta en el paquete, pero con lock y escáner de seguridad bloqueante; riesgo moderado por el acoplamiento a internals de libtmux y fastmcp.

## 7. Convenciones y onboarding

- `CONTRIBUTING.md` (11 KB) y `DEVELOPMENT.md` (18,8 KB) existen; `CODEBASE.md` orienta sobre paquetes. `AGENTS.md`/`CLAUDE.md` en la raíz: no encontrados en lo leído (el `ls -a` raíz no los lista; hay `.agents` con 1 fichero y `.claude-plugin`).
- Guía de estilo: black, isort y ruff en `pyproject.toml` (`[tool.black]`, `[tool.isort]`, `[tool.ruff]`, líneas 401-456), puertas en CI.
- Cómo se añade una pieza: hay una skill dedicada y detallada para proveedores (`cao-provider`, 9 pasos), otra para plugins (`cao-plugin`), workflows (`cao-workflow`), apps MCP; las skills están empaquetadas y regeneradas por `make agent-plugin` y comprobadas por `make check-agent-plugin` (`Makefile`).
- Plantillas de GitHub: solo `ISSUE_TEMPLATE/feature_request.md` y `CODEOWNERS` (`git ls-files .github`); la API de comunidad confirma PR template.
- INTERPRETACION: onboarding bueno para añadir un proveedor; el primer día de un contribuyente nuevo se enfrenta a 16 workflows y ficheros de 12k líneas.

## 8. Salud del proyecto (GitHub, solo GET)

- Repo: 1.619 estrellas, 342 forks, creado 2025-07-29, último push 2026-11-21, no archivado, Apache-2.0 (`gh api repos/awslabs/cli-agent-orchestrator`).
- Últimas 5 releases (`/releases`): v2.5.3 2026-10-08, v2.5.2 2026-10-06, v2.5.1 2026-09-11, v2.5.0 2026-08-28, v2.4.1 2026-08-04 (tras v2.3.0 de 2026-07-12; no aparece v2.4.0). Cadencia: ~2-4 semanas; la última tiene ~6 semanas a fecha de consulta mientras `main` ya dice 2.5.4.
- Issues: 141 abiertas, 297 cerradas (`search/issues`). PRs: 38 abiertas, 793 fusionadas. Últimos 30 días (desde 2026-10-22): 131 issues creadas, 96 cerradas, 235 PRs fusionadas.
- Tiempos (muestra): 10 PRs cerradas más recientes, tiempo de creación a fusión: 0,9 h, 0,9 h, 6,3 h, 7 h, 17,5 h, 18,2 h, 21 h, 21,3 h, 22,5 h, 34 h; mediana ~18 h. Issues cerradas: la muestra devolvió solo 5 issues (el resto eran PRs): 53 h, 31 h, 40 h, 50 h, 35 h; mediana ~40 h. No medí el tiempo hasta la primera respuesta, solo hasta el cierre; la muestra es sesgada hacia lo reciente y cerrado.
- Ficheros de comunidad (`community/profile`): code_of_conduct, contributing, license, pull_request_template, readme; salud 87 %.
- Contribuyentes: 97, con 794 contribuciones en total; los tres primeros 177, 91 y 79 (43,7 % entre los tres). La API cuenta contribuciones a la rama por defecto.

## 9. Factor de bus y deuda

- DATO: marcas TODO/FIXME/HACK/XXX en `src`: TODO 2, resto 0; en test TODO 1 (`grep -wn`). Muy pocas marcas.
- INTERPRETACION: la deuda se declara en prosa, no con etiquetas: p. ej. "LIVE SIBLING of #735, not fixed here" (`agent_step.py:826-831`) y "temporary OpenCode inbox poller... GH #115 tracks replacing" (`api/main.py:1926-1928`). Un recuento por etiqueta infraestima la deuda.
- Código muerto/sin llamador: el CI ejecuta ruff como puerta de código muerto (`ci.yml:441`); hallazgo histórico documentado: "sweep_runs had NO production caller" (`api/main.py:1880-1884`), ya corregido. No ejecuté un análisis de llamadores completo.
- Flags experimentales: backends herdr y herdr-app (`backends/factory.py:84, 100`); Agent Plugins gated por defecto, AG-UI como extra opcional, OTel opcional y desactivado por defecto (`pyproject.toml` extras y comentarios).
- Bus: 43,7 % de las contribuciones en 3 autores; en la muestra de PRs recientes se repiten los mismos autores (`plauzy`, `tedswinyar`, `zichen0116`).

## 10. Riesgos de mantenimiento, ordenados

1. Ficheros gigantes y acoplamiento (`api/main.py` 12.562; `terminal_service.py` 7.370; paquetes con ciclos salvados con imports diferidos: 128 en api->services). Evidencia: recuentos de la sección 2 y análisis de imports.
2. Detección de estado por regex sobre el TUI: cada versión de CLI exige cambios; 27.568 líneas de proveedores y 57.961 de sus tests; fixtures de salida real.
3. Migraciones manuales sin versión de esquema (24 funciones, orden por convención, `database.py:1374-1376`).
4. Ritmo de cambio y concentración: 235 PRs fusionadas en 30 días y 43,7 % de contribuciones en 3 autores.
5. Dependencias a internals (`libtmux<0.62.1`, monkey-patch de `fastmcp`), y una cadena larga de gates de CI que cuesta reproducir.
6. Fallo conocido sin resolver: reutilización de terminal puede dar finalización obsoleta (`agent_step.py:826-831`).
7. Doble configuración de mypy (`mypy.ini` y `[tool.mypy]`), `tomli` con marcador muerto, y guía de proveedor incompleta frente al diccionario de `manager.py`.
8. Solo POSIX (`pyproject.toml` clasificadores).

## 11. Cobertura y confianza

- Contado de forma exhaustiva por comando: ficheros y líneas por tipo y por directorio, TODO/FIXME/HACK, noqa, type: ignore, skips, workflows, paquetes del lock, metadatos de GitHub. Confianza alta en estos números (salvo líneas de test por carpeta: sumadas por directorio, el resto de `test/` raíz no está en la tabla).
- Leído con cita: `ci.yml` (solo líneas filtradas por `grep` de nombres, comandos, timeouts: no lo leí entero), `pyproject.toml` 1-330 y secciones de tool, `.pre-commit-config.yaml`, `mypy.ini`, `.coverage-baseline.json`, `Makefile` 1-60, `test/e2e/conftest.py` 1-25, `DEVELOPMENT.md`/`README.md` por grep.
- No leído: `CONTRIBUTING.md` y `DEVELOPMENT.md` completos, workflows distintos de `ci.yml`, contenido de `docs/`, el resto de `pyproject.toml` (ruff/isort/pytest addopts en detalle), cualquier test individual. La ausencia de `AGENTS.md`/`CLAUDE.md` se basa en el listado de la raíz.
- Ratio test/código mide líneas, no cobertura efectiva; el 87 % de líneas es lo declarado por el suelo del ratchet, no lo medí.
