# Sensor: Lineage Staleness

**ID:** `DP-02`
**Capa:** drift-periódico
**Cuándo corre:** periódico — cadencia configurable por proyecto (`⟨a definir por el proyecto⟩`; valor por defecto recomendado: diario)
**Triggereado por:** scheduler externo. También puede lanzarse manualmente tras un cambio de schema significativo en assets de capa upstream.

## Qué comprueba

Este sensor protege la invariante de lineage: cuando un asset upstream cambia (schema, lógica de transformación, frecuencia de actualización), todos los assets downstream que dependen de él deben reflejar ese cambio de forma explícita — ya sea actualizando su propia lógica, su contrato de datos o sus tests.

El lineage stale es una forma insidiosa de deuda técnica: el cambio upstream es visible, el asset downstream sigue funcionando (no hay error), pero produce resultados incorrectos o desactualizados sin que nadie lo haya notado. El sensor detecta esta divergencia de forma periódica, antes de que se convierta en un incidente.

El sensor compara dos fuentes: (1) el grafo de lineage registrado en el catálogo o en los metadatos del pipeline, y (2) el historial de cambios de los assets (último commit o última modificación de schema). Si un asset upstream fue modificado más recientemente que sus dependientes downstream y ninguno de los downstream tiene un cambio registrado posterior, el sensor lo marca como "stale".

El umbral de tiempo para considerar un lineage stale es configurable por proyecto: por defecto, si el upstream fue modificado hace más de N días sin que el downstream haya registrado ningún cambio o revisión explícita, se reporta.

## Checks incluidos

| Check | Tipo | Invariante |
|---|---|---|
| Downstream no actualizado tras cambio de schema upstream | determinista | Si el schema de un asset upstream cambió, todos sus dependientes directos tienen un cambio o revisión posterior |
| Downstream no actualizado tras cambio de lógica upstream | determinista | Si la lógica de transformación de un asset upstream cambió, todos sus dependientes directos tienen un cambio o revisión posterior |
| Dependencia no documentada en el grafo de lineage | semi-determinista | Todo asset que consume datos de otro asset lo declara explícitamente en su definición (sin dependencias implícitas) |
| Asset con lineage desconocido | determinista | Todos los assets tienen al menos una dependencia upstream documentada o están marcados explícitamente como "fuente primaria" |

## Feedback emitido

Cuando el sensor detecta un lineage stale, emite al contexto del agente:

```
SENSOR FAIL: DP-02
Asset: {nombre del asset downstream afectado — p.ej. "marts.fct_orders"}
Check: {nombre del check que falló — ver tabla de checks}
Finding: {descripción concreta — p.ej. "Asset 'marts.fct_orders' depende de 'staging.stg_order_items', que fue modificado hace 8 días (2026-06-01). 'fct_orders' no registra ningún cambio posterior. Umbral configurado: 3 días."}
Remediation: {instrucción accionable — p.ej. "Revisar si el cambio en 'stg_order_items' (añadido campo 'discount_type') requiere actualizar la lógica de 'fct_orders'. Si el cambio no afecta a fct_orders, añadir un comentario de revisión en el contrato de datos con la fecha de hoy para reiniciar el contador."}
```

## Categoría de implementación (D11)

Este sensor cubre lineage, **fuera de las cinco categorías de sensor de D11**
(hard_spec.md §5: lint SQL, typecheck/lint Python, validación de schema/contratos,
freshness/nulls/volumetría, tests de pipeline). El perfil de stack del proyecto
(`project-template/stack-profile.yml`) no tiene un campo dedicado a lineage — el campo
informativo `plataforma_cliente` es la referencia más cercana si el proyecto declara una
herramienta de lineage o catálogo centralizada (OpenLineage, DataHub). En su ausencia, la
cobertura de este sensor es responsabilidad del perfil de cada proyecto, sin un campo
dedicado en el contrato D11.

## Implementaciones de referencia

| Herramienta | Cómo implementa este sensor | Disponibilidad |
|---|---|---|
| Script Python + git log | Compara fechas de último commit de los ficheros de modelo SQL/PySpark para detectar divergencias de actualización | Implementación de referencia mínima, sin dependencias adicionales |
| dbt lineage graph + `dbt ls` | El DAG de dbt contiene el lineage; un script compara `updated_at` de los modelos con el del upstream | Si el proyecto usa dbt como motor de transformación (declarado en `plataforma_cliente`) |
| OpenLineage / Marquez | Framework de lineage estándar (OpenLineage spec); registra eventos de ejecución con relaciones entre assets | Si el proyecto adopta OpenLineage |
| DataHub lineage API | Lineage consumible via API; permite queries de "¿qué dependientes tiene este asset y cuándo fue actualizado cada uno?" | Si el proyecto tiene un catálogo de datos centralizado declarado en `plataforma_cliente` |

> Nota: este sensor requiere que el lineage esté documentado de alguna forma (DAG de pipeline, catálogo, o metadatos de fichero). Si el proyecto no tiene lineage documentado, el check `Asset con lineage desconocido` fallará para todos los assets, lo que es el primer feedback accionable: documentar el lineage.
