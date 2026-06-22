# Protocolo de handoff y gestión de estado

> Documento de referencia model-agnostic. Define cómo se transfiere trabajo entre agentes, cómo se gestiona el estado entre sesiones y qué reglas evitan fallos silenciosos y declaraciones de victoria prematuras.
> Este documento no es una plantilla rellenable: es el conjunto de reglas que rige el uso de las plantillas de `core/state-templates/`.

---

## 1. Por qué Structured Contract (D7, P14)

El handoff entre agentes puede implementarse mediante tres patrones:

| Patrón               | Descripción                                                                  | Por qué se descarta                                                                   |
|----------------------|------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| **Shared Blackboard** | Un espacio compartido donde los agentes leen y escriben estado libremente. | Estado implícito: difícil de auditar a mano. El humano no puede verificar qué agente escribió qué ni cuándo. Riesgo de sobreescritura silenciosa. |
| **Direct Message Passing** | El agente emisor envía un mensaje (string) al receptor en el momento del handoff. | Sin persistencia: si la sesión termina, el mensaje se pierde. Acopla al emisor y al receptor en el mismo contexto. No hay audit trail. |
| **Structured Contract** | Un fichero versionado en el repositorio que contiene el payload completo del handoff, los criterios de aceptación, la validación pre-handoff, el audit trail y la lógica de fallback. | **Se elige este patrón.** |

Se elige **Structured Contract** porque:

- **Payload explícito.** El receptor no necesita inferir el contexto: lo tiene en el contrato.
- **Persistencia de estado.** El contrato es un fichero en el repositorio. Sobrevive a cualquier context reset o cierre de sesión.
- **Audit trail built-in.** El campo `audit_trail` registra de forma append-only quién hizo qué y cuándo sin depender de la memoria del agente.
- **Fallback codificado.** Las condiciones de fallo y las acciones a tomar están en el propio contrato, no en instrucciones verbales que pueden perderse entre sesiones.
- **Verificable por el humano.** El humano puede leer cualquier contrato JSON y entender el estado exacto del trabajo sin preguntar a ningún agente.
- **Coherencia con repo-local truth (O4).** El contrato vive en `tasks/` junto al código y los artefactos; el estado del proyecto es siempre accesible desde el repositorio.

### 1.1 Handoff bidireccional (D13)

El contrato de tarea **no es unidireccional** (planificador → implementador). El ciclo completo es:

```
planificador → [contrato con purpose/context/acceptance_criteria/artifacts/fallback]
   → orquestador invoca al implementador (status → active)
      → implementador produce los artefactos de artifacts.output
      → implementador ANEXA al mismo JSON el bloque execution_summary
        (qué se hizo, decisiones y su porqué, desviaciones respecto al plan,
         artefactos realmente producidos)
   → el contrato VUELVE enriquecido con execution_summary
      → evaluador lee execution_summary + acceptance_criteria como entrada
      → evaluador emite veredicto (APTO / NO APTO)
   → planificador/orquestador escribe SOLO status según el veredicto
```

**El orquestador ya no transmite verbalmente la lista de artefactos producidos.** Esa
información vive en `execution_summary`, dentro del propio contrato JSON, y es la que el
evaluador consume.

**Justificación (P14/Inferensys).** Un handoff verbal — el orquestador resumiendo de palabra
al evaluador qué produjo el implementador — es exactamente el anti-patrón "string simple" que
el patrón Structured Contract (P14) prohíbe: información de handoff sin persistencia, sin
estructura y sin audit trail. `execution_summary` es el campo homólogo de ese patrón para el
handoff de salida implementador → evaluador: estructurado, persistido en el repositorio y
verificable por el humano igual que el resto del contrato.

> Nota: `execution_summary` no sustituye a `artifacts.output`. `artifacts.output` es lo que el
> planificador esperaba al crear el contrato; `execution_summary.artifacts_produced` es lo que
> el implementador produjo realmente, y puede diferir (p. ej. si surgió un artefacto adicional
> no previsto). La diferencia, si la hay, se documenta en
> `execution_summary.deviations_from_plan`.

---

## 2. Protocolo de validación pre-handoff

El bloque `pre_handoff_validation` del contrato es la firma del receptor. Su propósito es evitar que un agente actúe sobre un contrato incompleto, ambiguo o inaccesible.

### 2.1 Qué hace el receptor al recibir un contrato

1. **Leer el contrato completo.** Verificar que `status` es `active` (el orquestador lo actualiza antes de invocar al implementador).
2. **Revisar `pre_handoff_validation`.** Para cada campo:
   - `context_understood` — ¿el `purpose` y el `context.current_state` son suficientes para actuar sin suposiciones adicionales?
   - `input_artifacts_accessible` — ¿todos los ficheros de `artifacts.input` existen y son legibles?
   - `acceptance_criteria_clear` — ¿cada criterio de `acceptance_criteria` es verificable sin interpretación subjetiva?
   - `no_schema_divergence_detected` — ¿el contrato es coherente con el esquema definido en el proyecto y con el resto de contratos?
   - `ready_to_proceed` — solo se marca `true` cuando los cuatro campos anteriores son `true`.
3. **Marcar cada campo como `true`** en el contrato cuando la condición se cumple.
4. **Si `ready_to_proceed` es `false`** — activar el fallback correspondiente (ver sección 5) antes de hacer cualquier otra cosa.
5. **Añadir entrada al `audit_trail`** con la acción "Validación pre-handoff completada" y el resultado (`ready_to_proceed: true/false`).

### 2.2 Quién confirma la validación

- **Implementador** — al recibir el contrato del orquestador antes de producir artefactos.
- **Evaluador** — al recibir el contrato del orquestador antes de verificar artefactos.

El planificador y el orquestador no confirman `pre_handoff_validation`; son quienes crean y transfieren el contrato.

---

## 3. Protocolo de audit trail

El campo `audit_trail` del contrato es un array append-only de entradas cronológicas. Cada entrada tiene cuatro campos obligatorios:

```
timestamp  — YYYY-MM-DD HH:MM (hora local de quien escribe la entrada)
agent      — uno de: orquestador | planificador | navegador | implementador | evaluador | humano
action     — descripción imperativa de la acción realizada (máx. 1 línea)
result     — resultado observable: estado resultante, veredicto, o descripción del output
```

### 3.1 Reglas del audit trail

- **Append-only sin excepciones.** Una vez escrita, una entrada no se modifica. Si hay un error, se añade una nueva entrada de corrección: `action: "Corrección de entrada #N: {descripción}"`.
- **Una entrada por acción significativa.** No agregar múltiples acciones en una misma entrada.
- **El planificador escribe la primera entrada** al crear el contrato (`action: "Contrato creado"`, `result: "status → drafted"`).
- **El orquestador escribe la entrada** al invocar al implementador (`action: "Implementador invocado"`, `result: "status → active"`).
- **El implementador escribe la entrada** al completar los artefactos (`action: "Artefactos producidos"`, `result: lista de ficheros generados`) y, en el mismo turno, rellena el bloque `execution_summary` del contrato (ver sección 1.1) con el detalle completo — la entrada del audit trail es un resumen breve, `execution_summary` es la fuente de verdad detallada.
- **El evaluador escribe la entrada** al emitir el veredicto (`action: "Veredicto emitido"`, `result: "APTO" o "NO APTO — {resumen de defectos}"`), tras leer `execution_summary` + `acceptance_criteria` como entrada.
- **El planificador escribe la entrada de cierre** al actualizar `status` al estado terminal correcto (`action: "Status actualizado"`, `result: "status → fulfilled"` si APTO; `violated` si se rompió una cota de `governance` o se agotaron los reintentos; ver §3.4 y `_status_semantics` del contrato).

### 3.2 Por qué el audit trail es inmutable

El audit trail es la única fuente de verdad sobre el historial del contrato que sobrevive a cualquier context reset. Si se permite modificarlo, el historial puede falsificarse inadvertidamente al retomar el trabajo en una nueva sesión. La inmutabilidad garantiza que el humano puede auditar el ciclo completo sin depender de la memoria de ningún agente.

### 3.3 Protocolo de telemetría de uso (`resource_usage`, D16)

El bloque `resource_usage` del contrato captura el **coste** de cada iteración del ciclo de 4 agentes: cuántos tokens, cuántos usos de herramienta y cuánto tiempo consume cada invocación de subagente, y el total agregado de la tarea. Es la materia prima del *tuning loop* (P10) y la evidencia que permite al humano auditar el coste real de un bloque (resuelve parcialmente R5).

Estructura:

- `by_agent` — array **append-only**, una entrada por **invocación** de subagente, con: `agent` (`planificador` | `navegador` | `implementador` | `evaluador`), `invocation` (contador secuencial por agente), `tokens`, `tool_uses` y `duration_seconds` (tiempo de pared de la invocación, en segundos enteros).
- `task_total` — agregado (`invocations`, `tokens`, `tool_uses`, `duration_seconds`) que se recalcula tras cada entrada nueva.

Reglas:

1. **Lo rellena exclusivamente el orquestador.** Es el único agente que recibe el bloque de uso (`<usage>`) al término de cada invocación de subagente; un subagente no puede medir su propio consumo y no escribe nunca en este bloque.
2. **El planificador inicializa el bloque vacío** al crear el contrato: `by_agent: []` y `task_total` con todos los campos a cero. (La plantilla incluye una entrada de ejemplo con placeholders que el planificador sustituye por un array vacío.)
3. **Una entrada por invocación, append-only.** Cada vez que el orquestador invoca a un subagente, añade una entrada al final de `by_agent`. Un reintento tras `NO APTO` genera una entrada **nueva** (con `invocation` incrementado para ese agente), nunca sobrescribe la anterior — igual que el audit trail, el histórico de coste es inmutable.
4. **`task_total` se recalcula tras cada entrada.** El orquestador suma todas las entradas de `by_agent`; `invocations` es el número de entradas.
5. **El consumo del propio orquestador no se contabiliza por entrada.** El orquestador es el hilo principal de la sesión: su coste es coste de sesión, no de una invocación discreta.

> Nota: `resource_usage` registra el coste *dentro* del mismo contrato estructurado (P14), no en un fichero de telemetría aparte. Coherente con O4 (repo-local truth): el coste de cada tarea es auditable desde el propio JSON, sin herramientas externas.

### 3.4 Protocolo de gobernanza de recursos (`governance`, D17)

Mientras `resource_usage` (§3.3) *mide* el coste a posteriori, el bloque `governance` lo **acota a priori** (P15). Es la diferencia entre observabilidad y gobernanza: el contrato deja de solo describir cuánto se consumió y pasa a declarar cuánto *puede* consumirse antes de activar la tarea.

**Quién declara qué (ex-ante).** El **planificador**, al crear el contrato, rellena `governance` y no lo vuelve a tocar:

- `R` — presupuesto multidimensional del ciclo completo (`tokens`, `invocations`, `retries`). Es el techo agregado de todas las invocaciones de subagente de la tarea. Es independiente de `artifacts.input`: un input pequeño puede consumir muchos tokens razonando.
- `T` — cota temporal (`ttl_sessions`, `deadline?`).
- `termination_conditions` (Ψ) — causas de cierre explícitas, independientes del éxito (presupuesto agotado, expiración, cancelación).
- `conservation` — referencia al presupuesto del bloque padre en `state.json`: Σ(presupuestos de las tareas del bloque) ≤ presupuesto del bloque. El planificador la verifica al crear el contrato.

**Quién hace cumplir (durante la ejecución).** El **orquestador** —ya único agente que escribe `resource_usage` (§3.3)— extiende esa responsabilidad al enforcement:

1. **Soft enforcement (budget-aware prompting).** Al invocar a un subagente, el orquestador le inyecta en el prompt el presupuesto restante de `R` (`R − task_total`). El subagente se autorregula: produce salida concisa cuando queda poco, explora cuando hay holgura.
2. **Hard enforcement.** Tras cada invocación, al recalcular `task_total`, el orquestador recalcula `resource_usage.budget_status`: utilización por dimensión (`task_total / R`), `most_constrained` (la dimensión con mayor utilización) y la señal `enforcement`. Mientras todas las dimensiones < 1, `enforcement = soft_warn`. En cuanto alguna alcanza 1, `enforcement = hard_halt`: el orquestador **detiene el ciclo**, marca `status = violated` y escala al humano. No espera a que el subagente "termine bien".

> Asimetría de cumplimiento (D17; ver DESIGN.md §3.7): el consumo de tokens solo se conoce *después* de cada llamada al modelo, no durante. Por eso el enforcement ocurre en los **límites de invocación** (antes/después de cada subagente), no dentro de una generación. Y como los subagentes son efímeros, no se les "sanciona": simplemente se detiene el ciclo, y la rendición de cuentas se atribuye a la estrategia de asignación del orquestador.

**Estado terminal causal.** El enforcement se refleja en el `status`: una cota rota lleva a `violated` (no a un genérico `failed`), una expiración a `expired`, una cancelación humana a `terminated`, y el cumplimiento de `acceptance_criteria` dentro de R y T a `fulfilled`. Cada contrato alcanza exactamente un estado terminal, lo que permite auditar *por qué* terminó.

---

## 4. Cuándo reset vs compaction (P5)

El arnés distingue dos situaciones de gestión de contexto:

### 4.1 Context reset (sesión terminada)

**Cuándo ocurre:** la sesión del orquestador termina (cierre manual, límite de tokens agotado sin recuperación, interrupción externa).

**Qué ocurre con el estado:** el estado del trabajo persiste en:
- `state.json` — campo de estado de cada bloque/tarea (`drafted`/`active`/`fulfilled`/`violated`/`expired`/`terminated`, alineado con el lifecycle del contrato; D17).
- `progress.md` — última entrada de sesión (qué se hizo, bugs, siguiente paso).
- `tasks/{bloque}-{slug}.json` — contrato con `status` actualizado y `audit_trail` con todas las entradas escritas hasta el momento.
- Artefactos producidos — versionados en el repositorio.

**Cómo retomar el trabajo:** ver la secuencia de re-entrada del protocolo de sesión (D9) en
`core/orchestration/session-protocol.md`. En resumen:
1. El nuevo orquestador arranca desde cero (sin memoria de la sesión anterior).
2. Lee `state.json` para identificar la tarea `active` o, si no hay ninguna, la siguiente `drafted`.
3. Lee la última entrada de `progress.md` para conocer qué se hizo en la sesión anterior.
4. Lee el contrato JSON de la tarea activa para conocer el `status` exacto y el `audit_trail`.
5. Lee los artefactos ya producidos que figuren en `audit_trail`.
6. Continúa el ciclo desde el punto donde se interrumpió, sin necesidad de repetir trabajo ya completado.

### 4.2 Compaction (sesión activa con contexto saturado)

**Cuándo ocurre:** la sesión está activa pero el contexto del orquestador se satura (la ventana de contexto se llena de historia de conversación que ya no es necesaria).

**Qué hacer:**
1. El orquestador descarta la historia de conversación pasada (compacta el contexto).
2. Retiene en contexto solo los artefactos de estado: `state.json`, el contrato JSON activo y la
   última entrada de `progress.md`.
3. Continúa el ciclo sin pérdida de estado funcional.

**Diferencia clave con el reset:** en la compaction la sesión no termina; el orquestador conserva su identidad y puede continuar spawneando subagentes. En el reset, la sesión termina y el próximo orquestador reconstruye el estado desde los ficheros.

### 4.3 Regla de retoma sin ambigüedad

Un orquestador que retoma el trabajo tras un reset no asume qué ocurrió: lee los ficheros de estado. Si `status` es `active` pero no hay entrada de "Artefactos producidos" en el `audit_trail`, el trabajo del implementador no está completo y debe reiniciarse. Si hay entrada de "Artefactos producidos" pero no de "Veredicto emitido", el evaluador no ha actuado y debe invocarse.

---

## 5. Cómo evitar la "declaración de victoria" prematura (P4)

La declaración de victoria prematura ocurre cuando un agente (típicamente el implementador) reporta que ha terminado sin que sus artefactos hayan sido verificados por el evaluador.

Las siguientes reglas mecánicas la impiden:

1. **El ciclo no avanza hasta que el evaluador emite `APTO`.** El orquestador no marca `status: fulfilled` ni actualiza el campo de estado de la tarea en `state.json` a `fulfilled` basándose en el reporte del implementador. Solo lo hace tras recibir el veredicto `APTO` del evaluador.
2. **El implementador no tiene autoridad para marcar criterios como cumplidos.** El campo `acceptance_criteria` del contrato no se modifica por el implementador; solo el evaluador tiene autoridad para pronunciarse sobre su cumplimiento.
3. **El evaluador es un agente distinto con contexto separado.** El evaluador no tiene acceso al historial de sesión del implementador. Lee solo el contrato JSON y los artefactos producidos. Esto impide que la confianza del implementador contamine el juicio del evaluador.
4. **El `audit_trail` como prueba.** Un bloque no puede marcarse como `fulfilled` si el `audit_trail` no contiene una entrada de "Veredicto emitido" con resultado `APTO`. El planificador verifica esta condición antes de actualizar el `status`.
5. **Máximo de reintentos explícito.** Si el evaluador emite `NO APTO` dos veces consecutivas para la misma tarea, el orquestador escala al humano antes de un tercer intento. El ciclo no puede iterar indefinidamente sin intervención humana.

---

## 6. Protocolo de fallback

Cada condición de fallback del contrato tiene un protocolo de activación. Este es el procedimiento estándar cuando cada condición se activa:

### `on_unclear_requirements`

**Quién lo activa:** el implementador, durante la validación pre-handoff o durante la ejecución.

**Procedimiento:**
1. El implementador no produce artefactos parciales.
2. Añade entrada al `audit_trail`: `action: "Fallback: on_unclear_requirements"`, `result: lista de ambigüedades concretas`.
3. Notifica al orquestador con la lista de ambigüedades.
4. El orquestador invoca al planificador con esa lista.
5. El planificador revisa el contrato, actualiza el `purpose` o los `acceptance_criteria` para resolver la ambigüedad, y añade entrada al `audit_trail`.
6. El orquestador reinvoca al implementador con el contrato actualizado.

### `on_blocked`

**Quién lo activa:** el implementador, cuando un obstáculo externo impide el avance.

**Procedimiento:**
1. El implementador documenta el obstáculo con precisión.
2. Añade entrada al `audit_trail`: `action: "Fallback: on_blocked"`, `result: descripción del obstáculo`.
3. El orquestador notifica al humano con el contexto completo (contrato + entrada del audit trail).
4. El bucle se pausa. No se lanza ningún subagente adicional hasta recibir instrucción del humano.
5. El humano decide: resolver el obstáculo, reformular el contrato o cancelar la tarea.

### `on_low_confidence`

**Quién lo activa:** el implementador, al completar los artefactos pero con incertidumbre sobre su corrección.

**Procedimiento:**
1. El implementador produce los artefactos igualmente (no bloquea el trabajo).
2. Añade entrada al `audit_trail`: `action: "Fallback: on_low_confidence"`, `result: descripción de la incertidumbre y los artefactos afectados`.
3. El evaluador recibe la señal y aplica criterio más estricto en su verificación.
4. Si el evaluador emite `NO APTO`, el orquestador escala al humano antes del segundo intento en lugar de reintentar directamente.

### `on_timeout`

**Quién lo activa:** el orquestador, al detectar que un contrato lleva sin avanzar más tiempo del definido por el proyecto.

**Procedimiento:**
1. El orquestador registra el estado actual de los artefactos parciales (si los hay).
2. Añade entrada al `audit_trail`: `action: "Fallback: on_timeout"`, `result: descripción del estado actual y tiempo transcurrido`.
3. Notifica al humano con el contexto suficiente para decidir.
4. El humano decide: continuar, reformular el contrato o cancelar.

### `on_schema_divergence`

**Quién lo activa:** el implementador, durante la validación pre-handoff (`no_schema_divergence_detected: false`).

**Procedimiento:**
1. El implementador no produce artefactos.
2. Añade entrada al `audit_trail`: `action: "Fallback: on_schema_divergence"`, `result: descripción de la divergencia y los campos afectados`.
3. Notifica al orquestador.
4. El orquestador devuelve el contrato al planificador con la lista de divergencias.
5. El planificador reconcilia el contrato con el esquema oficial y lo devuelve al orquestador.
6. El orquestador reinvoca al implementador con el contrato corregido.

---

## 7. Resumen de responsabilidades por agente

| Agente          | Crea contrato | Confirma pre-handoff | Escribe audit trail       | Escribe `execution_summary` | Lee `execution_summary` | Escribe `resource_usage` + enforcement | Actualiza status        | Marca tarea `fulfilled` en `state.json` |
|-----------------|:-------------:|:--------------------:|:-------------------------:|:----------------------------:|:------------------------:|:------------------------:|:-----------------------:|:---------------------------------------:|
| orquestador     | —             | —                    | Al invocar implementador  | —                             | —                        | Sí (tras cada invocación; hard/soft enforcement de `governance.R`, D17) | drafted → active; → violated si rompe cota   | —                        |
| planificador    | Sí (incl. `governance`) | —          | Al crear; al cerrar       | —                             | —                        | Inicializa vacío (al crear) | active → fulfilled/violated/expired | Sí            |
| navegador       | —             | —                    | —                         | —                             | —                        | —                        | —                       | —                        |
| implementador   | —             | Sí                   | Al validar; al producir   | Sí (al terminar)              | —                        | —                        | —                       | —                        |
| evaluador       | —             | Sí                   | Al emitir veredicto       | —                             | Sí (como entrada, junto a `acceptance_criteria`) | — | — | —                        |
| humano          | —             | —                    | Al intervenir (escaladas) | —                             | —                        | —                        | —                       | —                        |

> Nota: el handoff implementador → evaluador es bidireccional vía el contrato (sección 1.1): el
> implementador escribe `execution_summary` y el evaluador lo lee directamente del JSON, sin
> handoff verbal del orquestador.
