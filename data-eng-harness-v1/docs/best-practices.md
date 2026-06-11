# Best Practices del arnés

> Principios P1-P14 con trazabilidad al corpus de diseño (reading-pack-harness-engineering-v2/) y a los elementos del arnés data-eng-harness-v1.
> Decisiones de diseño D1-D13 con sus justificaciones.
> Backlog de decisiones abiertas R1-R4.

---

## Principios de diseño

Cada principio incluye: su fundamento en el corpus, los elementos del arnés donde se materializa y la(s) decisión(es) de diseño que lo implementan.

| ID | Principio | Fundamento (artículo/sección) | Elemento(s) del arnés donde aplica |
|---|---|---|---|
| P1 | Separar core reutilizable de configuración de proyecto | AGENT_BOOTSTRAP §6 | `core/` (portable, inmutable por proyecto) vs `adapters/claude-code/` (vendor-specific) vs `project-template/` (por proyecto). Véase D2. |
| P2 | Progressive disclosure: CLAUDE.md ~100 líneas como índice → docs/ | OpenAI Codex §"repository knowledge the system of record" | `project-template/CLAUDE.md` (~80-100 líneas con tabla "dónde encontrar qué"). Profundidad en `project-template/docs/`. |
| P3 | Repo-local truth: conocimiento versionado en el repo | OpenAI Codex §"Agent legibility is the goal" | `core/state-templates/state.json`, `core/state-templates/progress.md`, `tasks/*.json`, `project-template/docs/`. Todo el estado del arnés vive en el repositorio; ningún agente retiene estado privado. Véase D10. |
| P4 | Una tarea por iteración, estado persistido | Huntley (one item per loop); Anthropic harness-design | `core/orchestration/stop-conditions.md` (regla explícita); `adapters/claude-code/skills/harness/SKILL.md` (parada obligatoria al cerrar `[x]`). Véase D5. |
| P5 | Handoff estructurado entre sesiones (reset sin perder estado) | Anthropic harness-design §"context resets" | `core/state-templates/task-contract.json` (audit trail + status persistidos); `core/state-templates/handoff-protocol.md` §4 (reset vs compaction). Véase D7. |
| P6 | Separar quien hace de quien juzga (generador ≠ evaluador) | Anthropic harness-design §"self-evaluation" | `core/contracts/implementador.md` y `core/contracts/evaluador.md` — roles distintos, contextos separados. `adapters/claude-code/agents/evaluador.md` no tiene acceso al historial del implementador. Véase D1, D8. |
| P7 | Contratos verificables de "done" antes de implementar | Anthropic harness-design §"sprint contract" | `core/state-templates/task-contract.json` — campo `acceptance_criteria` escrito por el planificador antes de invocar al implementador. El evaluador usa este campo como fuente de verdad, no el reporte del implementador. |
| P8 | Invariantes mecánicas, no micromanagement | OpenAI Codex §"Enforcing architecture and taste" | `core/sensors/catalog.md` — los 7 sensores verifican invariantes mecánicamente. `project-template/data-conventions.md §4` — tabla de 8 invariantes por sensor. Post-D11, los 7 sensores son especificaciones de categoría parametrizables, instanciadas por proyecto vía `project-template/stack-profile.yml`. Véase D4, D11. |
| P9 | Sensores en capas: fast-feedback vs drift periódico | Maintainability sensors; OpenAI Codex §"Entropy" | `core/sensors/fast-feedback/` (FF-01 a FF-04: in-session y CI) vs `core/sensors/drift-periodic/` (DP-01 a DP-03: periódico). Post-D11, cada categoría de sensor se instancia con la herramienta concreta que declare `project-template/stack-profile.yml` para ese proyecto. Véase D4, D11. |
| P10 | Evals como prueba del arnés: prompt → run → checks → score | Testing skills with evals; Anthropic §"tuning loop" | `core/evals/baseline/eval-01-schema-validation.yaml`, `eval-02-pipeline-generation.yaml`, `eval-03-harness-protocol.yaml`. `core/evals/scorecard-schema.json`. `core/evals/tuning-loop.md`. |
| P11 | Empezar simple, añadir complejidad solo cuando aporte | Anthropic harness-design §"Iterating on the harness" | Baseline de 3 evals conceptuales (no ejecutables) como punto de partida. Mecanismo de tuning por adición (nunca editar checks existentes). `core/evals/tuning-loop.md`. |
| P12 | Herramientas pensadas para agentes (namespacing, tokens eficientes) | Writing effective tools; SWE-agent (ACI) | `adapters/claude-code/agents/*.md` — cada agente declara solo los tools que necesita (evaluador no tiene Write ni Edit; navegador no tiene Write). Feedback de sensores: formato referenciable, accionable, eficiente en tokens. |
| P13 | Human-on-the-loop: el humano mejora el bucle, no ejecuta pasos | OpenAI Codex §"Humans steer" | `core/orchestration/stop-conditions.md` — condiciones de parada y escalada al humano. `core/state-templates/handoff-protocol.md §6` — protocolo de fallback (`on_blocked: escalate_to_human`). Parada obligatoria al cerrar `[x]`. Véase D1, D8. |
| P14 | Handoff como Structured Contract (payload explícito, audit trail, fallback) | Inferensys, Handoff Protocols §2-§6 | `core/state-templates/task-contract.json` — 5 campos booleanos de validación pre-handoff, audit trail append-only, 5 casos de fallback. `core/state-templates/handoff-protocol.md` — tabla de 3 patrones con pros/cons y justificación. Véase D7. |

---

## Decisiones de diseño D1-D13

Tabla de referencia para evaluar propuestas de cambio. Una propuesta que contradiga una decisión de diseño debe explicar por qué el principio subyacente ya no aplica.

| ID | Decisión | Principios que materializa | Elemento(s) del arnés |
|---|---|---|---|
| D1 | Ciclo de 4 agentes (planificador / navegador / implementador / evaluador) para gestionar complejidad multi-sesión de tareas de datos. | P4 (una tarea por iteración), P6 (generador ≠ evaluador), P13 (human-on-the-loop) | `core/orchestration/cycle.md`, `adapters/claude-code/skills/harness/SKILL.md`, `adapters/claude-code/agents/` |
| D2 | Core model-agnostic + adaptador Claude Code aislado y sustituible. | P1 (separar core de config), O3 (Claude Code hoy, model-agnostic mañana) | `core/` (portable), `adapters/claude-code/` (vendor-specific, sustituible sin tocar core) |
| D3 | Estado en repo, markdown/JSON como sistema de registro. | P2 (progressive disclosure), P3 (repo-local truth), O4 (repo-local truth) | `core/state-templates/state.json`, `core/state-templates/progress.md`, `tasks/*.json`, `project-template/docs/` |
| D4 | Invariantes de datos como sensores mecánicos, no prosa en CLAUDE.md. | P8 (invariantes mecánicas), P9 (sensores en capas) | `core/sensors/catalog.md`, `core/sensors/fast-feedback/` (FF-01 a FF-04), `core/sensors/drift-periodic/` (DP-01 a DP-03) |
| D5 | Una tarea por iteración como regla explícita del bucle. | P4 (una tarea por iteración) | `core/orchestration/stop-conditions.md`, `adapters/claude-code/skills/harness/SKILL.md §5` (parada obligatoria) |
| D6 | Empaquetado como plugin/clonable (un único artefacto instalable). | O2 (portabilidad) | `adapters/claude-code/.claude-plugin/plugin.json`, `adapters/claude-code/.claude-plugin/marketplace.json` |
| D7 | Handoff via Structured Contract en lugar de Shared Blackboard o Direct Message Passing. | P5 (handoff estructurado), P14 (Structured Contract) | `core/state-templates/task-contract.json`, `core/state-templates/handoff-protocol.md §1` (tabla de 3 patrones) |
| D8 | Orquestador como hilo principal separado del planificador (el orquestador coordina; el planificador decide). | P6 (separar roles), P13 (human-on-the-loop) | Definido en `core/orchestration/cycle.md`; materializado en `adapters/claude-code/skills/harness/SKILL.md`; documentado en `core/contracts/orchestrator.md` |
| D9 | Protocolo de sesión único con política de checkpoint parametrizable: re-entrada → ciclo de 4 agentes → cierre de sesión; regla "una tarea por sesión"; contrato de checkpoint humano. | P2 (economía de contexto), P4 (una tarea por iteración), P11 (empezar simple), P13 (human-on-the-loop) | `core/orchestration/session-protocol.md`, `core/orchestration/cycle.md` (diagrama de cierre de sesión) |
| D10 | Estratificación spec/estado/progreso: `hard_spec.md` (qué/por qué, casi inmutable) + `state.json` (índice de bloques/tareas con campo de estado) + `progress.md` (notas de sesión append-only). Sustituye a `SPEC.md`. | P2 (economía de contexto), P3 (repo-local truth, "JSON sobre Markdown"), P5 (handoff entre sesiones, asimetría de desechabilidad) | `core/state-templates/state.json`, `core/state-templates/progress.md` |
| D11 | Perfil de stack por proyecto: el core define categorías de sensor parametrizables (lint SQL, lint/typecheck Python, validación de schema/contratos, freshness/nulls/volumetría, tests de pipeline) en lugar de codificar herramientas concretas para el abanico multi-cloud (~40 tecnologías) de los clientes de The Cocktail. Cada proyecto instancia las categorías mediante un perfil de stack declarado en `project-template/`. Python + SQL + Spark/PySpark, con `sqlfluff`, `ruff` + `mypy` y DuckDB, son el soporte de referencia de primera clase del core. | P1 (separar core de configuración de proyecto), P11 (empezar simple, complejidad solo cuando aporte) | `core/sensors/catalog.md` (categorías de sensor parametrizables), `project-template/stack-profile.yml` (perfil de stack por proyecto) |
| D12 | Dos políticas de checkpoint que instancian el parámetro de D9: `validacion-por-tarea` (modo por defecto en v1, el humano revisa el checkpoint al final de cada tarea y cierra sesión con `/clear`) y `full-auto` (encadena tareas sin checkpoint humano intermedio, requiere mecanismo de continuación entre sesiones). En ambas políticas, los checkpoints duros (operaciones DDL/DML destructivas o sobre sistemas de cliente, y modificaciones de `hard_spec.md`) son un invariante del protocolo de sesión, no delegable ni automatizable. | D9 (protocolo de sesión, parámetro de política de checkpoint), P13 (human-on-the-loop) | `core/orchestration/` (protocolo de sesión y condiciones de parada que instancian las dos políticas) |
| D13 | El evaluador no modifica artefactos del implementador, pero EJECUTA: corre pipelines, lanza queries contra el entorno de pruebas, ejecuta tests y sensores fast-feedback, y basa su veredicto en resultados de **ejecución**, no solo en lectura de código/SQL. El contrato de tarea pasa a ser bidireccional: el implementador anexa `execution_summary` (qué hizo, decisiones, desviaciones, artefactos producidos) que el evaluador lee como entrada de contexto. | P6 (separar quien hace de quien juzga, sin diluirlo), P9 (sensores fast-feedback invocables como evidencia de veredicto), P5 (handoff estructurado, contrato bidireccional) | `core/contracts/evaluador.md`, `core/sensors/catalog.md` y `core/sensors/fast-feedback/*.md` (FF-01 a FF-04, invocables por el evaluador), `adapters/claude-code/agents/evaluador.md`, `core/state-templates/task-contract.json` (`execution_summary`) |

> Nota (D11/D12): estas filas cubren la formulación de síntesis de la resolución de R1 (por D11) y
> R2 (por D12), tal como quedan documentadas en `hard_spec.md §6`. La instanciación operativa de
> D11 (categorías de sensor parametrizables y su perfil concreto) ya está propagada a
> `core/sensors/catalog.md` y `project-template/stack-profile.yml`; la instanciación operativa de
> D12 (las dos políticas de checkpoint y los checkpoints duros) ya está propagada a
> `core/orchestration/`. Esta tabla no reescribe esos artefactos, solo enlaza la decisión con su
> ubicación operativa.

> Nota (D13): esta fila cubre la formulación del evaluador-ejecutor, los sensores fast-feedback
> invocables y el contrato bidireccional `execution_summary`. La faceta de D13 sobre git como
> mecanismo de recuperación del bucle (commit solo tras `APTO`, política de reintentos,
> `git log -N` como recuperación) se documenta donde corresponda a su propagación
> (`core/orchestration/`), sin reescribir esta fila.

---

## Backlog de decisiones abiertas (R1-R4)

Decisiones que no están resueltas en la versión actual del arnés y afectan al uso en producción. Se actualiza conforme se resuelven.

| ID | Descripción | Estado | Impacto en bloques |
|---|---|---|---|
| R1 | Stack de datos del equipo sin confirmar (dbt, Airflow, Spark, Snowflake/BigQuery). | Resuelto (2026-06-10) por D11 — Perfil de stack por proyecto (hard_spec.md §6) | Bloquea las implementaciones de referencia de todos los sensores (FF-01 a FF-04, DP-01 a DP-03). Bloquea `project-template/data-conventions.md §5` (tabla de herramientas). |
| R2 | Grado de autonomía / política de aprobación humana por bloque (cuántas paradas obligatorias, quién aprueba qué). | Resuelto (2026-06-10) por D12 — Dos políticas de checkpoint (hard_spec.md §6) | Afecta a `core/orchestration/stop-conditions.md` y a la parada obligatoria de la skill al cerrar `[x]`. |
| R3 | Frontera exacta core/adaptador: vigilar que ningún fichero del adaptador duplique contratos del core, ni que conceptos de Claude Code filtren al core. | Vigilar activamente | Si la frontera se erosiona, O3 (model-agnostic) deja de ser viable sin refactoring. Señal de alerta: un fichero de `adapters/` empieza a declarar criterios de done, condiciones de parada o definiciones de entradas/salidas que ya deberían estar en `core/contracts/`. |
| R4 | Alcance ejecutable de evals: los 3 evals del baseline son conceptuales; sin ejecución real hasta disponer de un proyecto de datos real con dataset, stack y pipeline definidos. | Sin confirmar | `core/evals/baseline/eval-01`, `eval-02`, `eval-03` tienen `status: conceptual`. El tuning loop (`core/evals/tuning-loop.md`) no puede activarse hasta ejecutar un run real. |

---

## Declaración de carpetas prohibidas

> Se verifica que los bloques B1-B8 no han usado ni referenciado las carpetas `harness/` ni `harness-v2-three-agents/` en ningún artefacto del entregable `data-eng-harness-v1/`.

**Método de verificación:** búsqueda recursiva de texto en todos los ficheros `.md`, `.json`, `.yaml` y `.yml` del directorio `data-eng-harness-v1/` usando `grep -r "harness-v2-three-agents\|harness/" .` desde ese directorio.

**Fecha de verificación:** 2026-06-09.

**Resultado:** ningún fichero en `data-eng-harness-v1/` contiene referencias a `harness/` ni a `harness-v2-three-agents/`. La verificación devolvió `No matches found`.

Los directorios `harness/` (arnés v1) y `harness-v2-three-agents/` (arnés v2) existen en el repositorio raíz como implementaciones anteriores independientes. No se han reutilizado ni referenciado durante el diseño de data-eng-harness-v1; el v3 fue diseñado desde cero a partir del corpus de artículos de investigación.
