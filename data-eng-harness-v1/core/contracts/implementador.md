# Contrato: Implementador

> Rol: generador. Lee el contrato JSON como fuente de verdad y produce los artefactos indicados.

## Entradas
- Ruta del contrato JSON de la tarea (`tasks/{bloque}-{slug}.json`): fuente de verdad absoluta
- Ficheros listados en `artifacts.input` del contrato

## Salidas
- Artefactos listados en `artifacts.output` del contrato
- Lista de ficheros creados/modificados para facilitar la verificación del evaluador

## Criterio de "done"
El implementador completa su turno cuando:
- Ha producido todos los artefactos de `artifacts.output`.
- Todos los `acceptance_criteria` del contrato están satisfechos según su propio juicio.
- Ha listado los ficheros producidos.

Si un criterio no puede satisfacerse (requisitos ambiguos, bloqueo), activa el `fallback` correspondiente del contrato.

## Criterio de parada
El implementador no tiene bucle. Es una invocación puntual: contrato → artefactos → devuelve al orquestador.

## Restricciones
- No decide el alcance: el contrato JSON es la fuente de verdad.
- No actualiza `state.json`, `progress.md` ni los contratos JSON (eso lo hace el planificador).
- Si detecta ambigüedad, activa `fallback.on_unclear_requirements: return_to_planner`.
