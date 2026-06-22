# Plantillas de estado

Capa de estado del arnés (D10; ver DESIGN.md §9): separa la especificación (el `hard_spec.md` del
proyecto, qué/por qué, casi inmutable) del estado de avance (`state.json`, mutación de alta
frecuencia) y de las notas de sesión (`progress.md`, append-only).

## Inventario

| Fichero                 | Rol                                                                                  | Estado      |
|--------------------------|--------------------------------------------------------------------------------------|-------------|
| `state.json`             | Índice de bloques/tareas con su campo de estado (`drafted`/`active`/`fulfilled`/`violated`/`expired`/`terminated`, D17) y presupuesto `R` por bloque. Única mutación frecuente: el campo de estado. | **Vigente (D10)** |
| `progress.md`            | Notas de sesión append-only: qué se hizo, veredicto, bugs, siguiente paso.            | **Vigente (D10)** |
| `task-contract.json`     | Plantilla de contrato de handoff (Structured Contract, D7/P14) para `tasks/{bloque}-{slug}.json`. | Vigente |
| `handoff-protocol.md`    | Reglas que rigen el uso de las plantillas anteriores: validación pre-handoff, audit trail, fallback. | Vigente |
| `dudas.md`               | Backlog de gaps y preguntas abiertas que requieren confirmación externa.              | Vigente |
| `active-plan.md`         | Plantilla retirada — su rol queda subsumido por `state.json` + `progress.md`.         | **Retirado (D10)** |
| `progress-log.md`        | Plantilla retirada — su rol queda subsumido por `progress.md`.                        | **Retirado (D10)** |

## Capa de estado canónica (D10)

La capa de estado canónica del arnés son **dos ficheros**:

1. **`state.json`** — índice de bloques/tareas con un campo de estado por entrada. Las definiciones,
   objetivos y criterios de aceptación de cada bloque/tarea viven en el `hard_spec.md` del proyecto;
   `state.json` solo referencia los IDs.
2. **`progress.md`** — notas de sesión append-only.

Cualquier proyecto que use este arnés copia `state.json` y `progress.md` a su raíz (o a la carpeta
de estado que defina su `CLAUDE.md`) y los rellena durante el ciclo. `active-plan.md` y
`progress-log.md` no se copian a proyectos nuevos: se mantienen en este directorio únicamente como
nota de migración para quien venga de una versión anterior del arnés.

## Resto de plantillas

- **`task-contract.json`** — contrato de handoff por tarea (Structured Contract). Se copia a
  `tasks/{bloque_id}-{slug}.json` por cada tarea.
- **`handoff-protocol.md`** — documento de referencia (no rellenable) con las reglas de validación
  pre-handoff, audit trail, reset/compaction y fallback que rigen el uso de `state.json`,
  `progress.md` y `task-contract.json`.
- **`dudas.md`** — backlog de dudas abiertas (gaps, preguntas al cliente o al equipo).
