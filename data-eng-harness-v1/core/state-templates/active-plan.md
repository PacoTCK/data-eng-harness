# Plan activo — retirado (D10)

> **Nota de migración (B10):** esta plantilla queda **retirada**. Su rol —llevar el estado de un
> bloque (tareas, su estado `[ ]`/`[~]`/`[x]`, decisiones tomadas)— queda subsumido por la capa de
> estado de D10 (hard_spec.md §5):
>
> - El **índice de bloques/tareas y su campo de estado** vive en `state-templates/state.json`
>   (`pending`/`in_progress`/`complete`/`failed`).
> - Las **decisiones tomadas durante el bloque** y las **notas/observaciones append-only** que antes
>   vivían en este fichero se registran como entradas de sesión en `state-templates/progress.md`.
> - El **objetivo, criterio de aceptación y agentes involucrados** de cada bloque ya no se duplican
>   en una plantilla de proyecto: viven en `hard_spec.md` (única fuente de qué/por qué, D10).
>
> No copies este fichero a `state/{BLOQUE_ID}-active-plan.md` en proyectos nuevos. Usa
> `state-templates/state.json` y `state-templates/progress.md`.
