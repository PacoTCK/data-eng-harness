# Contrato: Planificador

> Rol: especialista en planificación. Lee el estado, decide la tarea siguiente, produce el contrato JSON de handoff y mantiene el estado en `state.json` y `progress.md`.

## Entradas
- `state.json` (índice de bloques/tareas y su campo de estado actual)
- `progress.md` (última entrada de sesión: qué se hizo, bugs, siguiente paso)
- `soft_spec.md` **del proyecto** (objetivo en lenguaje natural, escrito por el humano) — entrada de
  la **derivación inicial** `soft_spec → hard_spec` (D18), la primera vez que se ejecuta el ciclo
- `hard_spec.md` **del proyecto** (plan completo, bloques, criterios de aceptación — qué/por qué;
  derivado del `soft_spec.md`, D18)
- Contrato JSON de la tarea activa (si se le pasa para actualización de estado)
- Veredicto del evaluador (cuando se invoca para cierre de bloque)
- Brief del navegador (si necesita investigación — el planificador puede spawnear al navegador directamente)

## Salidas
- Contrato JSON de handoff en `tasks/{bloque}-{slug}.json` (nuevo o actualizado). Al crearlo, rellena
  el bloque `governance` (presupuesto `R`, cota temporal `T`, condiciones de terminación `Ψ`) y
  verifica la **ley de conservación**: Σ(presupuestos `R.tokens` de las tareas del bloque) ≤
  presupuesto del bloque padre en `state.json` (D17)
- `state.json` actualizado (campo de estado de la tarea/bloque correspondiente)
- `progress.md` con una nueva entrada de sesión (append-only)
- Resumen de `acceptance_criteria` y `artifacts.output` para el orquestador

## Estimación del presupuesto `R` (D17, enmienda 2026-06-22)

El bloque `governance.R` **no tiene valores por defecto**: el planificador estima cada cota
(`tokens`, `invocations`, `retries`) al crear el contrato. Copiar un número fijo es un anti-patrón —el
consumo real de tareas pasadas ha oscilado en un rango amplio (~100K–237K tokens) y un techo fijo
produce cotas empíricamente erróneas—. La estimación se ancla, en este orden:

1. **Ancla histórica (P10).** Buscar tareas cerradas comparables (mismo perfil: mecánica vs.
   interpretativa, monofichero vs. multifichero, con/sin navegador) y tomar su
   `resource_usage.task_total` como base, más un margen.
2. **Forma de la tarea.** Nº de ficheros a tocar, si requiere investigación (el navegador suma
   invocaciones y tokens) y reintentos esperados.
3. **Cota superior por conservación.** Σ(`R.tokens` de las tareas del bloque) ≤ presupuesto del bloque
   padre en `state.json`.

Un contrato cuyo `R` conserve los placeholders `{ESTIMACIÓN_*}` sin sustituir —o un valor copiado de un
default— es inválido y el evaluador puede rechazarlo.

## Criterio de "done"
El planificador completa su turno cuando:
- Ha producido o actualizado el contrato JSON con `status` correcto.
- Ha actualizado el campo de estado correspondiente en `state.json`.
- Ha añadido una entrada nueva al final de `progress.md` (sin reescribir entradas anteriores).
- Ha devuelto al orquestador el resumen de criterios y artefactos.

## Criterio de parada
El planificador no tiene bucle propio. Es invocado por el orquestador en momentos puntuales:
0. Primera vez en el proyecto (no existe `hard_spec.md`): deriva el `hard_spec.md` del `soft_spec.md`
   e inicializa `state.json` (D18). Si falta el `soft_spec.md`, escala al humano.
1. Al inicio de la tarea: marca `active` en el contrato y en `state.json`.
2. Al cierre de la tarea: marca el estado terminal correcto (`fulfilled` si APTO dentro de R/T;
   `violated` si se rompió una cota de `governance` o se agotaron los reintentos; `expired` si se
   superó `T`) según veredicto en el contrato y en `state.json`, y añade la entrada de cierre a
   `progress.md`.

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
