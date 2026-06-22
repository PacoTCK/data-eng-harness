# CLAUDE.md — [Nombre del proyecto] ⟨pendiente de confirmar⟩

> Punto de entrada para el agente. ~100 líneas. Para profundidad, ver `docs/`.
> No modifiques este fichero para añadir información extensa; añade entradas en `docs/`.

---

## Qué es este proyecto

[RELLENAR: 1-2 frases que describan el objetivo de negocio del proyecto.]

[RELLENAR: 1 frase sobre el equipo propietario y el contexto de The Cocktail en el que opera.]

---

## Stack de datos

El stack concreto de este proyecto se declara en `stack-profile.yml` (perfil de stack,
D11): dialecto SQL, lint/typecheck de Python, motor de schema/contratos, motor de data
quality, framework de tests de pipeline y warehouse de pruebas.

El core (`${CLAUDE_PLUGIN_ROOT}/core/sensors/`) define **categorías** de sensor parametrizables, no
herramientas concretas; `stack-profile.yml` es lo que las instancia para este proyecto.
Soporte de primera clase del core: Python + SQL + Spark/PySpark con `sqlfluff`
(dialecto configurable), `ruff` + `mypy` y DuckDB como warehouse local de pruebas — ver
el bloque `ejemplo_referencia` de `stack-profile.yml` para un perfil ya relleno.

Ver detalle en `docs/references/stack.md` y `data-conventions.md §5`.

---

## Capas de datos

⟨pendiente de confirmar⟩

Elegir **una** de las dos familias al arrancar el proyecto (decisión independiente del
perfil de stack de D11):

- **Opción A — landing / staging / marts** (nomenclatura clásica de Data Warehouse)
- **Opción B — bronze / silver / gold** (nomenclatura medallón, habitual en Data Lakehouse)

Ver invariantes y contratos de cada capa en `docs/architecture/data-layers.md`.

---

## Comandos más usados por el agente

| Comando / acción | Propósito |
|---|---|
| Leer `docs/architecture/decisions.md` | Entender decisiones de arquitectura ya tomadas (ADRs) |
| Leer `docs/pipelines/index.md` | Ver el catálogo de pipelines activos |
| Leer `docs/quality/slas.md` | Consultar SLAs de freshness y calidad por dataset |
| Leer `docs/dudas.md` | Revisar preguntas abiertas antes de asumir valores no confirmados |
| Leer `data-conventions.md` | Consultar naming, contratos de datos y reglas de calidad del proyecto |
| Leer `${CLAUDE_PLUGIN_ROOT}/core/sensors/catalog.md` | Ver qué sensores del arnés cubren cada dimensión de calidad |

---

## Dónde encontrar qué

| Qué busco | Dónde está |
|---|---|
| Objetivo del proyecto (lenguaje natural) | `soft_spec.md` (de él se deriva el `hard_spec.md`, D18) |
| Plan curado: bloques y criterios | `hard_spec.md` (derivado del `soft_spec.md`) |
| Decisiones de arquitectura (ADRs) | `docs/architecture/decisions.md` |
| Capas de datos y sus invariantes | `docs/architecture/data-layers.md` |
| Catálogo de pipelines | `docs/pipelines/index.md` |
| SLAs y contratos de datos | `docs/quality/slas.md` |
| Convenciones de naming y calidad | `data-conventions.md` |
| Perfil de stack (categorías D11 instanciadas) | `stack-profile.yml` |
| Stack confirmado | `docs/references/stack.md` |
| Preguntas abiertas | `docs/dudas.md` |
| Contratos de agente (core) | `${CLAUDE_PLUGIN_ROOT}/core/contracts/` |
| Sensores de calidad (core) | `${CLAUDE_PLUGIN_ROOT}/core/sensors/` |
| Plantillas de estado (core) | `${CLAUDE_PLUGIN_ROOT}/core/state-templates/` |

---

## Frontera core / proyecto

El directorio `${CLAUDE_PLUGIN_ROOT}/core/` contiene el runtime portable del arnés: contratos de agente,
sensores de calidad, plantillas de estado y el protocolo de orquestación.

> `${CLAUDE_PLUGIN_ROOT}` es la raíz del plugin del arnés instalado (la carpeta `data-eng-harness-v1/`
> que Claude Code cachea al hacer `/plugin install`). El agente la resuelve en runtime; no es una
> ruta dentro de este proyecto. Si el arnés se ha copiado/vendorizado en el repo en vez de instalado
> como plugin, sustituir mentalmente `${CLAUDE_PLUGIN_ROOT}` por la ruta a esa carpeta `data-eng-harness-v1/`.

**No se modifica por proyecto.** Si necesitas cambiar el comportamiento de un sensor o
un contrato de agente, abre una duda en `docs/dudas.md` y escala al equipo de plataforma.

Lo que sí es específico de este proyecto y reside aquí:

- `CLAUDE.md` — este índice (stack, capas, comandos)
- `data-conventions.md` — naming, contratos de datos y reglas de calidad propias
- `docs/` — arquitectura, pipelines, SLAs y decisiones del proyecto
- `docs/pipelines/active/` — planes activos de pipelines (equivalente a exec-plans/active/)
