# Puntuación de arquitectura y mantenibilidad

Sobre los siete repos de `ARQUITECTURA.md` y `MANTENIBILIDAD.md` (2026-11-21). Cada criterio va de 0 a 5; la nota de cada eje es la media de sus criterios puntuados, multiplicada por 2 (escala 0 a 10). **Una puntuación es un juicio con reglas explícitas, no una medida.** Las reglas están abajo para que se puedan discutir y cambiar.

**Confianza.** La lectura línea a línea de estos repos es del ~0,2 % al ~32 % (ver `ARQUITECTURA.md`). Las notas que salen de recuentos (M1, M2) son fiables. Las que dependen de leer comportamiento (A1 a A5) son tan buenas como lo leído. Un `n/c` es «no comprobado»: ese criterio se deja fuera de la media, no se rellena.

## Resultado

| # | Repo | Origen | Arquitectura | Mantenibilidad | **Global** | Global con tamaño ×2 |
|---:|---|---|---:|---:|---:|---:|
| 1 | `deer-flow` | Empresa (ByteDance) | 7,2 | 6,4 | **6,8** | 6,5 |
| 2 | `dsh-superagent-prototype` | Oficial (DeepSeek), prototipo | 7,5 | 5,2 | **6,4** | 6,8 |
| 3 | `agent-swarm` | Organización (desplega-ai) | 6 | 6 | **6** | 5,5 |
| 4 | `cli-agent-orchestrator` | Empresa (AWS Labs) | 4,5 | 7,2 | **5,9** | 5,3 |
| 5 | `LofiSu/dsh-super-agent` | Particular | 5,5 | 5,2 | **5,4** | 5,3 |
| 6 | `agent-orchestrator` | Organización, producto comercial | 5,2 | 4,8 | **5** | 4,6 |
| 7 | `gastown` | Organización, de un solo autor en la práctica | 5,2 | 4,4 | **4,8** | 4,6 |

La última columna repite el cálculo contando el tamaño de ficheros (M1) **doble** en mantenibilidad, para ver si el orden depende de esa elección.

**Origen.** Con empresa o equipo detrás (deer-flow, agent-orchestrator, agent-swarm, cli-agent-orchestrator) salen el mejor CI y los mejores tests, y también los ficheros más grandes. Los particulares salen con código pequeño y coherente, pero con un solo autor. Son siete repos, de escalas muy distintas: es una pista, no una regla.

## Rúbrica

### Mecánica (misma regla para todos)

- **M1 Tamaño de ficheros.** Por el fichero de código no generado más grande: ≤1.000 líneas = 5; ≤2.500 = 4; ≤4.000 = 3; ≤6.000 = 2; ≤10.000 = 1; más = 0. Se resta 1 (mínimo 0) si hay alguna función de más de 1.000 líneas.
- **M2 Tests.** Por la proporción de líneas de test sobre líneas de código: ≥2 = 5; ≥1,2 = 4; ≥0,9 = 3; ≥0,5 = 2; algo = 1; nada = 0. Es proporción de líneas, no cobertura. Todos los e2e usan modelo simulado.

### Por checklist

- **M3 CI y puertas.** Un punto por cada uno: hay CI; los tests corren en CI de forma seria (shards, `-race`, integración); linter y formateador; tipado estricto, ratchet o cobertura con umbral; puertas de seguridad o de deriva de código generado.

### De juicio (con la evidencia citada abajo)

- **A1 Fronteras.** 5 = declaradas y hechas cumplir por test o CI, sin fugas vistas; 3 = declaradas y casi cumplidas, con fugas documentadas; 1 = declaradas e incumplidas; 0 = no hay.
- **A2 Acoplamiento.** Ciclos entre módulos y dependencias mutuas. 5 = sin ciclos vistos; 2 = muchos ciclos o dependencias circulares disimuladas con imports diferidos.
- **A3 Extensibilidad.** Cuántos ficheros hay que tocar para añadir un harness, proveedor o rol. 5 = una entrada de registro; 1 = decenas de ficheros.
- **A4 Estado y consistencia.** Dónde vive el estado, transacciones, migraciones, un solo escritor.
- **A5 Resiliencia.** Qué pasa ante un fallo a mitad de tarea (reinicio, atasco, timeout).
- **M4 Documentación fiel.** 5 = verificada contra el código por CI; 3 = abundante con algún desfase; 1 = desfases graves.
- **M5 Salud del proyecto.** Cadencia de releases, personas activas que responden, concentración de commits.

## Notas por criterio

| Repo | A1 Fronteras | A2 Acoplamiento | A3 Extensibilidad | A4 Estado | A5 Resiliencia | **Arquitectura /10** |
|---|---:|---:|---:|---:|---:|---:|
| `dsh-superagent-prototype` | 4 | 3 | n/c | 5 | 3 | **7,5** |
| `LofiSu/dsh-super-agent` | 3 | 3 | n/c | 3 | 2 | **5,5** |
| `deer-flow` | 4 | 2 | 4 | 4 | 4 | **7,2** |
| `agent-orchestrator` | 2 | 4 | 1 | 3 | 3 | **5,2** |
| `agent-swarm` | 3 | 2 | 2 | 4 | 4 | **6** |
| `cli-agent-orchestrator` | 2 | 2 | n/c | 2 | 3 | **4,5** |
| `gastown` | 3 | 3 | 1 | 3 | 3 | **5,2** |

| Repo | M1 Tamaño | M2 Tests | M3 CI | M4 Docs | M5 Salud | **Mantenibilidad /10** |
|---|---:|---:|---:|---:|---:|---:|
| `dsh-superagent-prototype` | 5 | 3 | 0 | 4 | 1 | **5,2** |
| `LofiSu/dsh-super-agent` | 2 | 4 | 3 | 3 | 1 | **5,2** |
| `deer-flow` | 1 | 5 | 4 | 3 | 3 | **6,4** |
| `agent-orchestrator` | 0 | 3 | 4 | 2 | 3 | **4,8** |
| `agent-swarm` | 0 | 4 | 5 | 3 | 3 | **6** |
| `cli-agent-orchestrator` | 0 | 5 | 5 | 4 | 4 | **7,2** |
| `gastown` | 1 | 3 | 3 | 2 | 2 | **4,4** |

## Justificación (evidencia de las notas por repo)

| Repo | Lo que sostiene las notas |
|---|---|
| `dsh-superagent-prototype` | A1/A2: `src/core` no importa nada de fuera (comprobado por script); en `src/app` hay un componente fuertemente conexo de ~10 ficheros. A4: una transacción síncrona por comando y cola de efectos durable. A3 n/c. M1: el mayor fichero de `src` tiene 673 líneas. M3: sin `.github`. M5: 1 commit, issues y PRs desactivados. |
| `LofiSu/dsh-super-agent` | A1: el `Engine` no conoce el arnés (hooks de `Host`), pero hay ciclos core↔channels. A4: sin savepoints, las transacciones anidadas se aplanan (`src/store/sqlite.ts:57-75`); el riesgo de escrituras parciales es deducción mía. M1: `runtime.ts` 3.231 y `Runtime.create` ~1.560 líneas. M5: 375 de 375 commits de un autor y un workflow que cierra lo ajeno. |
| `deer-flow` | A1: un test AST impide que el harness importe `app`; dentro hay 23 pares con import mutuo. A3: paquete `extension-api` con versión fijada. A5: leases, heartbeat y reconciliación; los subagentes `task` no se reanudan. M1: 4.749 líneas el mayor y `run_agent` ~1.600. M2: ratio 2,2. M5: 1 de 10 PRs de la muestra con comentario humano. |
| `agent-orchestrator` | A1: las reglas documentadas se incumplen en puntos concretos (`adapters/scm/multi/provider.go:14`, `storage/.../shell_terminal_store.go:10`, `observe/usage/ingestor.go:17`). A2: el compilador de Go impide ciclos de paquetes. A3: ~35 ficheros más un CHECK de SQLite a mano (`db.go:2303-2557`), pese a que `registry.go:1-5` dice «una edición». M1: `manager.go` 8.382 y `daemon.Run` ~1.550. |
| `agent-swarm` | A1: script de CI y dependency-cruiser, pero la regla transitiva está en `warn` (`.dependency-cruiser.cjs:29-34`). A2: muchos ciclos (`be` con casi todo). A4: la API es la única dueña de SQLite y el reclamo es un `UPDATE` atómico. A5: barrido de heartbeat cada 90 s; la clase C solo registra y no hay tope por tarea. M1: `db.ts` 14.753 y `runAgent` ~1.960. M3: 26 jobs, gitleaks, deriva. |
| `cli-agent-orchestrator` | A1/A2: la capa `CODEBASE.md:62-71` se cruza (8 ficheros mencionan tmux directo; 128 imports diferidos desde `api/main.py`). A4: 24 migraciones a mano sin Alembic ni versión de esquema. A5: `run_agent_step` bien resuelto, con un fallo reconocido (`agent_step.py:826-831`). M1: `api/main.py` 12.562. M2: ratio 2,4. M3: cobertura exigida al 80 %, ratchet de mypy. |
| `gastown` | A1/A2: sin SCC entre paquetes de primer nivel, pero el daemon llama a `gt` por subproceso para evitar un ciclo y hay tres caminos de acceso a Dolt (subproceso `bd`, store en proceso y SQL directo). A3: añadir un rol toca ~80 ficheros. M1: 18 ficheros de más de 2.000 líneas y `runDone` de 1.313. M4: `mail-protocol.md` desfasado y un `desktop-build` que apunta a un directorio que no existe. M5: 65 % del top 100 de un autor, sin release desde 2026-06-06. |

## Lectura y cautelas

1. **Lo que pasa con los ficheros gigantes.** Tenías razón: con la regla M1, `agent-swarm` (14.753), `cli-agent-orchestrator` (12.562) y `agent-orchestrator` (8.382 y una función de ~1.550) sacan 0 en tamaño. Es el criterio donde peor quedan, aunque sus tests y su CI sean los mejores.
2. **El orden depende de cuánto pese el tamaño.** Con pesos iguales sale deer-flow primero; contando el tamaño doble, el prototipo oficial de DeepSeek pasa delante. El prototipo puntúa alto en gran parte **porque es pequeño** (15.587 líneas de código, sin CI y con un commit). No es comparable con proyectos de 160.000 a 1,6 millones de líneas. Antes de leer «el mejor» hay que ver la escala.
3. **Tres repos tienen criterios de arquitectura sin puntuar** (`dsh`, `LofiSu`, `cli-agent-orchestrator`, en A3): no se midió cuántos ficheros cuesta añadir una pieza. Su media usa solo los criterios que tienen evidencia.
4. **Los umbrales de M1 y M2 son una decisión mía.** Con otros cortes, los números cambian; el orden relativo del tamaño (swarm, CAO y AO al final) no.
5. **No puntúa lo que hace cada uno** (capacidades, seguridad, aislamiento): eso está en `TABLAS.md`. Esta nota solo habla de cómo está construido y de lo caro que será mantenerlo.
