# Catálogo de sensores — índice ejecutivo

> Fuente de verdad para saber qué sensores existen, cuándo corren y qué protegen.
> Cada fila enlaza al fichero de especificación completa del sensor.
> El core especifica **categorías** de sensor parametrizables (D11); cada proyecto las
> instancia mediante su perfil de stack — ver `project-template/stack-profile.yml`.

> **Invocables por el evaluador (D13).** Los sensores **fast-feedback** (FF-01 a FF-04) no son
> solo herramientas del implementador durante el desarrollo: el `evaluador` los **ejecuta como
> parte de su veredicto** (`core/contracts/evaluador.md`) cuando un `acceptance_criterion` requiere
> evidencia de ejecución (schema válido, tests en verde, lint sin errores, datos sin nulls
> críticos) y no solo lectura de código. Los sensores **drift-periódico** (DP-01 a DP-03) no son
> invocados por el evaluador en el ciclo por-tarea: corren con su propia cadencia.

## Tabla de sensores

| ID | Nombre | Capa | Cuándo corre | Invariante que protege | Spec |
|---|---|---|---|---|---|
| FF-01 | Schema Contract | fast-feedback | in-session + CI-pipeline | El schema de una tabla/dataset coincide con su contrato de datos declarado | [fast-feedback/schema-contract.md](fast-feedback/schema-contract.md) |
| FF-02 | Data Quality | fast-feedback | CI-pipeline | Nulls en columnas críticas, duplicados, freshness, volume anómalo, validez semántica | [fast-feedback/data-quality.md](fast-feedback/data-quality.md) |
| FF-03 | Pipeline Tests | fast-feedback | in-session + CI-pipeline | Los tests unitarios e de integración del pipeline pasan sin errores | [fast-feedback/pipeline-tests.md](fast-feedback/pipeline-tests.md) |
| FF-04 | SQL/Python Lint | fast-feedback | in-session (pre-commit) + CI-pipeline | El código SQL y Python cumple las reglas de estilo y convenciones del equipo | [fast-feedback/sql-lint.md](fast-feedback/sql-lint.md) |
| DP-01 | Golden Data Principles | drift-periódico | periódico (cadencia configurable) | Degradación acumulada de calidad, documentación y gobernanza de datos | [drift-periodic/golden-data-principles.md](drift-periodic/golden-data-principles.md) |
| DP-02 | Lineage Staleness | drift-periódico | periódico (cadencia configurable) | Assets con upstream modificado cuyo lineage downstream no se ha actualizado | [drift-periodic/lineage-staleness.md](drift-periodic/lineage-staleness.md) |
| DP-03 | Catalog Completeness | drift-periódico | periódico (cadencia configurable) | Tablas sin owner documentado, assets sin contrato de datos, columnas sin descripción | [drift-periodic/catalog-completeness.md](drift-periodic/catalog-completeness.md) |

## Cobertura por dimensión de invariante

| Dimensión | Sensor(es) que la cubren |
|---|---|
| Schema | FF-01 |
| Freshness | FF-02 |
| Nullability | FF-02 |
| Duplicados | FF-02 |
| Validez/semántica | FF-02 |
| Volume/row count | FF-02 |
| Contrato de datos | FF-01 |
| Tests de pipeline | FF-03 |
| Calidad de código SQL/Python | FF-04 |
| Lineage | DP-02 |
| Completitud del catálogo | DP-03 |
| Golden principles de datos | DP-01 |

## Correspondencia con las 5 categorías de sensor de D11

D11 (`DESIGN.md` §9) define **5 categorías** de sensor parametrizables que cada
proyecto instancia en `project-template/stack-profile.yml`: lint SQL (con dialecto),
typecheck/lint Python, validación de schema/contratos de datos, freshness/nulls/volumetría
y tests de pipeline. Esta tabla relaciona cada uno de los 7 sensores del catálogo con la
categoría de D11 que cubre, o indica que queda fuera de las 5 categorías.

| Sensor | Categoría de D11 | Campo de `stack-profile.yml` que lo instancia |
|---|---|---|
| FF-01 Schema Contract | Validación de schema/contratos de datos | `schema_contracts.engine` |
| FF-02 Data Quality | Freshness/nulls/volumetría | `data_quality.engine` |
| FF-03 Pipeline Tests | Tests de pipeline | `pipeline_tests.framework` |
| FF-04 SQL/Python Lint | Lint SQL (con dialecto) + typecheck/lint Python | `sql.dialect`/`sql.linter` y `python.lint`/`python.typecheck` |
| DP-01 Golden Data Principles | **Fuera de las 5 categorías de D11** (catálogo/gobernanza) | sin campo dedicado — `plataforma_cliente` como referencia informativa (ver "Categoría de implementación (D11)" en [drift-periodic/golden-data-principles.md](drift-periodic/golden-data-principles.md)) |
| DP-02 Lineage Staleness | **Fuera de las 5 categorías de D11** (lineage) | sin campo dedicado — `plataforma_cliente` como referencia informativa (ver "Categoría de implementación (D11)" en [drift-periodic/lineage-staleness.md](drift-periodic/lineage-staleness.md)) |
| DP-03 Catalog Completeness | **Fuera de las 5 categorías de D11** (catálogo/gobernanza) | sin campo dedicado — `plataforma_cliente` como referencia informativa (ver "Categoría de implementación (D11)" en [drift-periodic/catalog-completeness.md](drift-periodic/catalog-completeness.md)) |

> Nota: FF-04 cubre dos categorías de D11 (lint SQL y typecheck/lint Python) porque agrupa
> ambas comprobaciones bajo un mismo sensor de "calidad de código". Los sensores
> drift-periódico (DP-01..03) cubren catálogo, gobernanza y lineage — dimensiones de
> calidad reales del ecosistema de datos, pero deliberadamente fuera del alcance de las 5
> categorías parametrizables de D11, que se centran en feedback rápido sobre código y datos
> en el ciclo de desarrollo. `stack-profile.yml` no necesita (ni debe) ampliarse con campos
> dedicados para DP-01..03 mientras esa frontera se mantenga.

## Notas de diseño

Los sensores de `core/sensors/` son **especificaciones**, no implementaciones ejecutables.
Describen qué comprueba cada sensor, cuándo corre y qué feedback emite; el core define la
**categoría** de cada sensor (lint SQL con dialecto parametrizable, typecheck/lint de
Python, validación de schema/contratos de datos, freshness/nulls/volumetría, tests de
pipeline), no herramientas concretas (D11). La capa concreta de implementación la
**instancia el perfil de stack del proyecto** (ver
`project-template/stack-profile.yml`), declarado en la sección "Implementaciones de
referencia" de cada fichero de sensor.

R1 (stack de datos sin confirmar) queda **resuelta por D11**: el core no fija
herramientas, sino categorías + perfil de stack por proyecto, con
Python + SQL + Spark/PySpark como soporte de primera clase y `sqlfluff`
(dialecto configurable), `ruff` + `mypy` y DuckDB como implementaciones de referencia.
