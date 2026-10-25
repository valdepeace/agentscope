---
id: investigacion-agentes-metodologia
title: Benchmarks y protocolo reproducible
---

# 10. Benchmarks y protocolo

SWE-bench Verified tiene 500 tareas verificadas por expertos según [OpenAI](https://openai.com/index/introducing-swe-bench-verified/) y la [guía upstream](https://github.com/SWE-bench/SWE-bench/blob/main/docs/guides/datasets.md). Mide resolver issues según pruebas, no aísla modelo frente a harness. Controlar contaminación, scaffold, contenedor, límites y evaluación.

| Exp. | Hipótesis | Variable independiente | Controles |
|---|---|---|---|
| A | prompts modifican conducta | prompt | snapshot, harness, tools y presupuesto |
| B | harness modifica resultados | harness | mismo modelo; herramientas equivalentes |
| C | modelo modifica resultado | snapshot | harness/prompt/tools fijos |
| D | subagentes cambian calidad/coste | delegación | presupuesto total y tarea fijos |
| E | skills modifican ejecución | skills on/off | versión y criterios iguales |
| F | contexto altera éxito | estrategia/tokens | modelo, harness y tarea fijos |

## Diseño

Congelar repo/base commit, benchmark, contenedor/dependencias, modelo/API ID, prompt hash, agente SHA, skills SHA, herramientas, permisos, sampling/effort y presupuesto. Aislar estado entre corridas; no reintentar silenciosamente. Piloto 3–5 semillas/tarea; confirmar n con análisis de potencia. Completar en repositorios diversos, y añadir tareas curadas para refactor, tests, seguridad, migración, comprensión y diseño.

## Métricas

Task pass rate/pass@1; tests externos aprobados; severidad de regresiones por revisión ciega; cumplimiento de requisitos; tiempo; tokens input/output/cache; coste real/estimado; llamadas LLM/tools; delegaciones/retries; máximo contexto; violaciones de permisos. Reportar distribución/intervalos y fallos/timeout; separar pruebas del agente de grader externo.

Riesgos: muestreo, alias mutables, contaminación, tests defectuosos, herramientas no equivalentes, límites distintos. Igual coste no equivale a igual cómputo; igual tokens no equivale a igual información.
