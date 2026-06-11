# DESIGN.md — Documento de diseño del arnés harness-v3-ai-generated

> Explica y justifica cada elemento del arnés con trazabilidad a los principios P1–P14 y al corpus.
> Se completa progresivamente en los bloques B2–B8.

## Design Principles

El arnés se ha diseñado siguiendo un conjunto de principios de diseño que gobiernan las decisiones
de arquitectura descritas en el resto de este documento. Estos principios se han derivado del
análisis de los papers y artículos más recientes sobre harness engineering —entre ellos los
informes de Anthropic sobre diseño de harnesses y agentes de larga duración, las prácticas de
Codex de OpenAI, el patrón "one item per loop" de Geoffrey Huntley y los protocolos de handoff
entre agentes especializados de Inferensys—, del estudio de otros arneses de referencia
disponibles en repositorios públicos, y de las iteraciones y la experiencia propia acumulada en
versiones previas de este arnés. No son funcionalidades concretas, sino ideas
conceptuales que orientan cómo se reparte la responsabilidad entre agentes, cómo se persiste el
estado, cómo se valida el trabajo y cómo se mantiene la calidad de los datos a lo largo del tiempo.
Cada principio se expresa en cuatro dimensiones: el principio en sí, sus implicaciones prácticas
para el diseño del arnés, dónde se materializa dentro de este arnés, y la fuente que lo fundamenta.

| Principio de diseño | Implicaciones | Materialización | Fundamento |
|---|---|---|---|
| **P1 — Separación core / configuración de proyecto** | El runtime, los contratos de agente y el patrón de orquestación quedan aislados de cualquier decisión específica de cliente (stack, capas de datos, convenciones de naming). La frontera es bidireccional: el core no asume nada sobre un proyecto concreto y un proyecto no necesita tocar el core para adaptarse. Toda evolución del core beneficia a todos los proyectos sin riesgo de romper instancias ya desplegadas. | Separación `core/` vs `adapters/claude-code/` vs `project-template/`; `project-template/CLAUDE.md` y `data-conventions.md` como capa de configuración por proyecto. | AGENT_BOOTSTRAP §6; README "Portable core" / "Project config" |
| **P2 — Progressive disclosure** | El punto de entrada de cada agente es un índice corto que orienta, no un manual exhaustivo; el detalle vive en documentos referenciados que se cargan solo cuando la tarea los necesita. Evita saturar el contexto del agente con información irrelevante para el paso actual. | `project-template/CLAUDE.md` (~80-100 líneas) como índice hacia `docs/`; estratificación `hard_spec.md` (casi inmutable) / `state.json` (índice de estado) / `progress.md` (notas de sesión) (D10). | OpenAI Codex §"repository knowledge the system of record"; README "Progressive disclosure" |
| **P3 — Repo-local truth** | Todo conocimiento, decisión o estado que un agente necesite existe como artefacto versionado y legible en el repositorio; lo que no está en el repo no existe para el agente. El repositorio es autosuficiente para reconstruir el contexto tras un reset de sesión. | `core/state-templates/` (`state.json`, `progress.md`, `task-contract.json`); `tasks/*.json`; `project-template/docs/dudas.md`. | OpenAI Codex §"Agent legibility is the goal"; README "Repo-local truth" |
| **P4 — Iteración atómica con estado persistido** | Cada vuelta del ciclo de agentes corresponde a una unidad de trabajo verificable y deja el repositorio en un estado consistente al terminar. El avance se mide por el estado persistido, no por la narrativa del agente, lo que impide declarar una tarea "hecha" sin evidencia. | Regla "una tarea por sesión" de `core/orchestration/session-protocol.md` (D9); `core/orchestration/stop-conditions.md`; campo de estado por tarea en `state.json`. | Huntley (one item per loop); Anthropic long-running agents §"feature_list + progress" |
| **P5 — Handoff persistente entre sesiones** | El estado de una tarea (plan, progreso, contratos en curso) sobrevive a la pérdida de la ventana de contexto. Una sesión nueva reconstruye "dónde se quedó el trabajo" leyendo únicamente artefactos del repositorio, sin depender de la memoria de la sesión anterior. | `core/orchestration/session-protocol.md` (secuencia de re-entrada, D9); `core/state-templates/state.json` y `progress.md`; distinción reset vs compaction (§3.5 de este documento). | Anthropic harness-design §"context resets"; effective-harnesses |
| **P6 — Separación generador / evaluador** | El agente que produce un artefacto y el que verifica su corrección son entidades distintas, sin compartir contexto de ejecución. El evaluador se calibra para ser escéptico por defecto, partiendo de que la autoevaluación del generador no es fiable. | Roles `implementador` y `evaluador` en `adapters/claude-code/agents/`; `core/contracts/implementador.md` y `core/contracts/evaluador.md`; D13 (el evaluador ejecuta sin modificar artefactos). | Anthropic harness-design §"self-evaluation" |
| **P7 — Contrato verificable de "done"** | Antes de iniciar el trabajo existe un conjunto de criterios de aceptación expresados en términos comprobables —mecánicamente o por rúbrica con umbral— compartido por quien implementa y quien evalúa, de modo que ambos miden "hecho" con el mismo criterio. | Bloque `acceptance_criteria` de `core/state-templates/task-contract.json`; checks deterministas y rubric-based de `core/evals/`. | Anthropic harness-design §"sprint contract" |
| **P8 — Invariantes mecánicas** | Las reglas de arquitectura y calidad que deben cumplirse siempre se codifican como comprobaciones automatizadas que devuelven feedback accionable, en lugar de depender de que un agente las recuerde o un humano las revise manualmente. El resto del espacio de diseño queda libre. | `core/sensors/catalog.md` y especificaciones de `core/sensors/fast-feedback/`. | OpenAI Codex §"Enforcing architecture and taste" |
| **P9 — Sensores estratificados (fast-feedback vs drift)** | Las comprobaciones se dividen según su cadencia y coste: las que corren en cada sesión o pipeline (schema, lint, tests) frente a las que detectan degradación acumulada con cadencia propia (lineage, catálogo, principios de datos). Cada capa tiene su propio formato de feedback y su propio disparador. | `core/sensors/fast-feedback/` (FF-01 a FF-04) y `core/sensors/drift-periodic/` (DP-01 a DP-03). | Maintainability sensors; OpenAI Codex §"Entropy and garbage collection" |
| **P10 — Evals como prueba del arnés** | El propio arnés se valida con un conjunto reducido de escenarios (prompt → ejecución capturada → checks → score) que cubren outcome, proceso, estilo y eficiencia. Los fallos detectados retroalimentan el diseño, generando nuevos checks o sensores (tuning loop). | `core/evals/` (`eval-01` a `eval-03`, `scorecard-schema.json`, `tuning-loop.md`). | Testing skills with evals; Anthropic harness-design §"tuning loop" |
| **P11 — Complejidad incremental justificada** | Cada componente del arnés se justifica como respuesta a una limitación concreta del modelo o del proceso. Si esa limitación deja de existir (p. ej. al cambiar de modelo), el componente es candidato a simplificación o retirada sin romper el resto del sistema. | Justificación del número de agentes del ciclo (D1); perfiles de stack en lugar de implementaciones codificadas por herramienta (D11); evals conceptuales pendientes de activación (R4). | Anthropic harness-design §"Iterating on the harness"; Building Effective Agents |
| **P12 — Tooling orientado a agentes** | Las herramientas expuestas a los agentes tienen un espacio de nombres claro y devuelven solo la información relevante de forma compacta, en lugar de envolver una API de propósito general sin adaptar. El formato de salida es consumible sin post-procesado costoso. | Declaración de `tools` por agente en `adapters/claude-code/agents/*.md`; formato de feedback de sensores (referenciable, accionable, eficiente en tokens, estructurado) en `core/sensors/`. | Writing effective tools for agents; SWE-agent (ACI) |
| **P13 — Human-on-the-loop** | El humano interviene en puntos de control definidos —revisión de veredicto, diff y estado— en lugar de supervisar cada acción. Estos puntos de control son también la señal que permitirá decidir en el futuro qué pasos pueden delegarse. | Contrato de checkpoint de `core/orchestration/session-protocol.md` (D9); políticas de checkpoint `validacion-por-tarea` y `full-auto` (D12). | OpenAI Codex §"Humans steer"; README "Human-on-the-loop" |
| **P14 — Handoff como contrato estructurado** | La transferencia de trabajo entre agentes especializados se realiza mediante un artefacto persistente con payload explícito —contexto, criterios de éxito, validación pre-handoff, audit trail inmutable y fallback— en lugar de un mensaje informal que se pierde al cerrar la sesión. | `core/state-templates/task-contract.json`; `core/state-templates/handoff-protocol.md`. | Inferensys, "Handoff Protocols Between Specialized Agents" §2-6 |

---

## 1. Patrón de orquestación (core/orchestration/)

El ciclo de 4 agentes se define en `core/orchestration/cycle.md` (diagrama + reglas) y `core/orchestration/stop-conditions.md` (condiciones de parada detalladas). El ciclo se ejecuta dentro del protocolo de sesión único de `core/orchestration/session-protocol.md` (D9, ver §1.1).

Principios aplicados: P4 (una tarea por iteración), P5 (handoff entre sesiones / reset vs compaction), P6 (separación generador/evaluador), P13 (human-on-the-loop). El patrón es model-agnostic: no menciona Claude Code ni ningún proveedor; la implementación concreta vive en `adapters/claude-code/` (D2, O3).

### 1.1 Protocolo de sesión único (D9)

`core/orchestration/session-protocol.md` define el protocolo de sesión que envuelve al ciclo de 4 agentes de `cycle.md`. El arnés define **un único protocolo de sesión**, no dos modos separados ("interactivo" vs "autónomo"): la política de checkpoint —quién relanza la sesión y qué se le muestra— es un **parámetro** de ese protocolo, no un diseño alternativo (D9, hard_spec.md §3.1.1 y §5).

**Regla "una tarea por sesión".** Cada sesión ejecuta como máximo una tarea completa del backlog: un ciclo planificador → implementador → evaluador → planificador, y termina en la secuencia de cierre. El bucle no se repite dentro de la misma sesión; es el humano quien, tras revisar el checkpoint de cierre, decide arrancar una sesión fresca para la siguiente tarea (P4, P2).

**Secuencia de re-entrada (arranque de cada sesión fresca):**

1. Leer `state.json` — índice de bloques/tareas y su campo de estado actual.
2. Identificar la tarea activa (`in_progress`) o, si no hay ninguna, la siguiente `pending`.
3. Leer la última entrada de `progress.md` — qué se hizo en la sesión anterior, bugs conocidos, siguiente paso anotado.
4. Inspeccionar el historial de control de versiones (p. ej. `git log -N`) para confirmar que el repositorio refleja lo que dicen `state.json`/`progress.md`.
5. Verificar el baseline: el repositorio está en un estado consistente, sin cambios sin confirmar de una sesión anterior que no cerró correctamente.
6. Invocar al `planificador` con ese contexto reconstruido.

**Secuencia de cierre (fin de cada sesión):**

1. El `evaluador` emite veredicto (`APTO`/`NO APTO` + defectos).
2. **Commit solo tras `APTO`.** Si `NO APTO`, no se confirma ningún cambio y la tarea permanece `in_progress` para la siguiente sesión, con los defectos anotados en `progress.md`.
3. El `planificador` actualiza el campo de estado de la tarea (y, si corresponde, del bloque) en `state.json`, y añade una entrada nueva al final de `progress.md`.
4. **Checkpoint humano** (ver "Contrato del checkpoint" más abajo).
5. **Limpieza de contexto** (p. ej. `/clear` en Claude Code): el humano limpia el contexto de la sesión y, si decide continuar, arranca una sesión fresca que repite la secuencia de re-entrada.

**Contrato del checkpoint.** Antes de cerrar la sesión, el humano revisa como mínimo: el veredicto del evaluador (`APTO`/`NO APTO` y defectos), el diff/artefactos producidos en la sesión, y el estado actualizado (`state.json` con el campo correcto + la entrada nueva en `progress.md`). Esta lista es exactamente lo que un futuro runner autónomo debería automatizar, degradar a muestreo, o mantener como punto de escalación obligatoria, según el riesgo de la tarea (D9 y R2; granularidad de delegación abierta).

> Nota: si el contexto se satura **dentro de una sesión activa** (sin haber llegado al cierre de la tarea), el orquestador compacta el contexto sin terminar la sesión — ver §3.5 (Reset vs compaction) y `core/state-templates/handoff-protocol.md` §4.2. La compaction no sustituye a la secuencia de cierre.

Justificación (D9, hard_spec.md §5): (a) el protocolo interactivo actual es una aplicación directa de **P11** (empezar simple: el humano es el mecanismo de checkpoint sin construir infraestructura de automatización) y **P13** (human-on-the-loop: cada checkpoint es además materia prima del tuning loop); (b) interactivo y autónomo comparten la misma secuencia de sesión — la única diferencia es quién la relanza y qué se revisa en el checkpoint; (c) el runner autónomo es un evolutivo declarado pero no implementado en esta fase (ver R2, hard_spec.md §6).

## 2. Contratos de agente (core/contracts/)

Cada agente tiene su contrato en `core/contracts/{rol}.md`: entradas, salidas, criterio de "done" y criterio de parada. Los contratos son la fuente de verdad que el adaptador Claude Code referencia sin duplicar (P1).

Contratos disponibles: `orchestrator.md`, `planificador.md`, `navegador.md`, `implementador.md`, `evaluador.md`. Ninguno menciona formatos vendor-specific (O3, R3).

## 3. Artefactos de estado y handoff (core/state-templates/)

El directorio `core/state-templates/` contiene las plantillas que constituyen el sistema de registro del arnés, estratificado según D10 (hard_spec.md §5): especificación (`hard_spec.md`, fuera de este directorio) + estado (`state.json`) + progreso (`progress.md`).

### 3.1 La capa de estado canónica y el resto de plantillas

| Fichero | Propósito | Estado |
|---|---|---|
| `state.json` | Índice de bloques/tareas con su campo de estado (`pending`/`in_progress`/`complete`/`failed`). Única mutación permitida: ese campo; las definiciones y criterios viven en `hard_spec.md`. | **Vigente (D10)** |
| `progress.md` | Notas de sesión append-only: qué se hizo, veredicto del evaluador, bugs/hallazgos, siguiente paso. Permite retomar trabajo tras el cierre de sesión (D9). | **Vigente (D10)** |
| `task-contract.json` | Contrato Structured Contract (P14): payload completo del handoff, validación pre-handoff, audit trail, fallbacks. | Vigente |
| `handoff-protocol.md` | Protocolo de referencia (no plantilla rellenable): reglas que rigen el uso de `state.json`, `progress.md` y `task-contract.json` (validación pre-handoff, audit trail, reset/compaction, fallback). | Vigente |
| `dudas.md` | Registro de preguntas abiertas al cliente o al equipo con formato estandarizado. | Vigente |
| `active-plan.md` | Plantilla retirada (B10): su rol —objetivo/criterios/decisiones del bloque en curso— queda subsumido por `state.json` (estado) + `hard_spec.md` (definiciones) + `progress.md` (decisiones y notas). Se mantiene solo como nota de migración. | **Retirado (D10)** |
| `progress-log.md` | Plantilla retirada (B10): su rol —log append-only por sesión— queda subsumido por `progress.md`. Se mantiene solo como nota de migración. | **Retirado (D10)** |

### 3.2 Por qué Structured Contract y no los otros patrones (D7, P14)

El handoff entre agentes puede implementarse mediante tres patrones. El arnés elige Structured Contract porque es el único que garantiza persistencia, auditabilidad y legibilidad humana sin depender de la memoria de ningún agente.

| Patrón | Descripción | Por qué se descarta |
|---|---|---|
| **Shared Blackboard** | Espacio compartido donde los agentes leen y escriben estado libremente. | Estado implícito: difícil de auditar. El humano no puede saber quién escribió qué ni cuándo. Riesgo de sobreescritura silenciosa. |
| **Direct Message Passing** | El agente emisor envía un mensaje al receptor en el momento del handoff. | Sin persistencia: si la sesión termina, el mensaje se pierde. Acopla emisor y receptor en el mismo contexto. No hay audit trail. |
| **Structured Contract** | Fichero versionado en el repositorio con payload completo, criterios de aceptación, validación pre-handoff, audit trail y fallbacks. | **Patrón elegido.** Payload explícito, persistente, auditable, con fallback codificado y verificable por el humano. |

### 3.3 Protocolo de validación pre-handoff

El bloque `pre_handoff_validation` del contrato tiene cinco campos booleanos que el receptor confirma antes de actuar. Solo cuando `ready_to_proceed` es `true` el receptor puede producir artefactos.

| Campo | Condición que debe cumplirse |
|---|---|
| `context_understood` | El `purpose` y el `context.current_state` son suficientes para actuar sin suposiciones adicionales. |
| `input_artifacts_accessible` | Todos los ficheros de `artifacts.input` existen y son legibles. |
| `acceptance_criteria_clear` | Cada criterio de `acceptance_criteria` es verificable sin interpretación subjetiva. |
| `no_schema_divergence_detected` | El contrato es coherente con el esquema del proyecto y con el resto de contratos en `tasks/`. |
| `ready_to_proceed` | Solo `true` cuando los cuatro campos anteriores son `true`. |

Si `ready_to_proceed` es `false`, el receptor activa el fallback correspondiente sin producir artefactos.

### 3.4 Audit trail append-only

El campo `audit_trail` del contrato es un array de entradas cronológicas. Cada entrada tiene cuatro campos obligatorios: `timestamp`, `agent`, `action` y `result`. Las reglas son:

- **Append-only sin excepciones.** Una entrada escrita no se modifica; un error se corrige añadiendo una nueva entrada.
- **Una entrada por acción significativa.** No agregar múltiples acciones en una entrada.
- El planificador escribe la primera entrada al crear el contrato; el orquestador al invocar al implementador; el implementador al completar los artefactos; el evaluador al emitir el veredicto; el planificador al cerrar.

La inmutabilidad del audit trail garantiza que el humano puede reconstruir el historial completo del ciclo sin depender de la memoria de ningún agente.

### 3.5 Reset vs compaction (P5, D9, D10)

El arnés distingue dos situaciones de gestión de contexto:

**Context reset (cierre de sesión, D9)** — la sesión termina (cierre normal tras el checkpoint humano, límite de tokens). El estado persiste en la capa de estado de D10: `state.json` (campo de estado de cada bloque/tarea), `progress.md` (última entrada de sesión: qué se hizo, bugs, siguiente paso) y `tasks/*.json` (contrato con `status` y `audit_trail`), además de los artefactos versionados. La siguiente sesión ejecuta la **secuencia de re-entrada** de `core/orchestration/session-protocol.md`: lee `state.json` para identificar la tarea `in_progress` o, si no hay ninguna, la siguiente `pending`, lee la última entrada de `progress.md`, inspecciona el historial de control de versiones para confirmar baseline, e invoca al planificador con ese contexto reconstruido.

**Compaction** — la sesión está activa pero el contexto se satura, sin haber llegado al cierre de la tarea. El orquestador descarta la historia de conversación, retiene solo `state.json`, `progress.md`, el contrato JSON activo y `hard_spec.md` (la parte relevante), y continúa sin pérdida de estado funcional. Ver `core/state-templates/handoff-protocol.md` §4.2.

La distinción es clave: en la compaction el orquestador conserva su identidad y la sesión continúa; en el reset (cierre de sesión, D9), la siguiente sesión es fresca y reconstruye el estado desde `state.json`/`progress.md`.

### 3.6 Prevención de declaración de victoria prematura (P4)

Cuatro mecanismos mecánicos impiden que el implementador declare la tarea completa sin verificación del evaluador:

1. El ciclo no avanza hasta que el evaluador emite `APTO`. El orquestador nunca marca `status: complete` basándose en el reporte del implementador.
2. El implementador no tiene autoridad para modificar `acceptance_criteria` del contrato.
3. El evaluador es un agente con contexto separado; no tiene acceso al historial del implementador.
4. El `audit_trail` como prueba: un bloque no puede marcarse `complete` si no hay entrada de "Veredicto emitido: APTO".

Principios: **P4** (una tarea por iteración, estado persistido), **P5** (reset sin perder estado), **P14** (Structured Contract con payload explícito, audit trail, fallback). Decisión: **D7** (Structured Contract vs Shared Blackboard vs Direct Message Passing).

---

## 4. Adaptador Claude Code (adapters/claude-code/)

El directorio `adapters/claude-code/` es la capa vendor-specific del arnés. Materializa los contratos model-agnostic del `core/` como agentes ejecutables en Claude Code.

### 4.1 Los cuatro agentes

Cada agente se define en `adapters/claude-code/agents/{rol}.md` con dos secciones: un frontmatter YAML de definición Claude Code y un body que referencia el contrato del core sin duplicarlo.

| Agente | Fichero | Tools declarados | Referencia al core |
|---|---|---|---|
| planificador | `agents/planificador.md` | Read, Write, Edit, Glob, Grep, WebSearch, WebFetch, Agent, Bash | `../../core/contracts/planificador.md` |
| navegador | `agents/navegador.md` | Read, Glob, Grep, Bash, WebFetch, WebSearch | `../../core/contracts/navegador.md` |
| implementador | `agents/implementador.md` | Read, Write, Edit, Glob, Grep, Bash | `../../core/contracts/implementador.md` |
| evaluador | `agents/evaluador.md` | Read, Glob, Grep, Bash | `../../core/contracts/evaluador.md` |

El frontmatter YAML define `name`, `description` y `tools` para que Claude Code registre el agente. El body añade únicamente las instrucciones de ejecución Claude Code: cómo invocar tools, cómo estructurar la respuesta al orquestador.

La lógica del rol (entradas, salidas, criterio de done, criterio de parada) vive en `core/contracts/{rol}.md`. El fichero del adaptador no la duplica: la referencia.

### 4.2 La skill `harness/SKILL.md`

La skill materializa el patrón de orquestación de `core/orchestration/cycle.md` como una secuencia ejecutable.

**Trigger:** el usuario escribe "inicia el planificador para realizar el proyecto usando agentes" o pide explícitamente ejecutar el ciclo del arnés.

**Cinco pasos del workflow** (ejecutados dentro de la secuencia de sesión de D9, ver §1.1):

1. **Planificador** — lee `state.json`, `progress.md` y `hard_spec.md`, identifica la tarea activa (`in_progress`) o la siguiente `pending`, produce o actualiza el contrato JSON en `tasks/`, marca la tarea como `in_progress` en `state.json`, devuelve ruta del contrato y lista de criterios.
2. **Implementador** — recibe la ruta del contrato JSON, confirma `pre_handoff_validation`, produce todos los artefactos de `artifacts.output`, lista los ficheros creados.
3. **Evaluador** — recibe la ruta del contrato y la lista de artefactos producidos, verifica cada `acceptance_criterion`, emite `APTO` o `NO APTO` con defectos concretos.
4. **Planificador (cierre)** — recibe el veredicto, actualiza `status` del contrato a `complete` o `failed`, actualiza el campo de estado en `state.json` y añade una entrada nueva a `progress.md`.
5. **Decisión de continuación** — si `APTO`, se ejecuta la secuencia de cierre de sesión de D9 (commit, checkpoint humano, `/clear`); el humano decide si arranca una sesión fresca para la siguiente tarea. Si `NO APTO`, reintenta el implementador (máx. 2 veces); si falla dos veces, escala al humano.

### 4.3 Plugin manifest

| Fichero | Propósito |
|---|---|
| `.claude-plugin/plugin.json` | Manifiesto del plugin: `name`, `displayName`, `version`, `skills` apuntando a `../skills`. |
| `.claude-plugin/marketplace.json` | Catálogo del marketplace: registra el plugin con source `./` e instrucciones de instalación. |

### 4.4 Frontera core/adaptador (O3, R3)

**Regla fundamental:** ningún fichero del adaptador duplica contratos del core. El adaptador solo añade la capa de ejecución Claude Code (tools, formato de respuesta, trigger de skill).

El **punto de sustitución** (O3) está claramente definido: para sustituir Claude Code por otro runtime (OpenAI Assistants, LangGraph, etc.), basta con crear `adapters/{nuevo-runtime}/` que referencie los mismos ficheros de `core/`. El `core/` no se modifica.

Vigilar R3: si algún fichero del adaptador empieza a duplicar lógica de los contratos del core (criterios de done, condiciones de parada, definición de entradas/salidas), es señal de que la frontera se está erosionando. Corrección: mover la lógica al fichero `core/contracts/{rol}.md` correspondiente.

Principios: **P1** (separar core reutilizable de configuración de proyecto), **P12** (herramientas pensadas para agentes: namespacing, tokens eficientes). Decisiones: **D2** (core model-agnostic + adaptador Claude Code aislado), **D6** (empaquetado como plugin/clonable). Objetivos: **O3** (Claude Code hoy, model-agnostic mañana), **R3** (frontera exacta core/adaptador a vigilar).

---

## 5. Sensores de calidad para ingeniería de datos (core/sensors/)

El directorio `core/sensors/` contiene las especificaciones de los sensores de calidad del arnés. Los sensores son especificaciones portables; la implementación de referencia en herramientas concretas (dbt, Great Expectations, sqlfluff) queda pendiente de confirmar el stack (R1).

### 5.1 Catálogo de 7 sensores

| ID | Nombre | Capa | Cuándo corre | Invariante que protege |
|---|---|---|---|---|
| FF-01 | Schema Contract | fast-feedback | in-session + CI-pipeline | Schema de tabla/dataset coincide con el contrato de datos declarado |
| FF-02 | Data Quality | fast-feedback | CI-pipeline | Nulls en columnas críticas, duplicados, freshness, volume anómalo, validez semántica |
| FF-03 | Pipeline Tests | fast-feedback | in-session + CI-pipeline | Tests unitarios e integración del pipeline pasan sin errores |
| FF-04 | SQL/Python Lint | fast-feedback | in-session (pre-commit) + CI | Código SQL y Python cumple reglas de estilo y convenciones del equipo |
| DP-01 | Golden Data Principles | drift-periódico | periódico (cadencia configurable) | Degradación acumulada de calidad, documentación y gobernanza |
| DP-02 | Lineage Staleness | drift-periódico | periódico (cadencia configurable) | Assets con upstream modificado cuyo lineage downstream no se ha actualizado |
| DP-03 | Catalog Completeness | drift-periódico | periódico (cadencia configurable) | Tablas sin owner, assets sin contrato, columnas sin descripción |

### 5.2 Cobertura por dimensión de invariante

| Dimensión | Sensor(es) |
|---|---|
| Schema / tipos | FF-01 |
| Freshness | FF-02 |
| Nullability | FF-02 |
| Duplicados | FF-02 |
| Validez semántica | FF-02 |
| Volume / row count | FF-02 |
| Contrato de datos | FF-01 |
| Tests de pipeline | FF-03 |
| Calidad de código SQL/Python | FF-04 |
| Lineage | DP-02 |
| Completitud del catálogo | DP-03 |
| Golden principles de datos | DP-01 |

### 5.3 Principio de diseño del feedback

Cada sensor produce un mensaje de feedback que cumple cuatro requisitos:

- **Referenciable** — incluye identificador del sensor (FF-01, DP-03…) para que el audit trail pueda registrarlo sin ambigüedad.
- **Accionable** — describe qué falla, qué se esperaba y qué remediación concreta se sugiere; no emite mensajes genéricos.
- **Eficiente en tokens** — formato estructurado y breve (tabla columna/tipo/resultado); no prosa larga que consuma contexto del agente.
- **Estructurado** — salida en formato verificable mecánicamente (`SENSOR OK` / `SENSOR FAIL` + payload de discrepancias), no solo texto libre.

### 5.4 Estado de implementación

Los sensores de `core/sensors/` son especificaciones; describen qué comprueba cada sensor, cuándo corre y qué feedback emite. La implementación concreta en herramientas del stack (dbt tests, Great Expectations, sqlfluff, etc.) se añade en la sección "Implementaciones de referencia" de cada fichero de sensor cuando R1 quede confirmado.

Principios: **P8** (invariantes mecánicas, no micromanagement), **P9** (sensores en capas: fast-feedback vs drift periódico). Decisión: **D4** (invariantes de datos como sensores mecánicos, no prosa).

---

## 6. Baseline de evals del arnés (core/evals/)

El directorio `core/evals/` contiene el baseline de evals que verifica que el arnés dirige correctamente al agente. Todos los evals son conceptuales (R4): tienen estructura completa pero sin ejecución real hasta disponer de un proyecto de datos real.

### 6.1 Los tres evals del baseline

| ID | Título | Área de riesgo principal |
|---|---|---|
| eval-01 | Validación de schema en ingestión | ¿Activa el agente FF-01 antes de avanzar? ¿Reporta discrepancias de forma accionable? |
| eval-02 | Generación de modelo de transformación (staging → mart) | ¿Produce código correcto, idiomático y bien documentado? |
| eval-03 | Seguimiento del protocolo del arnés | ¿Sigue el agente el protocolo completo (contrato, sensores, actualización de `state.json`/`progress.md`, no victoria prematura)? |

### 6.2 Estructura de un eval YAML

Cada eval tiene los siguientes bloques:

```
id, title, status     — identificación y estado (conceptual / activo)
description           — escenario completo del eval: qué recibe el agente, qué se espera
prompt                — el prompt exacto que recibe el agente en el run
run_capture           — referencia al JSON de la transcripción capturada del run
checks
  deterministic       — comprobaciones mecánicas sin interpretación (det-01, det-02…)
  rubric_based        — comprobaciones con rúbrica 1-5 y umbral (rub-01, rub-02…)
score
  axes                — 4 ejes: outcome, process, style, efficiency (pesos suman 1.0)
  formula             — weighted_average(outcome, process, style, efficiency)
  pass_threshold      — 0.70 (por defecto)
tuning_notes          — fallos esperados en primeras iteraciones y correcciones sugeridas
```

### 6.3 scorecard-schema.json: campos clave

El fichero `core/evals/scorecard-schema.json` define el esquema JSON para registrar el resultado de un eval. Campos obligatorios:

| Campo | Descripción |
|---|---|
| `eval_id` | Identificador del eval ejecutado (ej: `eval-01`). |
| `variant` | Variante de config/prompts evaluada. Permite comparar A vs B (ej: `baseline`, `v1.1-prompt-refinement`). |
| `run_date` | Fecha de ejecución (YYYY-MM-DD). |
| `conceptual` | `true` si el scorecard es conceptual sin ejecución real (R4). |
| `axes` | Objeto con los 4 ejes; cada uno con `weight`, `raw_score`, `weighted_score`. |
| `overall.score` | Resultado de `weighted_average`; `null` si conceptual. |
| `verdict` | `PASS` / `FAIL` / `CONCEPTUAL`. |
| `failed_checks` | Lista de checks fallidos con `finding` y `tuning_action` (`add_check_to_eval` / `add_sensor_to_core` / `escalate_to_human`). |

### 6.4 tuning-loop.md: mecanismo de aprendizaje

El tuning loop convierte fallos observados en checks o sensores que impiden su repetición.

**Mecanismo 1 — Inmediato: fallo → nuevo check en el eval YAML.** Cuando el fallo es puntual o la primera vez que se observa, se añade un check determinista o rubric-based al YAML del eval afectado. Los checks existentes nunca se editan; solo se añaden nuevos (el historial de scorecards sigue siendo comparable).

**Mecanismo 2 — Diferido: fallo recurrente → sensor en `core/sensors/`.** Cuando el mismo fallo aparece en dos evals distintos o en dos ejecuciones consecutivas del mismo eval, el fallo es sistémico. Se crea o actualiza un sensor en `core/sensors/fast-feedback/` (in-session/CI) o `core/sensors/drift-periodic/` (periódico), y el check del eval pasa a verificar la salida del sensor.

Si el fallo persiste tras dos actualizaciones del mecanismo correspondiente, se escala al humano con una entrada en `docs/dudas.md` con la etiqueta `[TUNING-ESCALADO]`.

### 6.5 Estado

Todos los evals del baseline son conceptuales (R4). La ejecución real requiere un proyecto de datos concreto con dataset, stack y pipeline definidos.

Principios: **P10** (evals como prueba del arnés: prompt → run → checks → score), **P11** (empezar simple, añadir complejidad solo cuando aporte).

---

## 7. Plantilla de proyecto de datos (project-template/)

El directorio `project-template/` contiene la estructura base que cada equipo de ingeniería de datos copia al arrancar un nuevo proyecto con el arnés.

### 7.1 CLAUDE.md como índice del proyecto (~80-100 líneas, P2)

El fichero `project-template/CLAUDE.md` actúa como punto de entrada del agente. Su función es ser índice, no repositorio de información: ~80-100 líneas que orientan al agente hacia `docs/` para profundidad.

Secciones del CLAUDE.md del proyecto:

| Sección | Contenido | Profundidad en |
|---|---|---|
| Qué es este proyecto | 1-2 frases de objetivo de negocio + equipo propietario. | — |
| Stack de datos | Stack elegido (marcar al confirmar R1); enlace a `docs/references/stack.md`. | `docs/references/stack.md` |
| Capas de datos | Opción A (landing/staging/marts) u opción B (bronze/silver/gold); a confirmar en R1. | `docs/architecture/data-layers.md` |
| Comandos más usados | Tabla de 6 comandos/lecturas frecuentes del agente. | — |
| Dónde encontrar qué | Tabla de 9 rutas: decisiones, capas, pipelines, SLAs, naming, stack, dudas, contratos del core, sensores. | `docs/` |
| Frontera core / proyecto | Regla: `../../core/` no se modifica por proyecto. | — |

### 7.2 data-conventions.md: convenciones específicas del proyecto

El fichero `project-template/data-conventions.md` declara las convenciones de datos que el agente debe seguir en este proyecto concreto.

**Capas de datos:** dos opciones mutuamente excluyentes (R1):

- **Opción A — landing / staging / marts:** `lnd_`, `stg_`, `mrt_`. Nomenclatura clásica de Data Warehouse. Invariantes: staging solo lee de landing; marts son la única capa expuesta a BI.
- **Opción B — bronze / silver / gold:** `brz_`, `slv_`, `gld_`. Nomenclatura medallón, habitual en Data Lakehouse. Invariantes: bronze no se expone directamente; gold es la única capa expuesta.

**Naming:** snake_case en tablas, modelos y ficheros. Columnas de auditoría obligatorias en todos los modelos: `id`, `created_at`, `updated_at`, `_source`, `_loaded_at`. Columnas monetarias con sufijo `_amount`.

**Contratos de datos:** fichero YAML por tabla con campos `asset`, `owner`, `sla_hours`, `columns` (tipo, nullability, descripción, `critical`), `quality` (primary_key, not_null, unique) y `tags` (layer, domain).

**8 invariantes de calidad:**

| Invariante | Sensor |
|---|---|
| Schema coincide con el contrato | FF-01 |
| Columnas críticas sin nulos | FF-02 |
| Clave primaria sin duplicados | FF-02 |
| Dataset dentro del SLA de freshness | FF-02 |
| Volume dentro de umbrales | FF-02 |
| Tests del pipeline pasan | FF-03 |
| Código SQL/Python cumple estilo | FF-04 |
| Todo asset en capa expuesta tiene owner | DP-03 |

**Stack:** pendiente de confirmar (R1). Las opciones estándar son dbt+Snowflake, dbt+BigQuery y Spark+Delta Lake. La tabla de herramientas en `data-conventions.md §5` lista opciones por categoría (transformación, orquestación, warehouse/lake, calidad, catálogo, linting SQL, linting Python).

### 7.3 docs/: sistema de registro del proyecto (5 subdirectorios)

El directorio `project-template/docs/` organiza la documentación específica del proyecto en cinco subdirectorios:

| Subdirectorio | Contenido |
|---|---|
| `docs/architecture/` | `decisions.md` (ADRs), `data-layers.md` (capas elegidas + invariantes). |
| `docs/pipelines/` | `index.md` (catálogo de pipelines activos). |
| `docs/quality/` | `slas.md` (SLAs de freshness y calidad por dataset). |
| `docs/references/` | `stack.md` (stack confirmado al resolver R1). |
| `docs/dudas.md` | Backlog de preguntas abiertas al cliente o al equipo. |

### 7.4 Frontera core/proyecto (P1)

El directorio `../../core/` es inmutable desde la perspectiva del proyecto. Si el equipo necesita cambiar el comportamiento de un sensor o un contrato de agente, abre una duda en `docs/dudas.md` y escala al equipo de plataforma. Lo que sí es específico del proyecto y reside en `project-template/` son: `CLAUDE.md`, `data-conventions.md` y todo `docs/`.

Principios: **P1** (separar core reutilizable de configuración de proyecto), **P2** (progressive disclosure: CLAUDE.md como índice), **P3** (repo-local truth: conocimiento versionado en el repo).

---

## 8. Empaquetado portable y README (README.md)

El arnés se distribuye como un repositorio git clonable. No requiere instalación de paquetes; el único runtime necesario es Claude Code.

### 8.1 Instrucciones de instalación

```
git clone {url_del_repo}
cd data-eng-harness
```

Apuntar Claude Code al plugin del adaptador:

```
claude --plugin harness-v3-ai-generated/adapters/claude-code/
```

O añadir el directorio `adapters/claude-code/` como plugin en la configuración del proyecto de Claude Code.

### 8.2 Cómo arrancar el ciclo

1. Copiar `project-template/` al directorio del nuevo proyecto de datos.
2. Rellenar `CLAUDE.md` del proyecto (objetivo, stack, capas).
3. Inicializar la capa de estado del proyecto a partir de `core/state-templates/state.json` y `core/state-templates/progress.md`.
4. Confirmar R1 (stack del equipo) antes de usar los sensores en producción.
5. Escribir al agente: `inicia el planificador para realizar el proyecto usando agentes`.

El ciclo completo (planificador → implementador → evaluador → planificador) se ejecuta dentro de la secuencia de sesión única de D9 (re-entrada → ciclo de 4 agentes → cierre de sesión). El estado persiste en `state.json` (campo de estado de cada bloque/tarea), `progress.md` (notas de sesión append-only) y `tasks/*.json`; si la sesión se interrumpe o cierra, la próxima sesión arranca ejecutando la secuencia de re-entrada de `core/orchestration/session-protocol.md`: lee `state.json`, identifica la tarea activa o siguiente, lee la última entrada de `progress.md` y verifica el baseline contra el historial de control de versiones.

### 8.3 Model-agnostic future (O3)

Los ficheros vendor-specific de Claude Code están todos en `adapters/claude-code/`. El `core/` no contiene ninguna referencia a Claude Code ni a ningún otro runtime.

Para sustituir Claude Code:

1. Crear `adapters/{nuevo-runtime}/` con los cuatro agentes y la skill equivalente.
2. Los agentes del nuevo adaptador referencian los mismos contratos de `core/contracts/`.
3. No modificar ningún fichero de `core/`.

### 8.4 Backlog de decisiones abiertas (R1-R4)

| ID | Decisión abierta | Estado | Impacto |
|---|---|---|---|
| R1 | Stack de datos del equipo (dbt, Airflow, Spark, Snowflake/BigQuery) | Resuelto por D11 (hard_spec.md §6) | Bloquea implementaciones de referencia de sensores; bloquea `data-conventions.md §5`. |
| R2 | Grado de autonomía / política de aprobación humana por bloque | Resuelto por D12 (hard_spec.md §6) | Afecta al número de paradas obligatorias del ciclo. |
| R3 | Frontera exacta core/adaptador | Vigilar | Si el adaptador duplica lógica del core, la frontera se erosiona. |
| R4 | Alcance ejecutable de evals | Sin confirmar | Los evals son conceptuales; se activan con proyecto real. |

Objetivo: **O2** (portabilidad: un único artefacto instalable/clonable), **O3** (Claude Code hoy, model-agnostic mañana).

---

## 9. Decisiones de diseño D1-D13

Esta sección registra las decisiones de diseño del arnés con sus justificaciones en los principios P1-P14 y los objetivos O1-O5. Es el punto de referencia para evaluar propuestas de cambio: cualquier modificación que contradiga una decisión de diseño debe explicitar por qué el principio subyacente ya no aplica.

| ID | Decisión | Principios que materializa | Elemento(s) del arnés |
|---|---|---|---|
| D1 | Ciclo de 4 agentes (planificador / navegador / implementador / evaluador) justificado por la complejidad multi-sesión de las tareas de datos. | P4, P6, P13 | `core/orchestration/cycle.md`, `adapters/claude-code/skills/harness/SKILL.md` |
| D2 | Core model-agnostic + adaptador Claude Code aislado. | P1, O3 | `core/` (portable), `adapters/claude-code/` (vendor-specific) |
| D3 | Estado en repo, markdown/JSON como sistema de registro. | P2, P3, O4 | `core/state-templates/state.json`, `core/state-templates/progress.md`, `tasks/*.json`, `project-template/docs/` |
| D4 | Invariantes de datos como sensores mecánicos, no prosa. | P8, P9 | `core/sensors/catalog.md`, `core/sensors/fast-feedback/`, `core/sensors/drift-periodic/` |
| D5 | Una tarea por iteración como regla del bucle. | P4 | `core/orchestration/stop-conditions.md`, `adapters/claude-code/skills/harness/SKILL.md` |
| D6 | Empaquetado como plugin/clonable. | O2 | `adapters/claude-code/.claude-plugin/plugin.json`, `adapters/claude-code/.claude-plugin/marketplace.json` |
| D7 | Handoff via Structured Contract (vs Shared Blackboard vs Direct Message Passing). | P5, P14 | `core/state-templates/task-contract.json`, `core/state-templates/handoff-protocol.md` |
| D8 | Orquestador como hilo principal separado del planificador. | P6, P13 | Definido en `core/orchestration/cycle.md`; materializado en `adapters/claude-code/skills/harness/SKILL.md` |
| D9 | Protocolo de sesión único con política de checkpoint parametrizable (re-entrada → ciclo de 4 agentes → cierre de sesión; regla "una tarea por sesión"; contrato de checkpoint humano). | P2, P4, P11, P13 | `core/orchestration/session-protocol.md`, `core/orchestration/cycle.md` (diagrama de cierre de sesión), `adapters/claude-code/skills/harness/SKILL.md` (§1.1, §4.2 de este documento) |
| D10 | Estratificación spec/estado/progreso: `hard_spec.md` (qué/por qué, casi inmutable) + `state.json` (índice de bloques/tareas con campo de estado, alta frecuencia de mutación) + `progress.md` (notas de sesión append-only). Sustituye a `SPEC.md`. | P2, P3, P5 | `core/state-templates/state.json`, `core/state-templates/progress.md`, `hard_spec.md` (referencia, no se modifica por esta decisión) |
| D11 | Perfil de stack por proyecto: el core define categorías de sensor parametrizables (lint SQL, lint/typecheck Python, validación de schema/contratos, freshness/nulls/volumetría, tests de pipeline) en lugar de codificar herramientas concretas para el abanico multi-cloud de los clientes de The Cocktail. Cada proyecto instancia las categorías mediante un perfil de stack. Python + SQL + Spark/PySpark, `sqlfluff`, `ruff` + `mypy` y DuckDB son el soporte de referencia de primera clase. | P1, P11 | `core/sensors/catalog.md`, `project-template/stack-profile.yml` |
| D12 | Dos políticas de checkpoint que instancian el parámetro de D9: `validacion-por-tarea` (por defecto en v1, checkpoint humano al final de cada tarea, cierre con `/clear`) y `full-auto` (encadena tareas sin checkpoint humano intermedio). Los checkpoints duros (operaciones DDL/DML destructivas o sobre sistemas de cliente, y modificaciones de `hard_spec.md`) son un invariante del protocolo de sesión, no delegable en ninguna política. | D9, P13 | `core/orchestration/` |
| D13 | El evaluador no modifica artefactos del implementador, pero EJECUTA: corre pipelines, lanza queries contra el entorno de pruebas, ejecuta tests y sensores fast-feedback (FF-01 a FF-04), y basa su veredicto en resultados de ejecución, no solo en lectura de código/SQL. El contrato de tarea es bidireccional: el implementador anexa `execution_summary` que el evaluador lee como entrada de contexto, sin sustituir la verificación directa. | P6, P9, P5 | `core/contracts/evaluador.md`, `core/sensors/catalog.md`, `core/sensors/fast-feedback/*.md`, `adapters/claude-code/agents/evaluador.md`, `core/state-templates/task-contract.json` |

### 9.1 Justificación de D9 y D10

**D9 — Protocolo de sesión único con política de checkpoint parametrizable.** El arnés define un único protocolo de sesión (`core/orchestration/session-protocol.md`), no dos modos separados ("interactivo" vs "autónomo"). La política de checkpoint —quién relanza la sesión y qué se le muestra— es un parámetro de ese protocolo:

- **(P11, empezar simple)** la fase actual usa al humano como mecanismo de checkpoint en cada iteración, sin construir infraestructura de automatización.
- **(P13, human-on-the-loop)** cada checkpoint donde el humano revisa veredicto + diff + estado actualizado es además materia prima del tuning loop: las observaciones del humano (qué corrige, qué aprueba sin más) son la señal que justificará en el futuro qué checkpoints son delegables (R2).
- **(P4, una tarea por iteración)** la regla "una tarea por sesión" es la misma regla "una vuelta = una tarea" del ciclo de 4 agentes, aplicada al límite de la sesión completa: cada sesión ejecuta como mucho un ciclo planificador → implementador → evaluador → planificador.
- **(P2, economía de contexto)** interactivo y autónomo comparten exactamente la misma secuencia (re-entrada determinista → ciclo de 4 agentes → cierre de sesión); la única diferencia es quién relanza la sesión y qué se muestra/requiere en el checkpoint. Un futuro runner autónomo (R2, no implementado en esta fase) ejecutaría la misma secuencia automatizando, muestreando o escalando selectivamente partes del contrato de checkpoint según el riesgo de la tarea.

**D10 — Estratificación `hard_spec.md` + `state.json` + `progress.md`.** Sustituye a `SPEC.md` (que mezclaba índice de estado con detalle de criterios) por una capa de estado con contrato propio:

- **(P3, repo-local truth + "JSON sobre Markdown")** el estado se muta en cada iteración mientras que las definiciones y criterios de `hard_spec.md` casi nunca cambian; un `state.json` estructuralmente trivial permite codificar mecánicamente la regla "solo se muta el campo de estado, nunca las descripciones" — algo que un `SPEC.md` en Markdown libre no garantiza (corpus Anthropic *effective-harnesses*: "JSON over Markdown for machine-readable state").
- **(P5, handoff entre sesiones, asimetría de desechabilidad)** `state.json`/`progress.md` son una proyección derivada de `hard_spec.md` + el estado real del repo: si se corrompen o se pierden, son regenerables inspeccionando `git log` y `hard_spec.md` (patrón `fix_plan.md` de Huntley: el plan es desechable y se regenera cada vuelta, mientras las especificaciones son la base estable).
- **(P2, economía de contexto aplicada al propio arnés)** el orquestador, al arrancar una sesión fresca y amnésica, necesita "cuál es la siguiente tarea" en un fichero corto, no la especificación completa — consistente con `AGENTS.md` (~100 líneas) como mapa de repositorio (OpenAI Codex) y con la separación `specs/*` vs `@fix_plan.md` (Huntley).
- **Precedente del corpus**: el mismo trío intención/estado/notas aparece en Anthropic (`app_spec` + `feature_list.json` + fichero de progreso), Huntley (`specs/*` + `@fix_plan.md` + `AGENT.md`) y OpenAI Codex (`product-specs/` + `exec-plans/active/` + `AGENTS.md`). La estratificación `soft_spec.md` → `hard_spec.md` → `state.json`/`progress.md` aplica el mismo patrón al propio arnés.

> Nota de migración (D10): `SPEC.md` como artefacto de estado activo del meta-proyecto queda sustituido por `state.json` + `progress.md`. El fichero `SPEC.md` de la raíz del repositorio se conserva físicamente como referencia histórica de las migraciones B9/B10, pero ningún elemento de `harness-v3-ai-generated/` lo trata como capa de estado activa.

### 9.2 Justificación de D13 (evaluador ejecutor + sensores invocables + contrato bidireccional)

**D13 — El evaluador no modifica artefactos, pero ejecuta.** `core/contracts/evaluador.md` deja de
describir al evaluador como un rol que "solo lee, no corrige" sin más matiz, y pasa a la
formulación precisa: el evaluador **no modifica artefactos del implementador** (invariante P6
intacta), **pero EJECUTA** — corre los pipelines, lanza queries contra el entorno de pruebas,
ejecuta los tests y los sensores fast-feedback de `core/sensors/catalog.md` (FF-01 a FF-04), y basa
su veredicto en resultados de **ejecución**, no solo en lectura de código o de SQL.

- **Trazabilidad (corpus Anthropic *harness-design*).** El evaluador del proyecto de referencia
  necesitó Playwright para encontrar bugs reales que la lectura de código no detectaba. En
  ingeniería de datos el equivalente es igual de real: un evaluador que solo lee SQL no detecta
  rutas que colisionan, joins que explotan filas o schemas incompatibles — errores que solo
  aparecen al ejecutar la query contra datos reales o un entorno de pruebas (DuckDB u otro
  warehouse local, según el perfil de stack de D11).
- **(P6, separar quien hace de quien juzga, sin diluirlo)** "no modifica artefactos" sigue siendo
  la línea que separa al evaluador del implementador; "ejecutar para obtener evidencia" no es
  "corregir" — el evaluador no escribe ni edita el entregable, solo lo ejecuta para verificarlo.
- **(P9, sensores en capas)** los sensores **fast-feedback** (`core/sensors/fast-feedback/*.md`,
  FF-01 a FF-04) pasan de ser solo herramientas del implementador en desarrollo a ser también
  **invocables por el evaluador** como parte de su veredicto, declarado explícitamente en
  `core/sensors/catalog.md`. Los sensores **drift-periódico** (DP-01 a DP-03) quedan fuera de esta
  invocación: corren con su propia cadencia, no en el ciclo por-tarea del evaluador.
- **(P5, handoff estructurado, contrato bidireccional)** el contrato de tarea deja de ser
  unidireccional planificador→implementador→evaluador: el implementador anexa al cerrar la tarea
  un bloque `execution_summary` (qué hizo, decisiones, desviaciones, artefactos realmente
  producidos) que el evaluador lee como entrada de contexto junto a `acceptance_criteria` — indica
  qué verificar y dónde, sin sustituir la verificación (y ejecución) directa de los artefactos.

> Esta decisión se documenta como **capacidad/contrato** del rol evaluador, no como un runner de
> pipelines implementado: el arnés sigue siendo especificación (P11, empezar simple).
