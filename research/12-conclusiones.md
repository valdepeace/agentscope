---
id: investigacion-agentes-conclusiones
title: Conclusiones y límites
---

# 12. Conclusiones

1. No hay fracción universal de comportamiento atribuible al modelo, prompt y harness; requiere ablaciones controladas.
2. El harness controla capacidades observables: tools, ejecución, permisos, estado, contexto y finalización.
3. Una instrucción de verificar no demuestra que se ejecutaron pruebas.
4. Modelo, chat template y prompt son componentes distintos.
5. Skills describen procedimientos; runtime del host aporta ejecución, aislamiento y tools.
6. Leaderboards mezclan variables y no son comparación causal por sí mismos.
7. Agente/modelo y oficial/comunitario son ejes distintos.
8. La colaboración entre CLIs sigue fragmentada: los equipos, subagentes, transferencia de contexto, permisos y retorno de resultados suelen depender del runtime concreto. En las fuentes revisadas hay rutas A2A oficiales confirmadas para Gemini CLI y Qwen Code; la cobertura restante no permite afirmar que otros carezcan de soporte.
9. A2A, ACP y MCP resuelven conexiones distintas. ACP enlaza editor y agente; MCP, agente y herramientas/datos; no sustituyen A2A.

La frase «cada uno hace la guerra por su cuenta» describe la falta de interoperabilidad por defecto, no una incompatibilidad absoluta. Gemini CLI documenta el consumo de agentes A2A remotos. Qwen Code documenta colaboración A2A por sesiones detrás de una opción experimental desactivada por defecto; sus adaptadores para Claude Code y Codex no demuestran que esas CLI implementen A2A nativamente.

«No localizado» no significa «no existe»: muchas filas siguen con evidencia A2A pendiente. Las 74 fichas son agentes/productos, no 74 repositorios; la muestra de código documentada comprende 40 repositorios para lenguajes y 41 enlaces con estrellas. El catálogo tampoco representa todos los editores existentes. Faltan SHA, rutas internas, modelos/precios y confirmaciones oficiales; mantener estos límites al publicar.
