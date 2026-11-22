# Mantenibilidad de los siete: comparativa

Segunda pasada (2026-11-21), solo lectura. Notas por repo en `study/mantenibilidad/`. Solo se usan señales contadas o leídas, separando DATO de INTERPRETACIÓN. Los clones son superficiales (1 commit): factor de bus y cadencia salen de la API de GitHub (solo GET), no del historial local. Los tests se cuentan por líneas y ficheros; no se ejecutaron ni miden cobertura.

## Comparativa

| Repo | Código (líneas) | Tests (líneas) | Ratio test/código | CI y calidad | Salud en GitHub | Factor de bus | Veredicto |
|---|---:|---:|---:|---|---|---|---|
| [`dsh-superagent-prototype`](study/mantenibilidad/dsh-superagent-prototype.md) | 15.587 | 14.960 | 0,96 | **sin CI**, sin linter ni formateador; `strict` en TS | sin releases; issues y PRs desactivados; 1 commit | 1 | calidad interna alta; riesgo externo alto |
| [`LofiSu/dsh-super-agent`](study/mantenibilidad/LofiSu-dsh-super-agent.md) | 35.281 | 43.079 | 1,22 | lint de invariantes, typecheck estricto, tests offline; sin linter de estilo | sin releases; 91 de las 100 últimas PR fusionadas (mediana 0,23 h) | **1 por política**: `owner-only.yml` cierra issues y PRs ajenos | mantenible por su autor, frágil para terceros |
| [`deer-flow`](study/mantenibilidad/deer-flow.md) | ~275.000 (Python) | 610.254 | 2,2 | 21 workflows, ruff, pre-commit, `uv.lock`; sin mypy ni cobertura | 3 releases; 530 PRs abiertas; 913 abiertas en 30 días | concentrado (top 4: 610/285/258/203 commits) | mantenible por quien ya lo conoce; caro de asimilar |
| [`agent-orchestrator`](study/mantenibilidad/agent-orchestrator.md) | ~1,65 M (código y tests, Go y TS) | ratio ~1 en casi todo el backend | ~1 | 23 workflows, `-race`, golangci-lint v2, deriva de código generado; sin lint de TS | releases diarias; 987 PRs fusionadas en 30 días; 592 abiertas | 3 personas = 60 % de los últimos 100 commits | media-alta por proceso, media-baja por tamaño de módulos |
| [`agent-swarm`](study/mantenibilidad/agent-swarm.md) | 362.665 (`src/`) | 435.404 | 1,2 | 26 jobs en `merge-gate.yml`, 8 shards, e2e de 198 escenarios (LLM simulado), gitleaks | 1.050 PRs fusionadas en 30 días; 12 releases en 12 días | bajo: dos identidades dominan y una es un bot | mantenible mientras los gates sigan vivos; frágil para un recién llegado |
| [`cli-agent-orchestrator`](study/mantenibilidad/cli-agent-orchestrator.md) | 166.505 | 399.459 | 2,4 | 16 workflows; cobertura (exige 80 %, suelo 87 %); ratchet de mypy; dead-code | 235 PRs fusionadas en 30 días; mediana de fusión ~18 h (muestra de 10) | top 3 = 43,7 % | mantenible hoy gracias a tests y CI; «leer, no heredar» |
| [`gastown`](study/mantenibilidad/gastown.md) | ~249.000 (Go) | ~226.000 (5.842 `Test`) | 0,9 | `-race`, lint (5 linters, tests excluidos), integración con Dolt, e2e nocturno; `go vet` solo en el job de Windows | sin release desde 2026-06-06; 532 issues y PRs abiertas; 9 de 10 issues de la muestra sin respuesta humana | 65 % del top 100 de un autor | riesgo medio-alto |

## Lo que se repite

1. **Los tests no son el problema.** Seis de los siete tienen tantas líneas de test como de código o más. Los e2e dependen casi siempre de un modelo simulado; el e2e con agente real no corre en CI (AO, deer-flow en opt-in).
2. **El tamaño de los ficheros sí lo es.** `db.ts` 14.753, `api/main.py` 12.562, `manager.go` 8.382, `runtime.ts` 3.231; gastown tiene 47 ficheros de más de 1.000 líneas.
3. **El ritmo supera al equipo.** AO, swarm, deer-flow y CAO fusionan cientos de PRs al mes con pocas personas, y en deer-flow solo 1 de 10 PRs de la muestra tuvo comentario humano. En swarm los mayores contribuidores son bots. Es un proyecto sostenido por automatización.
4. **Los dos DSH no tienen comunidad.** Uno por abandono (1 commit, issues desactivados) y otro por política (cierra lo ajeno). Calidad interna alta, ninguna vía para un tercero.
5. **La deuda se declara en prosa, no en marcas.** Casi no hay `TODO` (1 en dsh, 0 en LofiSu, 2 en AO, 2 en CAO), pero sí 234 `noqa` en CAO y desfases doc/código comprobados en gastown, LofiSu y swarm.
6. **gastown es el único con la cadencia rota**: commits hasta 2026-11-14 pero sin release desde junio.

## Límites

- Cobertura de lectura baja (0,2 a 32 %, ver `ARQUITECTURA.md`). Las cifras de tamaño y tests son recuentos; una cifra de ficheros de test no es cobertura.
- Las medianas de respuesta salen de muestras de 10 issues o PRs: orientativas.
- Un dato de la propia fuente conviene vigilar: «PRs fusionadas por 30 días» mezcla bots y personas.
