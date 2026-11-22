# Presentación: nueve orquestadores de agentes de código

Estudio de solo lectura (2026-11-21). Cinco repos a fondo (dsh, cc-super-agent, agent-orchestrator, cli-agent-orchestrator, gastown); los otros cuatro solo en las comparativas.

## Qué hay aquí

| Qué | Dónde |
|---|---|
| Guion de 27 diapositivas (fuente única) | `deck/content.md` |
| Página del deck (HTML, Mermaid nativo, modo lectura y modo presentar) | `deck/index.html` |
| Diagramas archify por repo (arquitectura y secuencia) y árbol de los nueve | `flows/*.html` (11, con sus recibos `*.json`) |
| Bloques Mermaid de las notas | `mermaid/*.mmd` (21; 14 usados en el deck) |
| Scripts de comprobación | `scripts/check-flows.mjs`, `scripts/extract-mermaid.mjs` |

## Repositorios

[`deer-flow`](https://github.com/bytedance/deer-flow) · [`dsh-superagent-prototype`](https://github.com/deepseek-ai/dsh-superagent-prototype) · [`gastown`](https://github.com/gastownhall/gastown) · [`agent-orchestrator`](https://github.com/OrchestratorInc/agent-orchestrator) · [`cli-agent-orchestrator`](https://github.com/awslabs/cli-agent-orchestrator) · [`agent-swarm`](https://github.com/desplega-ai/agent-swarm) · [`LofiSu/dsh-super-agent`](https://github.com/LofiSu/dsh-super-agent) · [`cc-super-agent`](https://github.com/yingpengma/cc-super-agent) · [`super-super-harness`](https://github.com/ketchupz1999/super-super-harness)

## Estado de verificación

- Los 11 HTML de archify pasan `archify check` en su sitio.
- Citas `ruta:línea` de las notas: 807 únicas reverificadas; 72 muestreadas (67 respaldadas, 4 desplazadas y corregidas, 1 no concluyente, 0 contradichas).
- **Mermaid sin validar al renderizar.** archify no valida flowchart, sequence, state ni mindmap, y en esta sesión no hubo forma de abrir el artefacto publicado para ver los 14 bloques dibujados. Solo pasaron la comprobación estructural de `extract-mermaid.mjs`. Falta abrir el enlace del deck y mirarlos; si alguno falla, se corrige en `deck/content.md` y se regenera `deck/index.html`.
- Cobertura baja (3 al 20 %) en agent-orchestrator, agent-swarm, cli-agent-orchestrator y gastown: ❓ significa «no encontrado en lo leído».
- No hay exportación a PPTX; PDF solo desde la impresión del navegador.

## Pendiente

- Residuos de archify sin limpiar (carpetas `.archify\\*`, incluida `review-2\\` de cc-super-agent).
