# Orquestadores de agentes de código: estudio de nueve repositorios

Lectura de solo lectura (2026-11-21) de nueve orquestadores de agentes: quién decide qué, el LLM o el código. Nada se ejecutó. Complementa el resto de este repositorio con un caso concreto: la arquitectura de orquestación (router, workers, supervisión).

**Confianza.** La lectura línea a línea cubre entre el 0,2 % y el 32 % de cada repositorio. Los recuentos de tamaño y los grafos de imports cubren todos los ficheros; el comportamiento, solo lo leído. Un «no encontrado» significa «no encontrado en lo leído». Las citas `ruta:línea` de las notas iniciales se reverificaron por muestreo (807 únicas, 72 muestreadas, ninguna contradicha); las de las notas de arquitectura y mantenibilidad (`study/arquitectura/`, `study/mantenibilidad/`) **no se han verificado por muestreo**.

## Por dónde empezar

| Quiero | Leer |
|---|---|
| El resumen y las conclusiones | [`STUDY.md`](STUDY.md) (§11 síntesis, §12 profundización) |
| Comparar los nueve en tablas | [`TABLAS.md`](TABLAS.md) |
| Cómo están construidos | [`ARQUITECTURA.md`](ARQUITECTURA.md) |
| Lo caros que son de mantener | [`MANTENIBILIDAD.md`](MANTENIBILIDAD.md) |
| Las puntuaciones y su rúbrica | [`PUNTUACION.md`](PUNTUACION.md) |
| Las fuentes y los huecos | [`REFS.md`](REFS.md) |
| Una presentación de 27 diapositivas | [`presentacion/`](presentacion/) |
| Diagramas interactivos por repo | [`presentacion/flows/`](presentacion/flows/) (HTML; ábrelos en un navegador) |

## Versiones consultadas

Capturadas el 2026-11-21 sobre clones superficiales de un commit. Las estrellas y otras métricas de GitHub cambian con el tiempo y no miden calidad.

| Repo | Commit consultado | Notas |
|---|---|---|
| [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) | `8c32a9dc5d72` | [`study/dsh-superagent-prototype.md`](study/dsh-superagent-prototype.md) |
| [yingpengma/cc-super-agent](https://github.com/yingpengma/cc-super-agent) | `1538e256f2d9` | [`study/cc-super-agent.md`](study/cc-super-agent.md) |
| [LofiSu/dsh-super-agent](https://github.com/LofiSu/dsh-super-agent) | `ca9823e5c096` | [`study/deepseek-super-agent.md`](study/deepseek-super-agent.md) |
| [ketchupz1999/super-super-harness](https://github.com/ketchupz1999/super-super-harness) | `df78e640b50d` | [`study/super-super-harness.md`](study/super-super-harness.md) |
| [bytedance/deer-flow](https://github.com/bytedance/deer-flow) | `900d1b65eb87` | [`study/deer-flow.md`](study/deer-flow.md) |
| [OrchestratorInc/agent-orchestrator](https://github.com/OrchestratorInc/agent-orchestrator) | `080a1a2a5830` | [`study/agent-orchestrator.md`](study/agent-orchestrator.md) |
| [desplega-ai/agent-swarm](https://github.com/desplega-ai/agent-swarm) | `48fa684f1c06` | [`study/agent-swarm.md`](study/agent-swarm.md) |
| [awslabs/cli-agent-orchestrator](https://github.com/awslabs/cli-agent-orchestrator) | `ba53d49bbe45` | [`study/cli-agent-orchestrator.md`](study/cli-agent-orchestrator.md) |
| [gastownhall/gastown](https://github.com/gastownhall/gastown) | `94cc82289d2e` | [`study/gastown.md`](study/gastown.md) |

## Estado y límites conocidos

- Las correcciones que salieron de la segunda pasada (arquitectura y mantenibilidad) están en [`ARQUITECTURA.md`](ARQUITECTURA.md) y **todavía no se han propagado** a `STUDY.md` ni a `TABLAS.md`. Donde difieran, manda la segunda pasada.
- Los diagramas Mermaid de `presentacion/` no se han validado renderizándolos; archify no valida ese tipo.
- Las puntuaciones de [`PUNTUACION.md`](PUNTUACION.md) son juicios con reglas explícitas, no mediciones, sobre siete de los nueve repositorios.
- Los recibos de procedencia de archify (`*.json`) no se incluyen: los HTML de `presentacion/flows/` son una copia sin sus recibos.
