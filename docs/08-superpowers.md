---
id: investigacion-agentes-superpowers
title: Superpowers
---

# 08. Superpowers

[obra/superpowers](https://github.com/obra/superpowers) es un framework de skills/procedimientos (MIT según repo), no un LLM ni harness. Instruye al agente anfitrión sobre brainstorming, diseño, planes, TDD, depuración, revisión y cierre. El host sigue aportando modelo, ejecución de tools y mecanismo de subagentes.

La skill [subagent-driven-development](https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md) prescribe implementador fresco por tarea, revisión de spec/calidad después, ciclo de corrección y revisión global al finalizar. Pide worktree por plan y un ledger. El prompt de implementador ([plantilla](https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/implementer-prompt.md)) consume un brief/contexto preparado por controller; no hereda todo el historial automáticamente por el diseño descrito.

## Ciclo

1. Principal descompone un plan en tareas independientes.
2. Controller prepara brief e interfaces/contexto pertinente.
3. Implementador trabaja, prueba y reporta en workspace aislado.
4. Revisor recibe requisitos, reporte y diff.
5. Controller adjudica y despacha correcciones acotadas.
6. Revisión amplia comprueba integración y evidencia final.

Worktrees son mecanismo Git usado por instrucciones de skill; Superpowers no los implementa como runtime universal. Skills tampoco crean aislamiento, paralelismo o permisos: dependen del host y de su fidelidad al protocolo.

| Host | Comparación que falta cerrar |
|---|---|
| Claude Code | Superpowers se instala como skills; host dispone de tools/subagents |
| Codex | comparar dispatch nativo, approvals y AGENTS.md contra mismos briefs |
| OpenCode | agents y permisos son configurables; comprobar import/activación de skills |
| Crush | inspeccionar prompt/templates/tools y compatibilidad de skills |
| Kimi Code | documenta coder/explore/plan aislados; comprobar skill protocol |
| Qwen Code | subagents con prompt/model/tools configurables; comparar contexto y dispatch |

Releases/manifest varían. Enlazar y resumir; si se copia contenido MIT, preservar atribución/licencia.
