# Sensor: Pipeline Tests

**ID:** `FF-03`
**Capa:** fast-feedback
**Cuándo corre:** in-session (al modificar un paso de transformación, ejecutable localmente) + CI-pipeline (en cada PR que toque lógica de pipeline)
**Triggereado por:** modificación de cualquier fichero que forme parte del pipeline (modelos SQL, scripts de transformación Python, configuración de orquestación, fixtures de test).
**Invocado por:** el implementador durante el desarrollo, y el `evaluador` como parte de su veredicto — corre los tests del pipeline y basa el `PASA`/`FALLA` en su resultado de ejecución, no en la lectura del código de test (D13).

## Qué comprueba

Este sensor protege la invariante de que la lógica del pipeline de transformación produce los resultados esperados bajo condiciones controladas.

Los tests de pipeline son la red de seguridad que permite al agente refactorizar transformaciones con confianza. Sin ellos, el agente no puede saber si un cambio en un paso intermedio rompe la salida final, y tampoco puede confiar en que una transformación nueva produce el resultado de negocio correcto.

**Tests unitarios** verifican la lógica de una transformación aislada: dada una entrada controlada (fixture), la salida debe coincidir con el resultado esperado. Son deterministas, rápidos y deben poder ejecutarse sin acceso a la base de datos de producción.

**Tests de integración** verifican que la cadena completa de transformaciones (ingestión → staging → marts o equivalente en las capas declaradas en `data-conventions.md`) produce el dataset final correcto cuando se ejecuta sobre datos de prueba. Detectan problemas de joins, filtros acumulativos y dependencias de orden que los tests unitarios no cubren.

**Tests de schema de salida** verifican que el pipeline produce el schema esperado (columnas, tipos) en el dataset de salida, incluso si los datos son sintéticos. Son el puente entre FF-03 y FF-01: un test de schema de salida falla antes de que FF-01 llegue a comparar contra el contrato.

## Checks incluidos

| Check | Tipo | Invariante |
|---|---|---|
| Tests unitarios de transformación | determinista | Cada función/modelo de transformación produce la salida esperada dado el fixture de entrada |
| Tests de integración de pipeline | determinista | La cadena completa de transformaciones produce el dataset final correcto sobre datos de prueba |
| Tests de schema de salida | determinista | El pipeline produce exactamente las columnas y tipos declarados en el contrato de datos de salida |
| Tests de idempotencia | determinista | Ejecutar el pipeline dos veces sobre los mismos datos produce el mismo resultado (sin duplicados, sin efectos secundarios) |
| Tests de datos de borde | determinista | El pipeline maneja correctamente casos límite declarados: dataset vacío, valores nulos en entrada, volumen máximo |

## Feedback emitido

Cuando el sensor falla, emite al contexto del agente:

```
SENSOR FAIL: FF-03
Asset: {nombre del pipeline o modelo — p.ej. "pipeline/staging/stg_orders.sql"}
Check: {nombre del check que falló — ver tabla de checks}
Finding: {descripción concreta — p.ej. "Test unitario 'test_dedup_logic' falla: salida contiene 3 filas duplicadas con order_id='ORD-001' cuando la entrada tiene 1 fila. Fixture: tests/fixtures/orders_input.csv"}
Remediation: {instrucción accionable — p.ej. "Revisar la cláusula GROUP BY en stg_orders.sql línea 42; el JOIN con la tabla de estados puede estar generando el fanout. Corregir y re-ejecutar el test antes de hacer merge."}
```

## Categoría de implementación (D11)

Este sensor cubre la categoría **tests de pipeline** del perfil de stack del proyecto
(`project-template/stack-profile.yml`, campo `pipeline_tests`).

## Implementaciones de referencia

Soporte de primera clase del core para Python + SQL + Spark/PySpark (D11):

| Herramienta | Cómo implementa este sensor | Categoría que instancia |
|---|---|---|
| pytest + fixtures locales | Tests unitarios de funciones Python/PySpark de transformación; fixtures como DataFrames o CSVs | Tests de pipeline (unitarios) |
| pytest + DuckDB | Tests de integración que ejecutan la cadena de transformaciones SQL/PySpark sobre tablas DuckDB de prueba (warehouse local de pruebas, D11) | Tests de pipeline (integración) |

Otras herramientas (dbt + dbt-unit-testing, pytest-dbt-core, Airflow unit tests/TaskFlow,
etc.) son válidas si el perfil de stack del proyecto las declara en
`pipeline_tests.framework`.

> Nota: la distinción entre tests unitarios y de integración es independiente del stack. El criterio es: ¿requiere el test acceso a un sistema externo (BD, API)? Si sí, es de integración. Si no, es unitario.
