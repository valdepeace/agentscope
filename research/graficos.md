# Propuesta de gráficos

Los gráficos se dividen en **descriptivos** (se pueden derivar de repositorios/documentación, sin ejecutar modelos) y **experimentales** (requieren corridas medidas). Ningún gráfico descriptivo debe titularse como ranking de calidad.

## Descriptivos: posibles sin llamadas al modelo

| Gráfico | Qué mostraría | Fuente y estado |
|---|---|---|
| Mapa de capacidades/evidencia | Agentes × prompt público, subagentes, A2A, ACP/MCP, compactación y permisos; codificar verificado, parcial, no localizado y no aplicable | `research/matriz.csv` ya tiene las 74 fichas únicas. De ellas, 10 conservan datos heredados sin cita por celda, 59 están pendientes de auditoría y 5 son familias/modelos sin harness confirmado; la matriz aún no respalda un gráfico de capacidades verificadas |
| Mapa de interoperabilidad | Nodos de productos y protocolos; aristas etiquetadas como cliente, servidor, adaptador, wrapper comunitario o integración interna | Fichas y fuentes; separar evidencia oficial y comunitaria con estilos distintos |
| Capas del prompt efectivo | Base del agente, instrucciones de proyecto, modo, herramientas, skills, historial/contexto y bloques dinámicos | Documentación/código de constructores de prompts; es diagrama explicativo, no medición de calidad |
| Flujo de delegación | Principal → implementador → revisor → corrección → verificación, indicando brief/contexto, workspace y retorno | Código/docs de cada agente o skill; dejar vacíos los campos no verificados |
| Disponibilidad de prompts | Recuento de evidencia A–E y tipos de artefacto (principal, subagente, plantilla, extracción comunitaria) | `research/agentes.json` contiene 27 fichas; verificar y normalizar categoría antes de publicar recuentos |
| Lenguajes observados | Frecuencia de lenguajes principales por repositorio en la captura de GitHub Languages | 40 repos públicos/archivados consultados el 2026-10-25. No equivale a lenguaje del binario ni explica por qué se eligió |
| Estrellas por repositorio | Barras o escala logarítmica de estrellas con enlace y fecha de captura | 41 repos el 2026-10-25; popularidad temporal, no calidad. Preferible dejarlo como filtro/tabla y no como ranking protagonista |
| Contraste de delegación (oleada 1) | Cinco columnas: herencia de contexto, ejecución local/remota, retorno/notificación, mensajería interna y evidencia A2A | `research/oleada-01.md`; evidencia de código fijada por SHA en `research/oleada-01-fuentes.csv`. Diagrama descriptivo de mecanismos, no de rendimiento ni de interoperabilidad probada |

## Experimentales: requieren ejecutar y registrar agentes

1. Éxito/regresión por tarea y condición para las ablaciones de prompt/harness/modelo.
2. Coste y tiempo frente a éxito, con distribuciones y presupuesto explícito.
3. Tokens por bloque, compactación y contexto consumido.
4. Llamadas al modelo/herramientas, delegaciones y retries como telemetría de ejecución, no como métrica de calidad independiente.
5. Comparación de una o varias corridas por tarea solo cuando las configuraciones sean equivalentes y se muestren fallos/intervalos.

La cobertura de filas ya está completa, pero no la auditoría de capacidades. Mantener los estados de revisión visibles y no graficar «no verificado» como «no soportado». La tabla web de 74, `agentes.json` de 27 y el registro de prompts tienen alcances distintos; no sumarlos como si fueran un único dataset.

## Primer gráfico experimental (cuando existan datos)

Usar puntos por tarea y condición para mostrar éxito/fallo, con proporción global e intervalo descriptivo; al lado, coste/tiempo y distribución de tokens. Mostrar n=10 tareas y tres repeticiones, fallos y timeouts; no usar barras de ranking aisladas ni ocultar dispersiones. Si no se habilita coste cero, no generar gráfico de resultados sintético: se puede publicar únicamente el diagrama del protocolo.
