# Catálogo de pipelines del proyecto

> Propósito: inventario centralizado de todos los pipelines de datos del proyecto.
> El agente debe leer este fichero antes de crear o modificar un pipeline, para evitar
> duplicar lógica o introducir dependencias no documentadas.

---

## Cómo rellenar este fichero

1. Añade una fila por cada pipeline al momento de crearlo (no retroactivamente).
2. Mantén el campo `Estado` actualizado (`activo` | `deprecado` | `en-desarrollo`).
3. El campo `Plan activo` apunta al fichero JSON o Markdown en `active/` que describe
   el plan de ejecución del pipeline (equivalente a exec-plans/active/ del core).
4. El campo `Contrato de datos` apunta al fichero de contrato del dataset que produce
   este pipeline (ver `../quality/contracts/`).

---

## Tabla de pipelines

| Nombre | Capa origen | Capa destino | Owner | SLA (hours) | Estado | Plan activo | Contrato de datos |
|---|---|---|---|---|---|---|---|
| [RELLENAR] | ⟨pendiente⟩ | ⟨pendiente⟩ | [RELLENAR] | ⟨pendiente⟩ | en-desarrollo | — | — |

> Nota: `SLA (hours)` es el tiempo máximo entre la disponibilidad del dato en origen
> y su disponibilidad en destino. Coherente con el campo `sla_hours` del contrato de datos
> y con el sensor FF-02 (Data Quality / Freshness) de `${CLAUDE_PLUGIN_ROOT}/core/sensors/`.

---

## Dependencias entre pipelines

> Rellenar cuando existan dependencias. Un pipeline no debe asumir que otro ha terminado
> sin declararlo aquí y en el contrato de su dataset de entrada.

```
[RELLENAR: diagrama Mermaid o lista de dependencias cuando el catálogo tenga entradas.]
```

---

## Planes activos (`active/`)

El subdirectorio `active/` contiene los planes de ejecución de los pipelines en
desarrollo activo. Cada fichero sigue el naming `{nombre-pipeline}-plan.md` o `.json`.

Los planes completados se mueven a `active/archive/` (crear el subdirectorio al necesitarlo).
