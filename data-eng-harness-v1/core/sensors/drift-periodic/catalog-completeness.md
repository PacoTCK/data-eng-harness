# Sensor: Catalog Completeness

**ID:** `DP-03`
**Capa:** drift-periódico
**Cuándo corre:** periódico — cadencia configurable por proyecto (`⟨a definir por el proyecto⟩`; valor por defecto recomendado: semanal)
**Triggereado por:** scheduler externo. También puede lanzarse manualmente antes de una revisión de calidad del catálogo o de un handoff de proyecto.

## Qué comprueba

Este sensor protege la invariante de que el catálogo de datos del proyecto está completo y es utilizable: cada tabla expuesta tiene un owner, cada asset tiene un contrato de datos asociado, y cada columna crítica tiene una descripción legible que permite al agente (y al humano) entender qué contiene sin tener que leer el código de transformación.

La incompletitud del catálogo es el problema de gobernanza más común en proyectos de datos que crecen rápido: se añaden tablas, se crean columnas, se despliegan pipelines, y la documentación queda siempre un paso atrás. El sensor lo detecta sistemáticamente, no a ojo.

La diferencia con DP-01 (Golden Data Principles) es de granularidad: DP-01 evalúa principios de arquitectura y gobernanza (p. ej. "ningún modelo tiene más de N dependencias"); DP-03 evalúa completitud de metadatos a nivel de campo (p. ej. "esta columna concreta no tiene descripción"). DP-03 es más granular y más fácil de remediar de forma incremental.

El sensor evalúa tres niveles de completitud:
- **Nivel tabla:** owner documentado, descripción de la tabla, capa del modelo declarada (staging, marts, etc.).
- **Nivel asset:** contrato de datos asociado (fichero YAML/JSON con schema y metadatos).
- **Nivel columna:** descripción de columnas críticas (las marcadas como `critical: true` en el contrato o en los principios del proyecto).

## Checks incluidos

| Check | Tipo | Invariante |
|---|---|---|
| Tabla sin owner documentado | determinista | Todo tabla expuesta tiene un campo `owner` en su contrato o catálogo |
| Tabla sin descripción | determinista | Toda tabla expuesta tiene una descripción no vacía en su contrato o catálogo |
| Asset sin contrato de datos | determinista | Todo asset de datos tiene un fichero de contrato asociado (YAML, JSON o equivalente) |
| Columna crítica sin descripción | determinista | Toda columna marcada como crítica (`critical: true`) tiene una descripción legible en el contrato |
| Columna sin tipo declarado | determinista | Toda columna del contrato tiene un tipo de dato explícito |
| Tabla sin capa declarada | determinista | Toda tabla tiene su capa de datos declarada (staging, intermediate, marts, raw, etc.) |

## Feedback emitido

Cuando el sensor detecta incompletitud, emite al contexto del agente:

```
SENSOR FAIL: DP-03
Asset: {nombre de la tabla, columna o contrato afectado — referencia exacta}
Check: {nombre del check que falló — ver tabla de checks}
Finding: {descripción concreta — p.ej. "Tabla 'staging.stg_payments' no tiene owner documentado. Lleva 22 días sin owner desde su creación (2026-05-18)."}
Remediation: {instrucción accionable — p.ej. "Añadir 'owner: {nombre_equipo}' al contrato de datos contracts/staging/stg_payments.yml. Si el propietario no está claro, revisar con el lead del proyecto quién es responsable de esta tabla."}
```

El sensor emite un bloque `SENSOR FAIL` por cada ítem incompleto. Si el número de ítems incompletos es muy alto (p. ej. al iniciarse en un proyecto con catálogo heredado), puede generar muchos bloques — en ese caso se recomienda priorizar por capa (marts primero, luego staging, luego raw) y abordar de forma incremental.

## Categoría de implementación (D11)

Este sensor cubre catálogo/gobernanza, **fuera de las cinco categorías de sensor de
D11** (hard_spec.md §5: lint SQL, typecheck/lint Python, validación de schema/contratos,
freshness/nulls/volumetría, tests de pipeline). El perfil de stack del proyecto
(`project-template/stack-profile.yml`) no tiene un campo dedicado a catálogo/gobernanza
— el campo informativo `plataforma_cliente` es la referencia más cercana si el proyecto
declara una herramienta de catálogo centralizada (DataHub, Atlan, Dataplex). En su
ausencia, la cobertura de este sensor es responsabilidad del perfil de cada proyecto,
sin un campo dedicado en el contrato D11.

## Implementaciones de referencia

| Herramienta | Cómo implementa este sensor | Disponibilidad |
|---|---|---|
| Script Python + parseo de YAML | Recorre los ficheros de contrato YAML del repositorio (formato declarado en `schema_contracts.formato_contrato` del perfil de stack) y reporta campos vacíos o ausentes | Implementación de referencia mínima, sin dependencias adicionales |
| dbt `schema.yml` + custom macros | Comprueba la presencia de `description`, `meta.owner` y `tests` en todos los nodos del DAG de dbt | Si el proyecto usa dbt como motor de transformación (declarado en `plataforma_cliente`) |
| dbt-docs + script de auditoría | Genera el catálogo HTML de dbt y un script evalúa los campos de metadatos contra los umbrales del sensor | Si el proyecto usa dbt |
| DataHub / Atlan / Dataplex | Herramienta de data catalog con checks de completitud integrados y dashboard de cobertura de metadatos | Si el proyecto tiene un catálogo de datos centralizado declarado en `plataforma_cliente` |

> Nota: el umbral de qué columnas son "críticas" se declara en el contrato de datos del asset o en los principios del proyecto (DP-01). Si ninguna columna está marcada como crítica, el check `Columna crítica sin descripción` evalúa todas las columnas.
