# Gas Town (gastownhall/gastown) - eje MANTENIBILIDAD

Solo lectura. DATO = recuento o lectura citada; INTERPRETACIÓN = juicio mío. Rutas relativas a la raíz del repo. Commit leído 94cc82289d2e (2026-11-12). GitHub consultado solo con GET el 2026-11-21.

## 1. Resumen y veredicto

- DATO: ~249 k líneas Go de código, ~226 k de tests, 5.842 funciones Test; CI con tests con -race, lint, integración con Dolt, e2e nocturno, Windows build+vet.
- DATO: 65 % de las contribuciones del top-100 son de un autor; en los últimos 100 commits (jul-nov 2026) 3 personas hacen 95.
- DATO: 47 ficheros de más de 1.000 líneas, funciones de hasta 1.313 líneas, seis paquetes sin ningún importador, 532 issues+PRs abiertas, última release 2026-06-06 con commits hasta 2026-11-14.
- INTERPRETACIÓN: base bien probada y con CI seria, pero el tamaño, la concentración en `internal/cmd` y el código muerto la hacen cara de mantener; cadencia de releases rota desde junio.
- Veredicto: mantenible por un equipo pequeño que ya la conoce; difícil de entrar. Riesgo medio-alto.

## 2. Tamaño y forma

Comandos (bash, raíz del repo):
- `find . -name '*.go' -not -name '*_test.go' | xargs cat | wc -l` y `... | wc -l`
- `find . -name '*_test.go' | xargs cat | wc -l`
- `find . -name '*.md' | xargs cat | wc -l`
- `find . -name '*.go' -not -name '*_test.go' | xargs wc -l | sort -rn`
- funciones largas: `awk` sobre `^func` y `^}` en ficheros no-test.

| Tipo | Ficheros | Líneas |
|---|---|---|
| Go código | 663 | 248.970 |
| Go tests | 569 | 225.826 |
| Markdown (117 ficheros; docs/ = 68 ficheros, 24.598 líneas) | 117 | 33.208 |
| Otros ficheros (total 1.565): toml 57, sh 38, json 26, yml 16, yaml 14, tmpl 13, py 4, js 6 | - | no contado |

Diez ficheros src más grandes (líneas, función):
1. internal/doltserver/doltserver.go 4.646: ciclo de vida del servidor Dolt (Start de 320 líneas en 1683).
2. internal/tmux/tmux.go 4.454: envoltorio tmux.
3. internal/witness/handlers.go 3.577: detección de zombis, nuke, handlers.
4. internal/git/git.go 3.573.
5. internal/polecat/manager.go 3.150.
6. internal/daemon/daemon.go 3.098: Run 391 líneas.
7. internal/config/loader.go 2.930.
8. internal/cmd/convoy.go 2.906.
9. internal/cmd/done.go 2.748: `runDone` 1.313 líneas (done.go:668).
10. internal/refinery/engineer.go 2.657.

Complejidad evidente (DATO): `runDone` 1.313 líneas con 15+ `goto notifyWitness` (done.go:1118-1796); `runSling` 945 (sling.go:200); `AddRig` 612 (internal/rig/manager.go:318); 48 funciones >200 líneas y 234 >100; 18 ficheros >2.000 líneas. `internal/cmd` = 253 ficheros y 103.691 líneas (41,6 % del src). Tests mayores: config/loader_test.go 5.872, beads_test.go 5.471, doltserver_test.go 5.155, cmd/sling_test.go 4.952.

## 3. Pruebas

DATO: razón líneas test/src global 0,91; ficheros 0,86. 5.842 funciones Test (1.428 en internal/cmd). 587 `t.Skip`, 20 `testing.Short()`.

Razón test/src por módulo (≥1.000 líneas src): tui 0,41; protocol 0,57; telemetry 0,61; **cmd 0,66**; util 0,69; reaper 0,75; web 0,78; witness 0,90; tmux 0,90; daemon 0,97; doctor 0,99; mail 1,01; doltserver 1,02; git 1,04; refinery 1,09; session 1,10; polecat 1,12; beads 1,18; rig 1,22; deacon 1,25; config 1,55; formula 2,02; proxy 2,24.

Sin ningún test: internal/health (177 líneas) y cmd/gt-proxy-client (137). Bajos entre pequeños: agentlog 0,27, constants 0,34, shell 0,40, deps 0,46, mayor 0,58.

Tipos (DATO, build tags en `*_test.go`): 17 `integration`, 1 `integration && scheduler_integration`, 1 `e2e`, 1 `browser` (internal/web/browser_e2e_test.go usa go-rod), 7 `!windows`, 2 `darwin`. Integración usa contenedor Dolt (testcontainers) y `bd` real: CONTRIBUTING.md:171-197. E2E en contenedor: Dockerfile.e2e y .github/workflows/e2e.yml:20-23. Un test AST impone política de subprocesos: internal/beads/bd_subprocess_policy_test.go:16.

Modelo real o simulado: en Go no encontré tests que llamen a un modelo real; la referencia a ANTHROPIC_API_KEY en tests solo aparece en internal/config/agents_test.go y cost_tier_test.go (configuración, no llamadas). Los tests de comportamiento de agentes están fuera de CI, en `gt-model-eval/` (promptfoo, necesita ANTHROPIC_API_KEY real: gt-model-eval/README.md). INTERPRETACIÓN: el comportamiento LLM de Witness/Refinery/Deacon, que es el corazón, se evalúa solo con esa suite manual.

## 4. Integración continua y calidad

Workflows locales (11, en .github/workflows/) más 5 dinámicos (Copilot, Dependabot, conflict-marker-check existe en GitHub pero no en el clon: 16 totales por API).

| Workflow | Qué comprueba |
|---|---|
| ci.yml (push/PR a main) | rechaza `replace` en go.mod (15-36) y .beads/issues.jsonl (39-60); Test con `-race -short -timeout=10m -coverprofile` (97-100) + Codecov (110-125, fail_ci_if_error false); Lint golangci-lint v2.11.4 (142-146); Integración `-tags=integration` ./internal/cmd/... y 4 bloques de scheduler (186-213) |
| windows-ci.yml | build + `go build ./...` + `go vet ./...` en MSYS2 (41-49); sin tests |
| nightly-integration.yml, e2e.yml | diario 06:00 UTC; e2e en Docker |
| release.yml | `make check-version-tag`, GoReleaser, SBOM, Homebrew, npm |
| block-internal-prs, triage-label, remove-needs-*, close-stale-needs, update-nix-flake | gestión de issues/PRs |

Gates: el job Test es bloqueante salvo la subida a Codecov. Cobertura: codecov.yml:14-23 exige proyecto "auto" con umbral 5 % y parche 60 % con umbral 10 %; no hay umbral fijo ni cifra de cobertura leída (n/d).

Linters: .golangci.yml:7-14 habilita solo errcheck, gosec, misspell, unconvert, unparam, con `tests: false` (línea 5) y 15 exclusiones de gosec (G304, G306, G302, G404, G104, G204, G115, G117, G201, G702-705) casi todas en `internal/` (:38-103). `go vet` solo corre en el job de Windows (windows-ci.yml:49); no hay `gofmt` ni `staticcheck` en CI (grep de "gofmt|go vet" en workflows, Makefile y .githooks solo da esa línea). CONTRIBUTING.md:88-93 pide gofmt y go vet. 178 `nolint` en src.

Tipado: Go estático; no hay modo estricto adicional. Pre-commit: no hay .pre-commit ni .editorconfig; hay .githooks/pre-push (bloquea pushes a ramas no permitidas, guarda HEAD-mismatch) con test (pre-push_test.sh). Acciones fijadas por SHA (ci.yml:20, 66...) y Renovate con automerge de minor/patch estables (renovate.json). Debilidades (DATO): `go install gotest.tools/gotestsum@latest` sin fijar (ci.yml:92, 179); Dolt instalado con `curl | sudo bash` (ci.yml:86); Dockerfile:66 ejecuta el script de install de beads desde la rama main.

## 5. Documentación

DATO: 117 .md; docs/ con 68 ficheros y 24.598 líneas (design/, concepts/, guides/, runtimes/, research/). README, AGENTS.md (185 líneas), CONTRIBUTING (212), RELEASING (190), SECURITY, CHANGELOG (Keep a Changelog), glosario. 

INTERPRETACIÓN: hay mucho diseño y propuestas, y el código manda. Desfases comprobados (doc vs código):
1. docs/design/mail-protocol.md:12-30 (POLECAT_DONE es mail con handler del Witness) vs internal/cmd/done.go:1933 (nudge) y internal/beads/beads_agent.go:54,621 ("instead of POLECAT_DONE mail").
2. docs/design/architecture.md:229-233 (batch-then-bisect "Blocked by Phase 1") vs internal/refinery/batch.go:58,211,485 (algoritmo implementado, sin llamador).
3. AGENTS.md:3 remite a "CLAUDE.md" y no existe CLAUDE.md en la raíz (`ls CLAUDE.md` falla).
4. docs/design/architecture.md:189 usa `~/.gt/.beads` (22 menciones de `~/.gt` frente a 177 de `~/gt` en docs y README).
5. SECURITY.md:56 solo soporta "0.1.x"; internal/cmd/version.go:17 es "1.2.1".
6. internal/cmd/sling_dispatch.go:74-86 lista 12 pasos con "Create Dolt branch"; el código tiene 11 pasos sin ese.
7. internal/config/agents.go:~62 dice que no hay switch por proveedor fuera del registro; hay en internal/util/orphan.go:412, internal/cmd/down.go:781, internal/doctor/orphan_check.go:427, agentlog/event.go:54.
8. Makefile:40-44 `desktop-build` y `desktop-run` apuntan a ./cmd/gt-desktop, que no existe (cmd/ solo tiene gt, gt-proxy-client, gt-proxy-server).
9. Puerto Dolt: docs/design/dolt-storage.md:13 y doltserver.go:140 dicen 3307; internal/daemon/dolt.go:87 por defecto 3306 (se sobrescribe con endpoint gestionado, dolt.go:170-186). Desfase parcial.
Ajustes sin desfase: INSTALLING.md:64-107 pide Go 1.26.11 y CI usa 1.26.11 (ci.yml:73); go.mod:3 declara 1.26.2 y toolchain 1.26.11.
Docs de seguridad de dependencias: docs/SECURITY-DEPENDENCY-EXCEPTIONS.md (revisado 2026-10-24) documenta 2 avisos altos aceptados en gt-model-eval.

## 6. Dependencias

DATO: go.mod: 30 requires directos y 218 indirectos (248 en total), go.sum 1.724 líneas, sin `replace`/`exclude` (CI lo impide, ci.yml:15-36). Gestor: Go modules con lockfile go.sum; flake.nix con vendorHash (68 líneas); npm: npm-package/package.json (wrapper binario) y gt-model-eval (devDependency promptfoo 0.125.0 exacta, con package-lock).
Versiones: fijadas por go.mod (MVS) y por Renovate; steveyegge/beads v1.0.5 (go.mod:20). 
Pesadas o de riesgo (INTERPRETACIÓN sobre DATO): 12 entradas dolthub, AWS SDK v2, cloud.google.com/* y gRPC como indirectas (go.mod:42-72), arrastradas por beads; testcontainers y go-rod en el go.mod principal aunque solo se usan en tests; 7 módulos OpenTelemetry; ICU4C necesario por CGo (ci.yml:77, Makefile:24-33); Dolt 2.6.0 mínimo y datos no portables entre versiones (CHANGELOG [Unreleased]). Dependencia estrecha de un binario externo `bd` (repo aparte) y de `dolt`.

## 7. Convenciones y onboarding

- CONTRIBUTING.md: flujo, estilo (gofmt/go vet, "funciones pequeñas"), filosofía ZFC y Bitter Lesson (:95-138), mensajes de commit (:150-154), guardas de test de integración (:171-197).
- Plantilla de PR (.github/PULL_REQUEST_TEMPLATE.md) y 2 plantillas de issue (bug_report.md, feature_request.md).
- AGENTS.md (instrucciones de agente: beads y comunicación) y skills en .claude/skills (crew-commit, ghi-list, pr-list, pr-sheriff) y .cursor/skills. 
- Cómo se añade una pieza: agente = entrada en builtinPresets (agents.go:232) o JSON de usuario; plugin = plugin.md + run.sh; fórmula = TOML en internal/formula/formulas/; rol = TOML + plantilla + constantes en 3 sitios. No hay guía única de "cómo añadir X" en CONTRIBUTING (no encontrado en lo leído); hay docs/agent-provider-integration.md y docs/design/plugin-system.md (no leídos).
- INTERPRETACIÓN: vocabulario propio (polecat, deacon, wisp, molecule) y 89 variables `GT_*` distintas elevan la curva.

## 8. Salud del proyecto vía GitHub (GET)

- Repo gastownhall/gastown, creado 2025-12-16, último push 2026-11-14, 18.862 estrellas, 1.704 forks, licencia MIT, no archivado.
- Releases (14 en total): v1.2.1 2026-06-06, v1.2.0 2026-05-30, v1.1.0 2026-05-07, v1.0.1 2026-04-25, v1.0.0 2026-04-03 (anteriores: v0.13.0 2026-03-29). INTERPRETACIÓN: ritmo de 1-4 semanas hasta junio; sin release desde hace ~5,5 meses (hoy 2026-11-21) con CHANGELOG [Unreleased] ya poblado.
- Issues: 397 abiertas, 882 cerradas. PRs: 135 abiertas, 1.612 fusionadas. (API search; "open_issues_count" del repo = 532 = issues+PRs).
- Respuesta: muestra de 10 issues (#4623-#4679, jul-ago 2026, cada 3.ª del listado): 9 sin respuesta humana en los primeros 10 comentarios, 1 con respuesta a las 195,8 h; 1 cerrada a las 0,9 h; 9 siguen abiertas. Las 10 más recientes (#4957-#4987, nov) tienen 0 a 19 comentarios, todas abiertas. Muestra de PRs cerrados (#4367-#4465, jul 2026): fusiones en 0,3 h, 0,3 h, 1,3 h, 2,7 h, 21,4 h, 122,6 h; 3 cerrados sin fusionar (125 h, 164 h, y bot 0 h). INTERPRETACIÓN: PRs de los mantenedores se fusionan en horas; las issues externas esperan mucho. La muestra es pequeña y sesgada.
- Ficheros de comunidad (community/profile): health 87 %; CODE_OF_CONDUCT, CONTRIBUTING, LICENSE, README, PR template presentes; issue_template figura null en la API aunque existe .github/ISSUE_TEMPLATE/ (2 .md). Enlace de documentación del repo apunta a `tree/master/docs` aunque la rama por defecto es main.
- Contribuidores (primeros 100): 7.390 contribuciones; autor 1: 4.831 (65,4 %), autor 2 470 (6,4 %), autor 3 367 (5,0 %), autor 4 208. Últimos 100 commits (2026-07-07 a 2026-11-12): autor 4 50, autor 3 31, autor 2 14.
- Nota: el módulo y copyright son de steveyegge (go.mod:1, LICENSE) pero el repo está en la organización gastownhall.

## 9. Factor de bus y deuda

- Marcas (grep -E 'TODO|FIXME|HACK|XXX' en .go): 12 líneas, de ellas 6 TODO reales en src (molecule_step.go:365, sling.go:698, synthesis.go:367, daemon.go:2188, rig/manager.go:1126, y la referencia sling_dispatch.go:63), 3 en tests y 3 falsos positivos (XXXX en formatos); FIXME 0, HACK 0; en md/toml/sh/yml/tmpl otras 9. INTERPRETACIÓN: densidad de marcas muy baja; la deuda no se anota. 
- Indicadores alternativos: 429 líneas con "legacy" o "backwards compat" en src, 9 "Deprecated:", 178 `nolint`, 14 comparaciones por nombre de agente fuera del registro.
- Código muerto o sin llamador (grep de importadores y de símbolos, excluyendo tests y comentarios; salvedad: no descarto uso por reflexión, plantillas o `go:linkname`): paquetes internal/protocol (1.374 líneas src + 789 test), connection (689), github (421), keepalive (137), mq (58), agent/state.go (62); funciones witness.HandlePolecatDone, HandlePolecatDoneFromBead, HandleLifecycleShutdown, HandleHelp, HandleSwarmStart, EscalateRecoveryNeeded; refinery ProcessMRInfo, ProcessBatch, AssembleBatch (cadena doMerge/bisectBatch solo desde ellos); deacon.LoadStuckConfig; Makefile desktop-*. Suma de paquetes sin importador: ~2.800 líneas src.
- Flags experimentales: SECURITY.md:40 declara el proyecto experimental (según nota previa); `GT_DOLT_STATS_ENABLED`; `--experimental-acp` de Gemini (config/agents.go:1215); 89 variables `GT_*`.
- Factor de bus: 1 autor concentra 65 % histórico; hoy 3 personas hacen casi todos los commits. Muchos comentarios dependen de IDs de issues internos (gt-xxxx) que solo viven en la base de beads del proyecto.

## 10. Riesgos de mantenimiento (ordenados)

1. **Concentración de lógica y tamaño**: `internal/cmd` 103 k líneas, runDone 1.313 líneas, 47 ficheros >1.000. Evidencia: sección 2.
2. **Cobertura de comportamiento LLM fuera de CI**: la lógica de patrulla es prompt+fórmula; solo gt-model-eval (manual, API real) la mide (gt-model-eval/README.md). Cambios de modelo pueden romper sin que CI lo vea.
3. **Código muerto y duplicado** que engaña al lector: motor de merge Go sin llamador mientras la doc habla de él; dos implementaciones de despacho (sling.go:698). Evidencia: sección 9.
4. **Bus factor y cadencia**: 65 % de commits históricos de un autor; sin release en 5,5 meses; issues externas sin respuesta en la muestra.
5. **Lint débil**: 5 linters, tests excluidos, 15 exclusiones de gosec, sin gofmt/staticcheck en CI, vet solo en Windows. Evidencia: .golangci.yml, ci.yml, windows-ci.yml:49.
6. **Dependencias externas acopladas y sin fijar**: `bd`, Dolt, tmux, ICU; `gotestsum@latest`; instalación por curl|bash (ci.yml:86, Dockerfile:66); 218 indirectas.
7. **Desfases doc/código** (sección 5) y constantes duplicadas (Role en 3 sitios).
8. **Windows nativo**: solo build y vet en CI, tests no corren (windows-ci.yml:1-49).

## 11. Cobertura y confianza

- Leí con detalle: ci.yml, windows-ci.yml (parcial), .golangci.yml, codecov.yml, renovate.json, Makefile 1-200 aprox., .githooks/pre-push 1-80, CONTRIBUTING.md, SECURITY.md:52-62, CHANGELOG cabecera, go.mod (primeras 80 líneas), package.json x2, docs/SECURITY-DEPENDENCY-EXCEPTIONS.md 1-30, fragmentos de código citados en el eje de arquitectura. Resto de workflows: greps de nombres y disparadores.
- Recuentos (ficheros, líneas, TODO, tags, importadores) cubren el 100 % de los ficheros; la lectura continua cubre ~1,5 % de líneas Go.
- GitHub: cifras de search API y muestras pequeñas (10 issues, 10 PRs); contribuidores limitados a los 100 primeros por la API (el porcentaje es sobre ese subconjunto). Sin historial git local (clon superficial), por eso no hay churn ni edad de ficheros. No ejecuté nada: la cobertura real de tests, flakiness y tiempo de CI son n/d.
- Confianza: alta en tamaños, estructura y CI; media en código muerto (búsqueda estática por nombre); baja en tiempos de respuesta (muestra pequeña).
