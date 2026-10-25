# Estado de investigación

Fecha de corte documental: 2026-10-25. Estado editable a mano; “completado” aquí significa pasada inicial, no cierre del criterio de publicación.

| Fase | Estado | Evidencia/pendiente |
|---|---|---|
| 1. Descubrimiento de agentes/fuentes | pasada inicial | Catálogo de 74 agentes/productos; no representa literalmente todos los editores existentes |
| 2. Versiones y commits | parcial | Se anotaron versiones para artefactos medidos; faltan SHA fijados para las estadísticas de repositorios |
| 3. Prompts/plantillas | parcial | Varios repos con ruta de código; falta extraer/hash, versionar y muestrear prompts dinámicos |
| 4. Arquitecturas harness | parcial | Mapa de rutas de 12 sistemas; falta recorrido de tool loop, retries, sandbox y sesión |
| 5. Catálogo de modelos | parcial | Familias estructuradas, snapshot exacto/precios/fechas casi siempre pendientes |
| 6. Comparativa técnica | parcial | GitHub Languages consultado para 40 repos; estrellas capturadas para 41 repos el 2026-10-25; razones de elección y otros artefactos pendientes |
| 7. Experimentos | piloto preinscrito; 0 ejecuciones | Ablación de prompts: 10 tareas × 2 condiciones × 3 repeticiones. Hay CLIs locales, pero no se confirmó configuración/coste de inferencia; Ollama sin directorio local de modelos. No hubo llamadas al modelo |
| 8. Redacción | borrador | 13 documentos de investigación; pareja EN/ES del artículo en rama `draft/ai-coding-agents-2026-10-25` de `valdepeace/my-website` |
| 9. Revisión crítica | parcial | `research/matriz.csv` ahora contiene las 74 fichas únicas: 10 filas heredadas con cita por celda pendiente, 59 por auditar y 5 entradas de modelo/familia sin harness confirmado. La matriz A2A/tecnología de la web tiene su propia cobertura y no se debe mezclar con esta tabla |
| 10. GitHub Pages | preparado en local | Workflow `.github/workflows/deploy-pages.yml`; este workspace aún no tiene remoto ni se ha publicado |

## Links verificados en esta pasada

Lectura web directa satisfactoria de las páginas/rutas de MiniMax Code, Trae Agent, ZCode, Xiaomi MiMo Code, Qwen Code prompt, OpenHands SDK, Codex prompt, Gemini CLI prompt docs, OpenCode agents docs y Superpowers SDD. No se hizo petición HTTP por cada enlace del inventario. La ruta OpenCode `system.ts` produjo una respuesta 404 en una apertura, aunque una búsqueda del propio GitHub devolvió el archivo de esa ruta en `dev`; debe revisarse al fijar SHA.

El 2026-10-25 se consultó `GET /repos/{owner}/{repo}/languages` para los 40 repos abiertos/archivados enlazados en las fichas; las tablas muestran los cuatro lenguajes con más bytes detectados. Se leyeron `stargazerCount` de páginas GitHub para los 41 enlaces a repositorio, incluido Seed como ecosistema/modelo. Las cifras de estrellas son una captura temporal, no una medida de calidad.

El repositorio `valdepeace/my-website` recibió en una rama de borrador los archivos EN/ES con el mismo nombre para revisión; no se creó PR ni se publicó el artículo. Los borradores locales preexistentes del blog se preservaron.

Qwen Code: la nota oficial `docs/design/2026-10-05-session-multi-agent.md` marca la colaboración A2A por sesiones como implementada en PR #13467/#13583, detrás de `experimental.agentCollaboration` (desactivada por defecto). En la nota oficial `docs/design/2026-09-09-a2a-frozen-contract.md`, Qwen declara binding JSON-RPC, polling (sin streaming/push en la primera versión), y que la interoperabilidad cruzada no está demostrada. La web local tiene 74 filas renderizadas, pero el objeto `a2aEvidence` contiene 12 entradas específicas; las otras usan el fallback genérico “no verificado”. Hermes, CodeBuddy y OpenClaw son casos positivos descubiertos fuera de esas 12 entradas, así que el mapa A2A del sitio omite evidencia localizada. La actualización oficial del 2026-10-15 anuncia Claude Code y Codex como compañeros. Además, Hermes Agent documenta un plugin first-party A2A v1.0, cliente y servidor, y PR #86660 confirma que el registro de tools ya se corrigió para CLI/TUI. CodeBuddy Code CLI documenta cliente y servidor A2A por stdio desde v2.143 y mejoras de coordinación en v2.145. OpenClaw documenta canal A2A v1.0 bidireccional, pero es un gateway general y su primera versión tiene límites de estado/streaming/seguridad. Gemini CLI documenta cliente A2A mediante Agent Cards y tiene `packages/a2a-server`; el issue oficial #14535 describe el servidor como A2A-like/no plenamente conforme, y el manifest revisado usa `@a2a-js/sdk` 0.3.11. Conformidad de servidor con la versión actual e interoperabilidad quedan sin demostrar. Separar el endpoint A2A externo de la coordinación interna por sesión/Host: comparten el paraguas experimental, pero no son el mismo transporte. Esto corrige la afirmación anterior de que Gemini era el único caso confirmado. No implica soporte A2A nativo en Claude/Codex; el coordinador Qwen usa adaptadores. No fijé SHA del repositorio Qwen en esta pasada.

## Riesgos de estado

### Oleada de análisis profundo 1 (2026-10-25)

Selección: Codex CLI, Gemini CLI, OpenCode, Crush y Qwen Code. Se inspeccionaron clones shallow del código oficial con SHA fijado; resumen y fuentes primarias pinneadas: [`oleada-01.md`](oleada-01.md) y [`oleada-01-fuentes.csv`](oleada-01-fuentes.csv). Hallazgo de esta pasada: los cinco ofrecen delegación con semánticas diferentes; solo Qwen mostró servidor A2A externo en el código revisado y detrás de opt-in; Gemini mostró cliente remoto A2A. No se probó interop ni se ejecutaron modelos. Los extractos de prompts son literales y tipados por procedencia; no son prompts efectivos completos.

- Fecha/modelos/precios son temporales; la fecha del sistema/proyecto difiere de los datos almacenados solo si se reusa este estudio más adelante.
- Links de ramas activas pueden moverse o desaparecer.
- El artículo dice borrador y los ficheros de investigación repiten contenido por separado para facilitar lectura/publicación; un cierre final debería eliminar duplicación o establecer una única fuente canónica.
- GitHub no ofrece una creación de repositorio mediante los conectores disponibles; `valdepeace/agentscope` no existe en la cuenta consultada y el navegador web de GitHub no estaba autenticado. El workflow de Pages aún no ha ejecutado.
