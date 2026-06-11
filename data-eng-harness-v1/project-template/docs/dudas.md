# Backlog de dudas — [Nombre del proyecto] ⟨pendiente de confirmar⟩

> Registro de gaps, preguntas abiertas y decisiones que requieren confirmación externa
> (cliente, equipo, tech lead).
> **Append-only:** nunca se borran entradas. Las dudas resueltas se marcan con
> estado `resuelta` y se rellena la columna `Resolución`.
> Las dudas que bloquean un artefacto deben citarse en ese artefacto como
> `⟨pendiente de confirmar — ver duda #{ID}⟩`.

---

## Instrucciones de uso

1. **Al detectar un gap** — cualquier agente puede añadir una entrada al final de la tabla.
   El ID sigue el patrón `D{numero_secuencial}` (D1, D2, …).
2. **Al referenciar una duda en un artefacto** — usar la notación
   `⟨pendiente de confirmar — ver duda #{ID}⟩` en el lugar del artefacto donde falta la información.
3. **Al resolver una duda** — añadir la fecha de resolución, cambiar el estado a `resuelta`
   y escribir la resolución en la columna correspondiente. No eliminar la fila.
4. **Quién puede resolver** — solo quien tenga autoridad para confirmar el dato
   (cliente, tech lead, humano que aprobó el bloque). Los agentes no marcan dudas como
   resueltas por su cuenta.

---

## Tabla de dudas

| ID | Fecha apertura | Bloque | Pregunta / Gap | Responsable | Estado | Fecha resolución | Resolución |
|----|----------------|--------|----------------|-------------|--------|------------------|------------|
| D1 | ⟨pendiente⟩ | R1 | ¿Qué stack de datos usa el equipo? (herramienta de transformación, warehouse, calidad) | tech lead / equipo | resuelta | 2026-06-10 | Resuelto a nivel de arnés por D11 (hard_spec.md §5): el stack es heterogéneo por cliente; cada proyecto rellena `stack-profile.yml` con su perfil concreto. Este proyecto debe rellenar `stack-profile.yml` con su propio perfil. |
| D2 | ⟨pendiente⟩ | — | ¿Qué familia de capas adopta el proyecto: landing/staging/marts o bronze/silver/gold? | tech lead | abierta | — | — |

---

## Detalle de dudas abiertas

### D1 — Stack de datos del equipo (resuelta)

**Contexto:** el `CLAUDE.md` y `data-conventions.md` de la plantilla no podían fijar
herramientas concretas (dbt, Airflow, Snowflake, Great Expectations, etc.) sin conocer
el stack real del equipo de ingeniería de datos de The Cocktail.

**Resolución (2026-06-10):** D11 (hard_spec.md §5) confirma que el stack es heterogéneo
y multi-cloud por cliente. El core (`../../core/sensors/`) define **categorías**
parametrizables (lint SQL con dialecto, lint/typecheck Python, validación de
schema/contratos, freshness/nulls/volumetría, tests de pipeline) con Python + SQL +
Spark/PySpark, `sqlfluff`, `ruff` + `mypy` y DuckDB como implementaciones de referencia.
Cada proyecto instancia esas categorías en `stack-profile.yml` (raíz del proyecto).

**Acción para este proyecto:** rellenar `stack-profile.yml` con el stack real de este
cliente/proyecto y sincronizar `docs/references/stack.md`.

---

### D2 — Familia de capas de datos

**Contexto:** la plantilla ofrece dos opciones estándar (landing/staging/marts y
bronze/silver/gold) sin decidir por ninguna. La elección impacta el naming de modelos,
los contratos de datos y los tests de pipeline. Es una decisión independiente del
perfil de stack de D11 (D11 no fija la arquitectura por capas).

**Bloquea:**
- `docs/architecture/data-layers.md` (sección de invariantes de la capa elegida)
- `docs/architecture/decisions.md` (ADR-001)
- `data-conventions.md §1` y `§2` (naming por capa)
- `docs/quality/slas.md` (campo `Capa`)

**Pendiente de:** tech lead del proyecto (decisión de arquitectura).
