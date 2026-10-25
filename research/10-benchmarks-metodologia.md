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

## Piloto propuesto: ablación de instrucciones A

Objetivo inicial: obtener una primera comparación legible y repetible sin presentarla como ranking general. Mantener fijo un harness, modelo/API ID, snapshot de repositorio, herramientas, permisos, presupuesto y criterios de evaluación; variar únicamente el bloque de instrucciones bajo estudio. Registrar el hash del prompt serializado efectivo, no solo el texto base.

| Elemento | Preinscripción |
|---|---|
| Unidad | Una tarea independiente en un worktree limpio desde el mismo commit base |
| Dataset piloto | 10 tareas seleccionadas y publicadas antes de ejecutar: 4 bugs, 2 cambios pequeños, 2 tests/regresiones, 1 refactor acotado y 1 revisión de seguridad |
| Condiciones | Prompt/harness de referencia y una variante de prompt; mismas instrucciones de repo, herramientas y límites |
| Repeticiones | 3 por tarea y condición como piloto exploratorio (n=60 ejecuciones); no tratarlo como estimación definitiva ni alegar potencia estadística |
| Evaluación | Tests ocultos o criterios de revisión definidos antes de correr; revisión ciega del diff para calidad y regresiones |
| Registro | ID/tarea, commit, agente/modelo exactos, prompt hash, configuración, llamadas, tokens/coste, duración, salida, pruebas, fallos y reintentos |
| Exclusiones | No descartar timeouts ni fallos; reportarlos como resultado. Un retry solo cuenta si estaba predefinido y queda registrado |

Ejecución en dos etapas: primero una prueba de instrumentación con 2 tareas × 2 condiciones (4 sesiones limpias, una por celda) para comprobar logging, grader y reset del repositorio. No usarla para comparar calidad. Solo si funciona y se conoce el coste, pasar al piloto exploratorio de 60 ejecuciones descrito en la tabla.

El piloto no se ejecuta hasta congelar las tareas y verificar licencia, contaminación, dependencias y grader. No usar API de pago sin un límite explícito de gasto; primero comprobar opción local realmente disponible y coste cero. No descargar modelos grandes ni lanzar una batería extensa automáticamente.

## Estado de instrumentación (2026-10-25)

Comprobaciones locales sin invocar inferencia: `codex-cli 0.164.0`, `Claude Code 2.1.310`, `OpenCode 1.17.13` y `Qwen Code 0.25.1` están disponibles en PATH. No se encontró un directorio local de modelos de Ollama en la ruta comprobada. `mastracode --version` no devolvió versión en la secuencia observada. Esto no confirma qué cuentas/API están configuradas, qué modelo usaría cada CLI ni si tienen cuota gratuita.
