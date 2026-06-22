# Convenciones de datos del proyecto

> Lectura recomendada antes de que el agente modifique cualquier modelo, pipeline o
> contrato de datos.
> Coherente con los sensores de `${CLAUDE_PLUGIN_ROOT}/core/sensors/` y con la terminología de los
> contratos de datos que esos sensores consumen (campos `owner`, `sla_hours`, `max_dependencies`).

---

## 1. Arquitectura por capas

⟨pendiente de confirmar⟩ — cada proyecto elige **una** de las dos opciones siguientes al
arrancar (decisión independiente del perfil de stack de D11, ver `stack-profile.yml`).

### Opción A — landing / staging / marts

```
landing   →  datos en bruto, sin transformación; 1:1 con la fuente
staging   →  limpieza, cast de tipos, deduplicación; 1 modelo por fuente
marts     →  tablas expuestas al negocio; modelo estrella o plano; joins y métricas
```

Invariantes de la opción A:
- Ningún modelo de `staging` lee de otro modelo de `staging` (solo de `landing`).
- Los modelos de `marts` son los únicos que se exponen a herramientas de BI o consumidores externos.
- Toda tabla en `marts` tiene un contrato de datos declarado (ver §3).

### Opción B — bronze / silver / gold (medallón)

```
bronze  →  ingesta fiel a la fuente; raw data; no modificar valores
silver  →  datos limpios y conformados; schema estable; unificación de fuentes
gold    →  tablas listas para consumo analítico o de producto; métricas y features
```

Invariantes de la opción B:
- Ningún dato en `bronze` se expone directamente a consumidores externos.
- Los modelos de `gold` son los únicos expuestos a BI o APIs de producto.
- Toda tabla en `gold` tiene un contrato de datos declarado (ver §3).

> Nota: rellenar la sección de la opción elegida en `docs/architecture/data-layers.md`
> y registrar la decisión en `docs/architecture/decisions.md` (ADR-001).

---

## 2. Naming conventions

### 2.1 Tablas y modelos

| Elemento | Regla | Ejemplo |
|---|---|---|
| Nombre de tabla/modelo | `snake_case`, sin mayúsculas | `orders_cleaned` |
| Prefijo por capa (opción A) | `lnd_`, `stg_`, `mrt_` | `stg_orders`, `mrt_revenue_daily` |
| Prefijo por capa (opción B) | `brz_`, `slv_`, `gld_` | `slv_orders`, `gld_revenue_daily` |
| Nombre de fichero de modelo | igual que el nombre de tabla | `stg_orders.sql` |
| Nombre de contrato de datos | igual que el nombre de tabla + `.yml` | `stg_orders.yml` |

> Nota: los prefijos son ⟨pendiente de confirmar⟩ hasta que el proyecto elija la opción de capas (§1).

### 2.2 Columnas

Columnas de auditoría obligatorias en **todos** los modelos:

| Columna | Tipo | Nulabilidad | Descripción |
|---|---|---|---|
| `id` | STRING o INTEGER | NOT NULL | Identificador único del registro en esta tabla |
| `created_at` | TIMESTAMP | NOT NULL | Timestamp de creación del registro en la fuente |
| `updated_at` | TIMESTAMP | NOT NULL | Timestamp de la última actualización del registro |
| `_source` | STRING | NOT NULL | Nombre del sistema de origen del dato |
| `_loaded_at` | TIMESTAMP | NOT NULL | Timestamp de ingestión en el warehouse/lake (generado por el pipeline) |

Reglas adicionales:
- Nombres de columnas en `snake_case`, sin abreviaturas no estándar.
- Las columnas de clave de negocio deben llevar el sufijo `_id` o `_key` según corresponda.
- Las columnas de flags booleanos deben empezar con `is_` o `has_`.
- Las columnas monetarias llevan el sufijo `_amount` y la divisa se declara en el contrato.

### 2.3 Contratos de datos

El fichero de contrato se sitúa junto al modelo que documenta.
Naming: `{nombre_tabla}.yml` en la misma carpeta que el modelo `.sql` o `.py`.

---

## 3. Contratos de datos

Un contrato de datos es el fichero YAML que declara explícitamente el schema, las reglas
de calidad y los metadatos de gobernanza de un dataset.
El sensor FF-01 (Schema Contract) verifica que el schema real coincida con este contrato.
El sensor FF-02 (Data Quality) usa los campos `sla_hours`, `min_rows`, `max_rows` y las
reglas de columna para verificar freshness, volume y validez.

### Estructura del contrato (plantilla)

```yaml
# Contrato de datos — {nombre_tabla}
# Coherente con core/sensors/fast-feedback/schema-contract.md (FF-01)
# y core/sensors/fast-feedback/data-quality.md (FF-02)

version: "1.0"
asset: "{schema}.{nombre_tabla}"
owner: "[RELLENAR: nombre o equipo]"              # Campo requerido por DP-03
sla_hours: 0                                       # ⟨RELLENAR — ver docs/quality/slas.md⟩
max_dependencies: 0                                # Número máximo de modelos downstream directos

columns:
  - name: id
    type: STRING                                   # ⟨ajustar al tipo del warehouse/engine declarado en stack-profile.yml⟩
    nullable: false
    description: "Identificador único del registro"
    critical: true                                 # Columna crítica: FF-02 aplica not-null + unicidad
  - name: created_at
    type: TIMESTAMP
    nullable: false
    description: "Timestamp de creación en la fuente"
  - name: updated_at
    type: TIMESTAMP
    nullable: false
    description: "Timestamp de última actualización"
  - name: _source
    type: STRING
    nullable: false
    description: "Sistema de origen del dato"
  - name: _loaded_at
    type: TIMESTAMP
    nullable: false
    description: "Timestamp de ingestión generado por el pipeline"
  # [RELLENAR: columnas específicas del modelo]

quality:
  primary_key: [id]                                # Columnas que forman la clave primaria
  not_null: [id, created_at, updated_at, _source, _loaded_at]
  unique: [id]
  # [RELLENAR: reglas adicionales de calidad]
  # freshness:
  #   column: updated_at
  #   warn_after: sla_hours                         # Leído del campo sla_hours arriba
  # row_count:
  #   min: 0                                        # ⟨RELLENAR⟩
  #   max: ~                                        # Opcional

tags:
  - layer: "⟨pendiente⟩"                          # landing|staging|marts o bronze|silver|gold — según opción elegida en §1
  - domain: "[RELLENAR]"
```

---

## 4. Reglas de calidad (invariantes mecánicas)

Las reglas siguientes son invariantes del proyecto. Los sensores del core las verifican
automáticamente; no es necesario añadirlas a mano en cada pipeline.

| Invariante | Sensor que la verifica | Referencia |
|---|---|---|
| Schema de tabla coincide con el contrato declarado | FF-01 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/schema-contract.md` |
| Columnas críticas no contienen nulos | FF-02 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/data-quality.md` |
| Clave primaria sin duplicados | FF-02 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/data-quality.md` |
| Dataset actualizado dentro del SLA declarado | FF-02 | `docs/quality/slas.md` |
| Volume dentro de umbrales `min_rows` / `max_rows` | FF-02 | `docs/quality/slas.md` |
| Tests unitarios e integración del pipeline pasan | FF-03 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/pipeline-tests.md` |
| SQL y Python cumplen reglas de estilo del equipo | FF-04 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/sql-lint.md` |
| Todo asset en capa expuesta tiene owner documentado | DP-03 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/drift-periodic/catalog-completeness.md` |

Reglas adicionales **propias de este proyecto**:

- [RELLENAR: reglas de negocio que los sensores del core no cubren.]
- Ejemplo: "Los modelos de `marts`/`gold` no pueden tener más de `max_dependencies`
  consumidores directos sin revisión de arquitectura."

---

## 5. Stack y herramientas

El stack de datos del equipo es heterogéneo y multi-cloud (varía por cliente); el core
no fija herramientas concretas, sino **categorías de sensor** parametrizables que cada
proyecto instancia mediante su **perfil de stack** (D11; ver `${CLAUDE_PLUGIN_ROOT}/DESIGN.md` §9).

El perfil de stack de este proyecto se declara en `stack-profile.yml` (raíz del
proyecto). Cada campo de ese fichero instancia una categoría de sensor de
`${CLAUDE_PLUGIN_ROOT}/core/sensors/`:

| Categoría de sensor (core) | Campo de `stack-profile.yml` | Opciones habituales |
|---|---|---|
| Lint de SQL con dialecto parametrizable (FF-04) | `sql.dialect`, `sql.linter` | sqlfluff con dialecto Spark SQL, Snowflake, BigQuery, Hive, T-SQL, etc. |
| Typecheck/lint de Python (FF-04) | `python.lint`, `python.typecheck` | ruff, mypy, flake8, black |
| Validación de schema/contratos de datos (FF-01) | `schema_contracts.engine` | pandera, dbt schema tests, Great Expectations, Soda Core |
| Freshness/nulls/volumetría (FF-02) | `data_quality.engine` | pandera/pydantic, dbt-expectations, Great Expectations, Soda Core |
| Tests de pipeline (FF-03) | `pipeline_tests.framework` | pytest, dbt-unit-testing, pytest-dbt-core, Airflow TaskFlow tests |
| Warehouse de pruebas (entorno local FF-01/FF-02/FF-03) | `warehouse_pruebas.engine` | DuckDB |

Implementación de referencia del core (D11) — soporte de primera clase para
**Python + SQL + Spark/PySpark**: `sqlfluff` (dialecto configurable vía perfil),
`ruff` + `mypy` para Python, y **DuckDB** como warehouse local de pruebas. El bloque
`ejemplo_referencia` de `stack-profile.yml` es un perfil completo y copiable con esta
implementación.

Información de plataforma del cliente (nube, warehouse/lake, orquestación, integración,
BI) que no condiciona directamente a los sensores del core se declara en
`stack-profile.yml` (`plataforma_cliente`) y en `docs/references/stack.md`.

> Nota: al confirmar el perfil de stack del proyecto, rellenar `stack-profile.yml`,
> actualizar `docs/references/stack.md` y registrar el ADR correspondiente en
> `docs/architecture/decisions.md`.

---

## 6. Frontera core / proyecto

### Qué viene del core (no modificar)

Los sensores del core (`${CLAUDE_PLUGIN_ROOT}/core/sensors/`) son especificaciones portables que aplican
a todos los proyectos del equipo de ingeniería de datos. Definen **qué** se comprueba
y **qué feedback** emite cada sensor. No dependen del stack concreto del proyecto.

| Elemento core | Ruta | Qué hace |
|---|---|---|
| Sensor FF-01 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/schema-contract.md` | Verifica schema vs contrato |
| Sensor FF-02 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/data-quality.md` | Verifica calidad, freshness, volume |
| Sensor FF-03 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/pipeline-tests.md` | Verifica tests del pipeline |
| Sensor FF-04 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/fast-feedback/sql-lint.md` | Verifica estilo de código |
| Sensor DP-01 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/drift-periodic/golden-data-principles.md` | Golden data audit periódico |
| Sensor DP-02 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/drift-periodic/lineage-staleness.md` | Detección de lineage desactualizado |
| Sensor DP-03 | `${CLAUDE_PLUGIN_ROOT}/core/sensors/drift-periodic/catalog-completeness.md` | Completitud del catálogo |

### Qué es específico de este proyecto (aquí, en project-template/)

- `CLAUDE.md` — índice: stack, capas, comandos del agente para este proyecto.
- `data-conventions.md` — este fichero: naming, contratos, reglas de calidad propias.
- `stack-profile.yml` — perfil de stack (D11): instancia las categorías de sensor del core.
- `docs/architecture/` — decisiones de arquitectura y capas elegidas.
- `docs/pipelines/` — catálogo y planes de pipelines del proyecto.
- `docs/quality/slas.md` — SLAs acordados con el equipo o cliente.
- `docs/quality/contracts/` — contratos de datos por asset (instancias de la plantilla del §3).
- `docs/references/stack.md` — stack confirmado (sincronizado con `stack-profile.yml`).
- `docs/dudas.md` — backlog de preguntas abiertas del proyecto.
