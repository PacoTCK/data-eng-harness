# Contrato: Planificador

> Rol: especialista en planificación. Lee el estado, decide la tarea siguiente, produce el contrato JSON de handoff y mantiene el estado en `state.json` y `progress.md`.

## Entradas
- `state.json` (índice de bloques/tareas y su campo de estado actual)
- `progress.md` (última entrada de sesión: qué se hizo, bugs, siguiente paso)
- `hard_spec.md` (plan completo, bloques, criterios de aceptación — qué/por qué)
- Contrato JSON de la tarea activa (si se le pasa para actualización de estado)
- Veredicto del evaluador (cuando se invoca para cierre de bloque)
- Brief del navegador (si necesita investigación — el planificador puede spawnear al navegador directamente)

## Salidas
- Contrato JSON de handoff en `tasks/{bloque}-{slug}.json` (nuevo o actualizado)
- `state.json` actualizado (campo de estado de la tarea/bloque correspondiente)
- `progress.md` con una nueva entrada de sesión (append-only)
- Resumen de `acceptance_criteria` y `artifacts.output` para el orquestador

## Criterio de "done"
El planificador completa su turno cuando:
- Ha producido o actualizado el contrato JSON con `status` correcto.
- Ha actualizado el campo de estado correspondiente en `state.json`.
- Ha añadido una entrada nueva al final de `progress.md` (sin reescribir entradas anteriores).
- Ha devuelto al orquestador el resumen de criterios y artefactos.

## Criterio de parada
El planificador no tiene bucle propio. Es invocado por el orquestador en momentos puntuales:
1. Al inicio de la tarea: marca `in_progress` en el contrato y en `state.json`.
2. Al cierre de la tarea: marca `complete` o `failed` según veredicto en el contrato y en
   `state.json`, y añade la entrada de cierre a `progress.md`.

## Restricciones
- No invoca al implementador ni al evaluador (eso es responsabilidad del orquestador).
- No decide ni aplica la política de checkpoint (`validacion-por-tarea` / `full-auto`, D12): esa
  política es de configuración del proyecto y la aplica el orquestador (ver
  `core/contracts/orchestrator.md` y `core/orchestration/session-protocol.md`).
- Puede invocar al navegador directamente cuando necesita investigación.
- Es el único agente que modifica el campo de estado en `state.json` y que añade entradas a
  `progress.md`.
- `state.json` solo admite la mutación de su campo de estado: el planificador no añade ni reescribe
  definiciones, objetivos o criterios (esos viven en `hard_spec.md`).
- No lee los documentos fuente directamente si puede delegarlo al navegador.
