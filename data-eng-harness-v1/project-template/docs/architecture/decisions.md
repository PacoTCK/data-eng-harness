# Registro de decisiones de arquitectura (ADRs)

> Propósito: registrar las decisiones de arquitectura significativas de este proyecto,
> su contexto, las opciones consideradas y la justificación de la elección.
> Append-only: nunca se elimina una entrada; las decisiones superadas se marcan como
> `SUPERSEDED BY ADR-NNN`.

---

## Cómo rellenar este fichero

1. Cuando el equipo tome una decisión de arquitectura no trivial, añade una entrada al final.
2. El ID sigue el patrón `ADR-NNN` (ADR-001, ADR-002, …).
3. El estado puede ser: `PROPOSED` | `ACCEPTED` | `DEPRECATED` | `SUPERSEDED BY ADR-NNN`.
4. Incluye siempre las opciones descartadas y por qué; facilita revisitar la decisión en el futuro.

---

## Índice

| ID | Título | Fecha | Estado |
|---|---|---|---|
| ADR-001 | [RELLENAR: primera decisión de arquitectura del proyecto] | ⟨pendiente⟩ | PROPOSED |

---

## Plantilla de entrada

```
## ADR-NNN — Título de la decisión

**Fecha:** YYYY-MM-DD
**Estado:** PROPOSED | ACCEPTED | DEPRECATED | SUPERSEDED BY ADR-NNN
**Autor:** [RELLENAR]

### Contexto

[RELLENAR: por qué surgió esta decisión. Qué problema o necesidad la motivó.]

### Opciones consideradas

1. **[Opción A]** — [descripción breve]. Ventajas: … Inconvenientes: …
2. **[Opción B]** — [descripción breve]. Ventajas: … Inconvenientes: …

### Decisión

[RELLENAR: qué opción se eligió y por qué.]

### Consecuencias

- [RELLENAR: qué cambia como resultado de esta decisión.]
- [RELLENAR: qué deuda técnica asume el equipo.]
```

---

## ADR-001 — Elección de la arquitectura por capas

**Fecha:** ⟨pendiente de confirmar — ver duda #D2⟩
**Estado:** PROPOSED
**Autor:** [RELLENAR]

### Contexto

El proyecto requiere una arquitectura de datos por capas para separar la ingesta, la
transformación y el consumo analítico. Hay dos familias estándar disponibles.

### Opciones consideradas

1. **landing / staging / marts** — nomenclatura clásica de Data Warehouse.
   Ventajas: ampliamente conocida; compatible con dbt out-of-the-box.
   Inconvenientes: menos flexible para Data Lakehouse o arquitecturas híbridas.
2. **bronze / silver / gold** — nomenclatura medallón, habitual en Data Lakehouse.
   Ventajas: natural en Databricks/Delta Lake; se adapta bien a datos semiestructurados.
   Inconvenientes: terminología menos familiar en equipos con background SQL/DWH.

### Decisión

⟨pendiente de confirmar — ver duda #D2⟩ — rellenar cuando el proyecto elija la familia de capas.

### Consecuencias

- [RELLENAR al resolver la duda #D2.]
