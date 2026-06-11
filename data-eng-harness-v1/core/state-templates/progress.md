# Progress log — {PROYECTO}

> Notas de sesión, append-only (D10, hard_spec.md §5).
> **Cada sesión añade una entrada nueva al final, nunca reescribe entradas anteriores.**
> Si una entrada anterior resultó incorrecta, añade una entrada de corrección que la referencie; no la edites ni la borres.
>
> Este fichero es la "memoria de trabajo" desechable del arnés (rol de `fix_plan.md`, hard_spec.md §5
> D10-b): si se pierde o se corrompe, es regenerable inspeccionando `state.json`, `git log` y los
> contratos en `tasks/`. La especificación estable (qué/por qué) vive en `hard_spec.md`, no aquí.

---

## Cómo añadir una entrada

Al cierre de cada sesión (ver protocolo de sesión, `core/orchestration/session-protocol.md`), el
`planificador` añade una entrada nueva al final con esta estructura mínima:

```markdown
## YYYY-MM-DD — {task_id}: {resumen corto de la sesión}

- **Qué se hizo**: {resumen de lo producido/modificado en esta sesión}
- **Veredicto del evaluador**: APTO / NO APTO — {resumen de defectos si NO APTO}
- **Bugs / hallazgos**: {observaciones relevantes para sesiones futuras, o "ninguno"}
- **Estado tras esta sesión**: {task_id} → {pending|in_progress|complete|failed} en `state.json`
- **Siguiente paso**: {qué debe hacer la próxima sesión, o "ninguno — bloque cerrado"}
```

> Nota: no hay un límite de longitud impuesto, pero cada entrada debe ser legible en una pasada por
> el humano durante el checkpoint (P2, economía de contexto). Si una sesión generó mucho detalle,
> resume aquí y enlaza a los artefactos/commits concretos en vez de copiar su contenido.

---

## Entradas

### YYYY-MM-DD — {task_id}: {resumen corto de la primera sesión}

- **Qué se hizo**: {...}
- **Veredicto del evaluador**: {...}
- **Bugs / hallazgos**: {...}
- **Estado tras esta sesión**: {...}
- **Siguiente paso**: {...}
