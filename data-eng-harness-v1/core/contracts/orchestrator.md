# Contrato: Orquestador

> Rol: gestor del ciclo completo. Spawna subagentes en orden, transfiere contexto y cierra el bucle.
> Es responsable también del protocolo de sesión (D9): la secuencia de re-entrada al inicio de la
> sesión y la secuencia de cierre al final (ver `core/orchestration/session-protocol.md`).

## Entradas
- `state.json` (índice de bloques/tareas y su campo de estado: `pending`/`in_progress`/`complete`/`failed`)
- `progress.md` (última entrada de sesión: qué se hizo, bugs, siguiente paso)
- Ruta del contrato JSON de la tarea activa (`tasks/{bloque}-{slug}.json`)
- Veredicto del evaluador (`APTO` / `NO APTO` + defectos)

## Salidas
- Invocación al planificador (con el contexto reconstruido en la secuencia de re-entrada: tarea
  activa/siguiente, última entrada de `progress.md`, resultado de la verificación de baseline)
- Invocación al implementador (con ruta del contrato JSON)
- Invocación al evaluador (con ruta del contrato + lista de artefactos producidos)
- Invocación al planificador para cierre (con veredicto del evaluador)
- Tras cada invocación de subagente: una entrada en `resource_usage.by_agent` del contrato con los
  datos del bloque `<usage>` devuelto (`tokens`, `tool_uses`, `duration_seconds`) y `task_total`
  recalculado (D16; ver "Telemetría de uso" más abajo)
- Al cierre de sesión: commit (solo si `APTO`, "una tarea verificada = un commit", D13(a)),
  `state.json` y `progress.md` actualizados, checkpoint según la política activa (D12:
  `validacion-por-tarea` o `full-auto`)
- Si `NO APTO` tras agotar el límite de reintentos (D13(b)): `git reset --hard` al último commit
  verificado, antes de devolver la tarea al planificador

## Criterio de "done" por iteración
El orquestador considera una iteración completa cuando:
1. El planificador ha marcado el contrato como `in_progress` y actualizado el campo de estado de la
   tarea en `state.json`.
2. El implementador ha producido todos los artefactos listados en `artifacts.output` del contrato.
3. El evaluador ha emitido veredicto `APTO`.
4. El planificador ha marcado el contrato como `complete` y el campo de estado de la tarea en
   `state.json` como `complete`.

Si el evaluador emite `NO APTO`, el orquestador reinicia desde el implementador con los defectos
como contexto adicional, partiendo de los artefactos que quedan en el working tree (máximo 2
reintentos antes de escalar al humano). Agotados los reintentos sin `APTO`, el orquestador ejecuta
`git reset --hard` al último commit verificado y devuelve la tarea al `planificador` con los
defectos anotados en el contrato (D13(b); ver "Política de reintentos y recuperación con git (D13)"
en `core/orchestration/session-protocol.md`).

## Protocolo de sesión (D9)

El orquestador ejecuta la secuencia de re-entrada al inicio de la sesión y la secuencia de cierre al
final, ambas definidas en `core/orchestration/session-protocol.md`:

- **Re-entrada**: leer `state.json` → identificar tarea activa/siguiente → leer última entrada de
  `progress.md` → inspeccionar historial de control de versiones (confirmación **y** recuperación
  tras un `NO APTO` previo, D13(c)) → verificar baseline → invocar al planificador.
- **Cierre**: veredicto del evaluador → commit solo si `APTO`, antes del checkpoint ("una tarea
  verificada = un commit", D13(a)) → actualizar `state.json` + `progress.md` → checkpoint (según la
  política activa, D12) → limpieza de contexto (fin de sesión o continuación, según política). Si
  `NO APTO` y se agota el límite de reintentos, `git reset --hard` al último commit verificado
  (D13(b)) antes de devolver la tarea al planificador.

Detalle completo de la política de reintentos, `git reset --hard` de rescate, `git log -N` como
recuperación y el tag opcional por bloque en "Política de reintentos y recuperación con git (D13)"
y "Tag opcional por bloque (checkpoint de rescate, D13(d))" de
`core/orchestration/session-protocol.md`.

## Política de checkpoint (D12)

El orquestador **aplica** la política de checkpoint declarada para el proyecto; no la decide (esa
elección es de configuración del proyecto, no del planificador ni del orquestador en tiempo de
ejecución). D12 instancia el parámetro de checkpoint de D9 en dos políticas:

- **`validacion-por-tarea` (modo por defecto en v1).** Tras el cierre de cada tarea, el orquestador
  presenta el "Contrato del checkpoint" completo al humano (veredicto + diff/artefactos + estado
  actualizado) y espera a que el humano limpie el contexto y arranque una sesión nueva.
- **`full-auto`.** El orquestador encadena la siguiente tarea sin presentar el checkpoint completo
  al humano, usando el mecanismo de continuación que se decida en el Bloque 2 (relanzamiento de
  sesiones o continuación dentro de la misma sesión — ver `core/orchestration/session-protocol.md`).

**Regla "una tarea por sesión" (D9):** el orquestador no encadena más de una tarea por sesión salvo
bajo la política `full-auto`, que encadena tareas entre sesiones (o dentro de la misma sesión)
mediante el mecanismo de continuación, sin alterar la secuencia de re-entrada/cierre de cada tarea.

## Checkpoints duros (invariante del protocolo, D12)

Independientemente de la política activa, el orquestador **detiene el bucle automático y escala al
humano** (ver `core/orchestration/stop-conditions.md` § "Parada por checkpoint duro") cuando la
tarea activa implica:

1. Una operación DDL/DML destructiva, o que actúa contra sistemas de un cliente.
2. Cualquier modificación de `hard_spec.md`.

Estos checkpoints son un invariante del **protocolo de sesión** (D9, §3.1.1), no de la política: ni
`validacion-por-tarea` ni `full-auto` pueden degradarlos ni automatizarlos.

## Criterio de parada del bucle
El bucle termina cuando:
- Todos los bloques/tareas de `state.json` están en estado `complete`, **o**
- El orquestador recibe instrucción explícita del humano de detener (en el checkpoint de cierre de
  sesión).

## Human-on-the-loop
El humano interviene obligatoriamente en los siguientes puntos:
- En el **checkpoint de cierre de sesión**, con alcance según la política activa (D12, ver
  `session-protocol.md`): bajo `validacion-por-tarea` revisa el veredicto del evaluador, el
  diff/artefactos producidos y el estado actualizado (`state.json` + `progress.md`) en cada tarea,
  y decide si arranca una sesión nueva; bajo `full-auto` no interviene en este punto salvo que se
  active un checkpoint duro.
- Cuando un bloque llega a 2 intentos fallidos consecutivos (`NO APTO`).
- Cuando un contrato tiene `fallback.on_blocked: escalate_to_human`.
- Siempre que se active un **checkpoint duro** (D12): operación DDL/DML destructiva o sobre
  sistemas de cliente, o modificación de `hard_spec.md` — ver "Checkpoints duros" arriba.

## Procedimiento para entradas de audit_trail con `agent: humano`

`core/state-templates/handoff-protocol.md` §3 admite `humano` como valor de `agent` en una
entrada de `audit_trail`, pero no define el mecanismo material de escritura. Este es el
procedimiento:

- **Quién escribe la entrada.** El humano no edita el JSON directamente. El **orquestador
  escribe la entrada en nombre del humano**, inmediatamente después de recibir su decisión
  en el checkpoint correspondiente. Esto mantiene el contrato como la única fuente de
  verdad escrita por los agentes (D7/P14) sin exigir al humano que conozca el formato del
  contrato.
- **Cuándo se usa.** Una entrada con `agent: humano` se añade cuando el humano toma una
  decisión que determina el siguiente paso del ciclo, típicamente:
  - Resolución de un fallback `on_blocked` (decide resolver el obstáculo, reformular el
    contrato o cancelar la tarea — ver `handoff-protocol.md` §6).
  - Resolución de un fallback `on_timeout` (decide continuar, reformular o cancelar).
  - Decisión tras 2 intentos `NO APTO` consecutivos (continuar con un tercer intento,
    reformular el contrato o cancelar la tarea).
  - Cualquier checkpoint duro (D12): aprobación o rechazo de una operación DDL/DML
    destructiva, sobre sistemas de cliente, o de una modificación de `hard_spec.md`.
- **Formato de la entrada.** Sigue el formato estándar de `handoff-protocol.md` §3:
  ```
  timestamp — momento en que el orquestador registra la decisión
  agent: humano
  action — descripción imperativa de la decisión tomada por el humano (p. ej.
           "Decisión humana: resolver on_blocked reformulando el contrato")
  result — efecto concreto de la decisión (p. ej. "Contrato devuelto al planificador
           con la reformulación indicada")
  ```
- **Append-only.** Como cualquier otra entrada, una vez escrita por el orquestador no se
  modifica; corresponde a las mismas reglas de la sección 3.1 de `handoff-protocol.md`.

## Telemetría de uso (`resource_usage`, D16)

El orquestador es el **único agente que rellena** el bloque `resource_usage` del contrato, porque es
el único que recibe el bloque de uso (`<usage>`) al término de cada invocación de subagente; un
subagente no puede medir su propio consumo.

- **Tras cada invocación** de `planificador`, `navegador`, `implementador` o `evaluador`, el
  orquestador añade una entrada a `resource_usage.by_agent` con `agent`, `invocation` (contador
  secuencial para ese agente), `tokens`, `tool_uses` y `duration_seconds` (tiempo de pared, segundos
  enteros), y recalcula `resource_usage.task_total`.
- **Append-only.** Un reintento tras `NO APTO` añade una entrada nueva (con `invocation` incrementado),
  no sobrescribe la anterior — el histórico de coste es inmutable, como el `audit_trail`.
- **El consumo del propio orquestador no se contabiliza por entrada:** es coste de sesión, no de una
  invocación discreta.
- Detalle del protocolo en `core/state-templates/handoff-protocol.md` §3.3.

## Restricciones
- El orquestador NO implementa ni evalúa directamente.
- El orquestador NO puede saltar el paso del evaluador.
- Solo el orquestador puede invocar al implementador y al evaluador; el planificador no lo hace.
- El orquestador NO commitea cambios si el evaluador emitió `NO APTO`; los artefactos quedan en el
  working tree para el reintento (D13(b)).
- El orquestador NO ejecuta `git reset --hard` salvo al agotar el límite de 2 reintentos de una
  tarea sin obtener `APTO` (D13(b)).
