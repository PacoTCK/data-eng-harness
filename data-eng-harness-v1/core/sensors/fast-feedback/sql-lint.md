# Sensor: SQL/Python Lint

**ID:** `FF-04`
**Capa:** fast-feedback
**Cuándo corre:** in-session (pre-commit hook, al guardar el fichero en algunos editores) + CI-pipeline (en cada PR que toque ficheros SQL o Python)
**Triggereado por:** modificación de cualquier fichero `.sql` o `.py` del repositorio.
**Invocado por:** el implementador durante el desarrollo, y el `evaluador` como parte de su veredicto cuando un `acceptance_criterion` requiere evidencia de que el código SQL/Python cumple las reglas de estilo (D13).

## Qué comprueba

Este sensor protege la invariante de que el código SQL y Python del equipo cumple las convenciones de estilo y las reglas de calidad de código acordadas.

El objetivo del sensor no es imponer gusto estético, sino proteger convenciones que tienen consecuencias funcionales o de mantenibilidad: referencias a tablas sin alias que generan ambigüedad, `SELECT *` que rompen contratos de schema, funciones no deterministas usadas en transformaciones reproducibles, o imports no utilizados que ensucian el contexto del agente al leer el código.

Las reglas de lint se configuran en ficheros de configuración del repositorio (`.sqlfluff`, `pyproject.toml` u otro equivalente declarado en el perfil de stack del proyecto, ver `project-template/stack-profile.yml`). El sensor no impone reglas por defecto — lee las reglas del proyecto. Si el proyecto no tiene fichero de configuración, el sensor reporta su ausencia como un fallo de tipo `CONFIG-MISSING`.

El sensor es deliberadamente rápido y determinista. No analiza semántica del negocio — eso es trabajo de los tests (FF-03) y de los sensores de drift (DP-01). Solo evalúa forma, convenciones sintácticas y patrones prohibidos explícitamente declarados.

## Checks incluidos

| Check | Tipo | Invariante |
|---|---|---|
| Sintaxis SQL válida | determinista | Todo fichero SQL parsea sin errores de sintaxis |
| Convenciones de estilo SQL | determinista | Indentación, capitalización de keywords, naming de CTEs y columnas siguen el estándar del proyecto |
| `SELECT *` prohibido | determinista | Ningún modelo de transformación usa `SELECT *` (rompe contratos de schema) |
| Referencias ambiguas | determinista | Toda referencia a columna en un JOIN tiene alias de tabla explícito |
| Sintaxis Python válida | determinista | Todo fichero `.py` parsea sin errores de sintaxis |
| Convenciones de estilo Python | determinista | El código Python cumple las reglas del linter configurado (PEP8 u otras declaradas) |
| Imports no utilizados | determinista | No hay imports que no se usen en el fichero Python |
| Fichero de configuración de lint presente | determinista | Existe al menos un fichero de configuración de lint en el repositorio |

## Feedback emitido

Cuando el sensor falla, emite al contexto del agente:

```
SENSOR FAIL: FF-04
Asset: {ruta/del/fichero.sql o ruta/del/fichero.py}
Check: {nombre del check que falló — ver tabla de checks}
Finding: {descripción concreta y referenciable — p.ej. "models/staging/stg_customers.sql:14 — SELECT * detectado. Regla: no-select-star"}
Remediation: {instrucción accionable — p.ej. "Sustituir SELECT * por la lista explícita de columnas necesarias. Consultar el contrato de datos en contracts/stg_customers.yml para la lista canónica."}
```

Si el fallo es `CONFIG-MISSING`:

```
SENSOR FAIL: FF-04
Asset: {repositorio raíz}
Check: Fichero de configuración de lint presente
Finding: No se encontró fichero de configuración de lint (.sqlfluff, pyproject.toml, .flake8 o equivalente). El sensor no puede aplicar reglas del proyecto.
Remediation: Crear un fichero de configuración de lint en la raíz del repositorio. Ver plantilla en project-template/data-conventions.md.
```

## Categorías de implementación (D11)

Este sensor cubre dos categorías parametrizables del perfil de stack:

- **Lint de SQL con dialecto parametrizable**: el dialecto SQL concreto (Spark SQL,
  Snowflake, BigQuery, Hive, T-SQL, etc.) lo declara cada proyecto en
  `project-template/stack-profile.yml` (`sql.dialect`).
- **Typecheck/lint de Python**: las herramientas concretas las declara cada proyecto en
  `project-template/stack-profile.yml` (`python.lint` / `python.typecheck`).

## Implementaciones de referencia

Soporte de primera clase del core para Python + SQL + Spark/PySpark (D11):

| Herramienta | Cómo implementa este sensor | Categoría que instancia |
|---|---|---|
| sqlfluff | Linter y formateador SQL con soporte a dialectos (Spark SQL, Snowflake, BigQuery, Hive, etc.); dialecto configurable vía `.sqlfluff` y `stack-profile.yml` (`sql.dialect`) | Lint de SQL con dialecto parametrizable |
| ruff | Linter Python ultrarrápido; reemplaza flake8 + isort + más; configurable vía `pyproject.toml` | Lint de Python |
| mypy | Typecheck estático de Python; configurable vía `pyproject.toml` o `mypy.ini` | Typecheck de Python |
| pre-commit framework | Orquesta sqlfluff, ruff y mypy como hooks de pre-commit; integrable en CI | Orquestación de las dos categorías anteriores |

Otras herramientas (sqlfmt, flake8 + black, dialectos no listados, etc.) son válidas si el
perfil de stack del proyecto las declara; el sensor solo ejecuta lo configurado en el
perfil y reporta el resultado en el formato estándar.

> Nota: la configuración del linter (qué reglas aplican) es responsabilidad del `project-template/data-conventions.md` y del `project-template/stack-profile.yml`, no de este sensor. El sensor solo ejecuta el linter configurado y reporta el resultado en el formato estándar.
