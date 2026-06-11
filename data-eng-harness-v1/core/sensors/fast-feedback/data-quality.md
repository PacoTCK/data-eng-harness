# Sensor: Data Quality

**ID:** `FF-02`
**Capa:** fast-feedback
**Cuándo corre:** CI-pipeline (en cada PR que materialice o modifique un dataset) y opcionalmente in-session tras una ejecución local del pipeline
**Triggereado por:** ejecución de un pipeline de transformación o ingestión que produce o actualiza un dataset.
**Invocado por:** el implementador tras ejecutar el pipeline localmente, y el `evaluador` como parte de su veredicto cuando un `acceptance_criterion` requiere evidencia de calidad de datos sobre el dataset producido (D13).

## Qué comprueba

Este sensor protege las invariantes de calidad del dato en cinco dimensiones: nullability, duplicados, freshness, volume y validez semántica.

**Nullability:** columnas declaradas como críticas (identificadores, claves de negocio, fechas de transacción) no deben contener valores nulos. Un nulo en una columna de identificador rompe joins y silencia errores en cómputos downstream.

**Duplicados:** combinaciones de columnas que deben ser únicas (claves primarias, claves de negocio compuestas) no deben tener duplicados. Los duplicados no detectados se propagan por el lineage y generan counts incorrectos que son difíciles de rastrear.

**Freshness:** el dataset debe haberse actualizado dentro del SLA declarado en su contrato de datos. Un dataset más antiguo que el SLA es silenciosamente incorrecto para los consumidores downstream.

**Volume:** el número de filas del dataset no debe caer por debajo de un umbral mínimo ni superar un umbral máximo configurados en el contrato. Un dataset vacío o anómalamente pequeño/grande indica un fallo de pipeline no capturado por tests de integración.

**Validez/semántica:** los valores de columnas críticas deben cumplir reglas declaradas: rangos numéricos, listas de valores permitidos, patrones de regex. Valores fuera de rango no se detectan a nivel de schema (el tipo es correcto) pero generan resultados de negocio incorrectos.

## Checks incluidos

| Check | Tipo | Invariante |
|---|---|---|
| Not-null en columnas críticas | determinista | Las columnas marcadas `critical: true` en el contrato no tienen nulos |
| Unicidad de clave primaria | determinista | La combinación de columnas declarada como PK no tiene duplicados |
| Unicidad de clave de negocio | determinista | Las claves de negocio compuestas declaradas en el contrato son únicas |
| Freshness dentro de SLA | determinista | `MAX(updated_at)` del dataset ≤ `NOW() - sla_hours` definido en el contrato |
| Volume mínimo | determinista | `COUNT(*)` ≥ `min_rows` declarado en el contrato |
| Volume máximo | determinista | `COUNT(*)` ≤ `max_rows` declarado en el contrato (si se define) |
| Rango numérico | determinista | Los valores de columnas numéricas críticas están dentro del rango `[min_value, max_value]` del contrato |
| Lista de valores permitidos | determinista | Los valores de columnas categóricas están dentro del `allowed_values` del contrato |
| Patrón regex | determinista | Los valores de columnas de texto cumplen el `pattern` declarado en el contrato |

## Feedback emitido

Cuando el sensor falla, emite al contexto del agente:

```
SENSOR FAIL: FF-02
Asset: {nombre_schema}.{nombre_tabla}
Check: {nombre del check que falló — ver tabla de checks}
Finding: {descripción concreta — p.ej. "Columna 'order_id' contiene 142 nulos (0.3% de 47,000 filas). Se esperaban 0 nulos según el contrato."}
Remediation: {instrucción accionable — p.ej. "Revisar el paso de transformación que popula 'order_id'; añadir un filtro WHERE order_id IS NOT NULL o corregir la fuente de datos upstream antes de hacer merge"}
```

Si varios checks fallan en el mismo asset, se emite un bloque `SENSOR FAIL` por cada check fallido (no se agrupan), para que el agente pueda tratarlos de forma independiente.

## Categoría de implementación (D11)

Este sensor cubre la categoría **freshness/nulls/volumetría** del perfil de stack del
proyecto (`project-template/stack-profile.yml`, campo `data_quality`).

## Implementaciones de referencia

Soporte de primera clase del core para Python + SQL + Spark/PySpark (D11):

| Herramienta | Cómo implementa este sensor | Categoría que instancia |
|---|---|---|
| Pandera / pydantic (Python) | Validación en tiempo de ejecución del DataFrame (Pandas/PySpark) antes de escribir: not-null, unicidad, rangos, valores permitidos | Freshness/nulls/volumetría |
| DuckDB | Warehouse local de pruebas: ejecuta consultas SQL de agregación (`COUNT`, `MAX(updated_at)`, rangos) sobre las tablas materializadas para validar freshness, volume y duplicados sin depender de infraestructura de cliente | Freshness/nulls/volumetría (entorno de pruebas) |

Otras herramientas (dbt tests + dbt-expectations, Great Expectations, Soda Core, etc.) son
válidas si el perfil de stack del proyecto las declara en `data_quality.engine`.

> Nota: el SLA de freshness y los umbrales de volume se declaran en el contrato de datos del asset, no en este sensor. El sensor solo lee esos valores y los evalúa.
