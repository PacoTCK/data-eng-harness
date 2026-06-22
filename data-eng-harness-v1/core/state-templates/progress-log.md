# Log de progreso — retirado (D10)

> **Nota de migración (B10):** esta plantilla queda **retirada**. Su rol —registro append-only de
> "quién hizo qué y cuándo" durante el ciclo de un bloque— queda subsumido por
> `state-templates/progress.md` (D10; ver DESIGN.md §9):
>
> - Cada sesión añade una entrada al final de `progress.md` con qué se hizo, el veredicto del
>   evaluador, bugs/hallazgos y el siguiente paso (ver la sección "Cómo añadir una entrada" de
>   `progress.md`).
> - El **estado resultante por tarea** (`status`) se refleja en `state-templates/state.json`, no en
>   una tabla de entradas de log.
> - El audit trail detallado de cada tarea (quién confirmó pre-handoff, quién escribió el veredicto,
>   etc.) sigue viviendo en el campo `audit_trail` del contrato JSON de la tarea
>   (`state-templates/task-contract.json`), que no cambia con esta migración.
>
> No copies este fichero a `state/{BLOQUE_ID}-progress-log.md` en proyectos nuevos. Usa
> `state-templates/progress.md` para las notas de sesión y `state-templates/state.json` para el
> estado de bloques/tareas.
