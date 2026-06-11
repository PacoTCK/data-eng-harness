# Condiciones de parada del orquestador

> Complemento a `cycle.md` y `session-protocol.md`. Define cuándo el orquestador para, espera o
> escala, y cómo esa parada se refleja en el cierre de sesión (D9).

## Parada normal
- Todos los bloques/tareas de `state.json` están en estado `complete` — proyecto completado.

## Parada por intervención humana
- El humano envía una instrucción explícita de detener (en cualquier momento del ciclo, incluido el
  checkpoint de cierre de sesión — ver `session-protocol.md`).
- El orquestador persiste el estado actual antes de parar: `state.json` (campo de estado de la tarea
  activa) + `progress.md` (entrada de sesión con el motivo de la parada) + contrato JSON actualizado.
- La sesión sigue la secuencia de cierre normal (sin commit si el evaluador no dio `APTO`) antes de
  terminar.

## Parada por checkpoint duro (invariante del protocolo, D12)
Se produce cuando la tarea activa implica:
1. Una **operación DDL/DML destructiva, o que actúa contra sistemas de un cliente**.
2. **Cualquier modificación de `hard_spec.md`**.

Este tipo de parada es **independiente de la política de checkpoint** (`validacion-por-tarea` o
`full-auto`, ver `session-protocol.md` § "Política de checkpoint (D12)"): ocurre **siempre**,
incluso si el proyecto está corriendo en `full-auto` y no hay checkpoint humano programado entre
tareas. El orquestador trata un checkpoint duro como una escalada (ver "Parada por escalada" abajo)
y no continúa el bucle —ni siquiera para encadenar la siguiente tarea en `full-auto`— hasta recibir
aprobación humana explícita.

## Parada por NO APTO y recuperación con git (D13)

Cuando el `evaluador` emite `NO APTO`, no hay parada inmediata: la tarea permanece `in_progress` y
los artefactos producidos quedan en el working tree (no se commitean ni se descartan), de modo que
la siguiente sesión reintenta a partir de ellos con los defectos del evaluador anotados en el
contrato y en `progress.md` (ver "Secuencia de cierre" en `session-protocol.md`).

- **Límite de 2 reintentos por tarea.** Una tarea puede recibir como máximo 2 veredictos `NO APTO`
  consecutivos.
- **Tras agotar los reintentos**, el orquestador ejecuta `git reset --hard` al **último commit
  verificado** (D13(b)) — descartando los artefactos sin confirmar de los reintentos fallidos — y
  la tarea vuelve al `planificador` con los defectos anotados en el contrato. Esto es lo que activa
  la "Parada por escalada" descrita abajo (condición 1).
- **Por qué elimina el deadlock.** Esta política garantiza que el bucle de `NO APTO` no se repite
  indefinidamente sobre un working tree degradado: el peor caso siempre vuelve a un estado conocido
  y verificado, desde el que el planificador puede reformular la tarea o escalar al humano.

> Detalle completo de esta política en "Política de reintentos y recuperación con git (D13)" de
> `session-protocol.md`.

## Parada por escalada
Se produce cuando:
1. Un bloque recibe veredicto `NO APTO` por segunda vez consecutiva (agotando el límite de 2
   reintentos de D13; el orquestador ya ha ejecutado `git reset --hard` al último commit
   verificado, ver arriba).
2. Un contrato activa `fallback.on_blocked: escalate_to_human`.
3. El implementador activa `fallback.on_unclear_requirements: return_to_planner` y el planificador no puede resolver la ambigüedad.
4. La tarea activa implica un **checkpoint duro** (ver arriba), en cualquier política de checkpoint.

En estos casos, el orquestador:
1. Persiste el estado: contrato JSON con `status: failed` o `in_progress`, y `state.json` con el
   campo de estado de la tarea reflejando `in_progress` o `failed` según corresponda.
2. Añade una entrada a `progress.md` describiendo el bloqueo y la decisión pendiente (parte del
   cierre de sesión, ver `session-protocol.md`).
3. Informa al humano del punto exacto de bloqueo y qué decisión necesita (este es el checkpoint
   humano de la secuencia de cierre).
4. Espera instrucción antes de continuar — no se arranca una sesión nueva hasta que el humano decida.

## Señales de "declaración de victoria" prematura (anti-pattern, P4)
El orquestador detecta y rechaza declaraciones de victoria cuando:
- El implementador declara "done" pero el evaluador no ha emitido veredicto.
- El planificador marca `[x]` sin veredicto `APTO` previo del evaluador.
- Un artefacto de `artifacts.output` no existe físicamente en el repo.
