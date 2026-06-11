# Sensor: Schema Contract

**ID:** `FF-01`
**Capa:** fast-feedback
**Cuándo corre:** in-session (al modificar un modelo/tabla) + CI-pipeline (en cada PR que afecte a un modelo o a su contrato)
**Triggereado por:** modificación de un fichero de modelo de datos, de un fichero de contrato de datos (YAML/JSON), o de una migración de schema.
**Invocado por:** el implementador durante el desarrollo, y el `evaluador` como parte de su veredicto cuando un `acceptance_criterion` requiere evidencia de que el schema real coincide con el contrato (D13).

## Qué comprueba

Este sensor protege la invariante de que el schema real de una tabla o dataset coincide en todo momento con el contrato de datos declarado para ese asset.

Un contrato de datos es el documento (YAML, JSON u otro formato estructurado) que declara explícitamente: columnas esperadas, tipos, nullability, descripción y restricciones. Cuando el schema real y el contrato divergen, el agente opera sobre una realidad distinta a la documentada, lo que genera errores silenciosos difíciles de depurar.

El sensor comprueba en ambas direcciones: columnas presentes en el schema pero no en el contrato (columnas "fantasma", añadidas sin actualizar la documentación) y columnas declaradas en el contrato pero ausentes del schema real (rotura de contrato por eliminación o renombrado). También detecta cambios de tipo incompatibles (p. ej. `INTEGER` → `VARCHAR` sin versionado del contrato).

## Checks incluidos

| Check | Tipo | Invariante |
|---|---|---|
| Columnas añadidas sin declarar | determinista | Toda columna nueva debe aparecer en el contrato antes de llegar a main |
| Columnas eliminadas del schema | determinista | Toda eliminación de columna debe reflejarse en el contrato con versionado |
| Tipo incompatible | determinista | El tipo de cada columna en el schema coincide con el declarado en el contrato |
| Nullability violada | determinista | Las columnas declaradas `NOT NULL` en el contrato no contienen nulos en el schema real |
| Contrato ausente | determinista | Todo modelo o tabla expuesta tiene un fichero de contrato asociado |

## Feedback emitido

Cuando el sensor falla, emite al contexto del agente:

```
SENSOR FAIL: FF-01
Asset: {nombre_schema}.{nombre_tabla} (o ruta del modelo)
Check: {nombre del check que falló — ver tabla de checks}
Finding: {descripción concreta — p.ej. "Columna 'customer_segment' presente en el schema pero no declarada en el contrato contracts/orders.yml"}
Remediation: {instrucción accionable — p.ej. "Añadir 'customer_segment' al contrato contracts/orders.yml con tipo, nullability y descripción antes de hacer merge"}
```

El bloque `SENSOR FAIL` es idéntico en formato al del resto de sensores del catálogo. El agente no necesita lógica condicional para procesarlo.

## Categoría de implementación (D11)

Este sensor cubre la categoría **validación de schema/contratos de datos** del perfil de
stack del proyecto (`project-template/stack-profile.yml`, campo `schema_contracts`).

## Implementaciones de referencia

Soporte de primera clase del core para Python + SQL + Spark/PySpark (D11):

| Herramienta | Cómo implementa este sensor | Categoría que instancia |
|---|---|---|
| Pandera (Python) | Validación de schema declarativa sobre DataFrames Pandas/PySpark; el contrato YAML del proyecto se traduce a un esquema Pandera | Validación de schema/contratos de datos |
| DuckDB | Warehouse local de pruebas: ejecuta `DESCRIBE`/consultas de catálogo sobre las tablas materializadas y compara contra el contrato sin depender de infraestructura de cliente | Validación de schema/contratos de datos (entorno de pruebas) |

Otras herramientas (dbt schema tests, Great Expectations, Soda Core, pre-commit hooks
personalizados sobre el warehouse de cliente, etc.) son válidas si el perfil de stack del
proyecto las declara en `schema_contracts.engine`. El contrato de este sensor (qué
comprueba, qué feedback emite) es independiente de la implementación elegida.
