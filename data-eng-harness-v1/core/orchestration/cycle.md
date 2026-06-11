# Patrón de orquestación: ciclo de 4 agentes

> Definición model-agnostic del bucle de trabajo. El adaptador de proveedor correspondiente (en `adapters/`) implementa este patrón con las herramientas concretas disponibles.
>
> Este documento describe el **ciclo de 4 agentes** (qué ocurre dentro de una sesión, una vez
> identificada la tarea). La secuencia completa de sesión —re-entrada antes del ciclo y cierre
> después— está formalizada en `session-protocol.md` (D9, hard_spec.md §3.1.1).

## Diagrama del ciclo (incluye el cierre de sesión)

```
[secuencia de re-entrada — ver session-protocol.md]
  │
  ▼
orquestador (hilo principal)
  │
  ├─► planificador
  │     ├─► [si necesita investigación] navegador → brief → planificador
  │     ├─► produce / actualiza tasks/{bloque}-{slug}.json (status: in_progress)
  │     └─► actualiza state.json (tarea: in_progress)
  │
  ├─► implementador  (recibe ruta del contrato JSON)
  │     └─► produce artefactos de artifacts.output
  │
  ├─► evaluador      (recibe ruta del contrato + lista de artefactos)
  │     └─► emite veredicto APTO / NO APTO
  │
  └─► planificador   (recibe veredicto)
        ├─► si APTO    → status: complete; actualiza state.json (tarea: complete)
        └─► si NO APTO → tarea permanece in_progress; reintento (máx. 2, D13) sobre el working
                          tree; agotados → git reset --hard al último commit verificado y status:
                          failed (ver "Política de reintentos y recuperación con git (D13)" en
                          session-protocol.md)
  │
  ▼
cierre de sesión
  │
  ├─► commit (SOLO si el evaluador dio APTO; "una tarea verificada = un commit", D13(a))
  ├─► actualizar state.json + progress.md (nueva entrada de sesión)
  ├─► checkpoint (según política activa, D12 — ver session-protocol.md)
  └─► limpieza de contexto (p.ej. /clear) → fin de sesión
```

> El bucle **no se repite dentro de la misma sesión**: lo cierra, según la política de checkpoint
> activa (D12), o bien el **humano** (`validacion-por-tarea`, modo por defecto: revisa el
> checkpoint y relanza una sesión fresca), o bien el **mecanismo de continuación** de `full-auto`
> (sin checkpoint humano intermedio, salvo checkpoint duro). Ambas políticas repiten la misma
> secuencia de re-entrada (regla "una tarea por sesión", D9 — ver `session-protocol.md`).

## Regla fundamental: una tarea por iteración (P4)

Cada vuelta del bucle procesa **exactamente una tarea**: la que el planificador identifica como el siguiente ítem pendiente (`pending`) o en curso (`in_progress`) en `state.json`. El orquestador no avanza a la siguiente tarea hasta que el evaluador emita `APTO` para la actual.

Esta regla evita:
- Que el agente "declare victoria" antes de que la tarea esté verificada.
- Que múltiples tareas abiertas simultáneamente saturen el contexto.
- Que los artefactos de una tarea contaminen el contexto de la siguiente.

> Nota: esta regla "una vuelta = una tarea" dentro del ciclo de 4 agentes es el mismo principio que
> la regla de sesión "una tarea por sesión" (D9): en este protocolo, cada sesión ejecuta como mucho
> una vuelta del ciclo.

## Protocolo de handoff entre sesiones (P5)

Cuando una sesión termina (cierre de sesión normal, límite de contexto, interrupción), el estado del trabajo persiste en:
- `state.json` — campo de estado de cada bloque/tarea (`pending`/`in_progress`/`complete`/`failed`).
- `progress.md` — última entrada de sesión (qué se hizo, bugs, siguiente paso).
- `tasks/{bloque}-{slug}.json` — contrato con `status` actualizado y `audit_trail` (si existe).
- Artefactos producidos — versionados en el repositorio.

Al inicio de la siguiente sesión, el orquestador ejecuta la secuencia de re-entrada de
`session-protocol.md`: lee `state.json` para identificar la tarea `in_progress` o, si no hay
ninguna, la siguiente `pending`, lee la última entrada de `progress.md` y retoma el trabajo sin
depender de la memoria de la sesión anterior.

**Reset vs compaction:** si el contexto está saturado pero la sesión puede continuar (no se ha
llegado al cierre de la tarea), el orquestador compacta (descarta historia vieja, retiene solo el
estado persistido) — ver `state-templates/handoff-protocol.md` §4.2. Si la sesión termina (cierre
de sesión completo), la siguiente sesión arranca desde cero leyendo los ficheros de estado.

## Separación generador / evaluador (P6)

El implementador y el evaluador son agentes distintos invocados en sesiones separadas (o al menos con contextos separados):
- El implementador NO tiene acceso al historial de la sesión del evaluador.
- El evaluador NO tiene acceso al historial de la sesión del implementador.
- El único canal entre ellos es el contrato JSON (criterios de aceptación) y los artefactos producidos.

Esta separación impide la autoevaluación, que estadísticamente sesga hacia `APTO` incluso cuando los criterios no se cumplen.

## Human-on-the-loop (P13)

El humano no ejecuta cada paso, pero interviene en los puntos de control definidos en el contrato del orquestador:
- Inicio de bloque: aprueba o redirige la propuesta del planificador.
- Segundo intento fallido: decide si continuar, reformular el contrato o pausar.
- Escalada explícita (`fallback.on_blocked: escalate_to_human`).
- Cierre de sesión: bajo `validacion-por-tarea` (modo por defecto, D12), revisa el checkpoint
  completo en cada tarea; bajo `full-auto`, solo interviene cuando se activa un **checkpoint duro**
  (operación DDL/DML destructiva o sobre sistemas de cliente, o modificación de `hard_spec.md` —
  ver `session-protocol.md` y `stop-conditions.md`).

El taste del humano se captura una vez (en los contratos, los criterios de aceptación y las notas de progreso) y se aplica continuamente sin requerir su presencia en cada micro-decisión.

## Criterio de parada del bucle

El bucle termina cuando se cumple una de estas condiciones:
1. **Todos los bloques/tareas de `state.json` están en `complete`** — proyecto completado.
2. **Instrucción explícita del humano** — el humano detiene el bucle manualmente (en el checkpoint
   de cierre de sesión, ver `session-protocol.md`).
3. **Escalada sin resolución** — un bloque llega a 2 intentos fallidos y el humano decide no continuar.
