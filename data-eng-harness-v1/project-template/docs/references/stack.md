# Stack de datos — referencia del proyecto

> Propósito: registrar el stack tecnológico confirmado para este proyecto.
> Fuente de verdad: `../../stack-profile.yml` (perfil de stack, D11). Este fichero es una
> vista legible del perfil para humanos; no debe divergir de `stack-profile.yml`.
> El core no fija herramientas concretas — define categorías de sensor parametrizables
> que el perfil de stack instancia. Ver `../../data-conventions.md §5` para el mapeo
> categoría ↔ campo del perfil.

---

## Cómo rellenar este fichero

1. Rellenar `../../stack-profile.yml` primero (es el contrato estructurado que leen los
   agentes y los sensores).
2. Volcar aquí la misma información en formato tabla legible, marcando `Estado` como
   `confirmada` o `descartada`.
3. Actualizar `../../CLAUDE.md` (sección "Stack de datos") con la opción elegida.
4. Actualizar `../architecture/decisions.md` con el ADR correspondiente.

---

## Tabla de stack

| Categoría | Herramienta elegida | Versión | Estado | Campo en `stack-profile.yml` |
|---|---|---|---|---|
| Lint SQL (dialecto) | ⟨pendiente de confirmar⟩ | — | pendiente | `sql.dialect`, `sql.linter` (ref.: sqlfluff) |
| Lint/typecheck Python | ⟨pendiente de confirmar⟩ | — | pendiente | `python.lint`, `python.typecheck` (ref.: ruff + mypy) |
| Validación de schema/contratos | ⟨pendiente de confirmar⟩ | — | pendiente | `schema_contracts.engine` (ref.: pandera) |
| Calidad de datos (freshness/nulls/volumetría) | ⟨pendiente de confirmar⟩ | — | pendiente | `data_quality.engine` (ref.: pandera/pydantic) |
| Tests de pipeline | ⟨pendiente de confirmar⟩ | — | pendiente | `pipeline_tests.framework` (ref.: pytest) |
| Warehouse de pruebas | ⟨pendiente de confirmar⟩ | — | pendiente | `warehouse_pruebas.engine` (ref.: DuckDB) |
| Procesamiento | ⟨pendiente de confirmar⟩ | — | pendiente | `lenguajes.procesamiento` (ref.: Spark/PySpark) |
| Orquestación | ⟨pendiente de confirmar⟩ | — | pendiente | `plataforma_cliente.orquestacion` |
| Warehouse / Lake (cliente) | ⟨pendiente de confirmar⟩ | — | pendiente | `plataforma_cliente.warehouse_lake` |
| Control de versiones | Git | — | confirmada | Estándar del equipo (no forma parte del perfil) |
| CI/CD | ⟨pendiente de confirmar⟩ | — | pendiente | `plataforma_cliente` o herramienta de CI del cliente |

> Ref. = implementación de referencia del core (D11) para Python + SQL + Spark/PySpark,
> usada en el bloque `ejemplo_referencia` de `stack-profile.yml`.

---

## Entornos

| Entorno | Propósito | Warehouse/Lake | Credenciales |
|---|---|---|---|
| desarrollo | Exploración y pruebas locales | ⟨pendiente de confirmar⟩ (ref.: DuckDB vía `warehouse_pruebas.engine`) | [RELLENAR] |
| staging | Validación pre-producción | ⟨pendiente de confirmar⟩ | [RELLENAR] |
| producción | Datos para consumo real | ⟨pendiente de confirmar⟩ | [RELLENAR] |

> Nota: las credenciales nunca se almacenan en este fichero. Referenciar el gestor de
> secretos del equipo (⟨pendiente de confirmar⟩).

---

## Implementaciones de sensores core

La columna "Implementación concreta" se deriva directamente de `../../stack-profile.yml`;
mantener ambas sincronizadas.

| Sensor | Categoría (core) | Implementación concreta (de `stack-profile.yml`) |
|---|---|---|
| FF-01 Schema Contract | Validación de schema/contratos de datos | ⟨pendiente de confirmar⟩ — ver `schema_contracts.engine` |
| FF-02 Data Quality | Freshness/nulls/volumetría | ⟨pendiente de confirmar⟩ — ver `data_quality.engine` |
| FF-03 Pipeline Tests | Tests de pipeline | ⟨pendiente de confirmar⟩ — ver `pipeline_tests.framework` |
| FF-04 SQL/Python Lint | Lint SQL (dialecto) + lint/typecheck Python | ⟨pendiente de confirmar⟩ — ver `sql.*` y `python.*` |
| DP-01 Golden Data Principles | drift-periódico | ⟨pendiente de confirmar⟩ |
| DP-02 Lineage Staleness | drift-periódico | ⟨pendiente de confirmar⟩ |
| DP-03 Catalog Completeness | drift-periódico | ⟨pendiente de confirmar⟩ |
