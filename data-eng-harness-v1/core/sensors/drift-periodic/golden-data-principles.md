# Sensor: Golden Data Principles

**ID:** `DP-01`
**Capa:** drift-periódico
**Cuándo corre:** periódico — cadencia configurable por proyecto (`⟨a definir por el proyecto⟩`; valor por defecto recomendado: semanal)
**Triggereado por:** scheduler externo (cron, orquestador de pipelines, o equivalente según stack). No lo dispara ningún cambio individual de código o dato.

## Qué comprueba

Este sensor es el equivalente data del sensor de "architecture review" de código: detecta la degradación acumulada de la calidad, gobernanza y arquitectura del ecosistema de datos, que ningún cambio individual dispara pero que se va acumulando con el tiempo.

El nombre "Golden Data Principles" viene del patrón del corpus (P9, D4): las reglas de calidad estructural del ecosistema de datos se declaran como "principios dorados" y se comprueban mecánicamente de forma periódica. Cuando se viola un principio, el sensor emite feedback accionable al agente (o al humano responsable de gobernanza), igual que un sensor fast-feedback, pero con una cadencia más lenta porque los problemas que detecta no son urgentes — son estructurales.

Los principios que comprueba este sensor se declaran en un fichero de configuración del proyecto (`golden-data-principles.yml` o equivalente), lo que permite que cada proyecto añada o quite principios sin modificar el sensor. El sensor lee ese fichero y evalúa cada principio contra el estado actual del catálogo y los contratos de datos.

Algunos ejemplos de principios que el sensor puede evaluar:
- Toda tabla expuesta en la capa marts tiene un owner documentado.
- Todo asset tiene al menos un test de schema activo.
- Ningún modelo tiene más de N dependencias directas (umbral configurable).
- Los pipelines críticos tienen documentado su SLA de freshness en el contrato de datos.
- No existen modelos "huérfanos" (sin consumidores downstream documentados) en la capa marts.

## Checks incluidos

| Check | Tipo | Invariante |
|---|---|---|
| Modelos sin owner documentado | semi-determinista | Toda tabla expuesta tiene un campo `owner` en su contrato de datos |
| Assets sin contrato de datos | determinista | Todo asset de datos tiene un fichero de contrato asociado |
| Assets sin tests de schema activos | determinista | Todo asset tiene al menos un test de schema declarado (FF-01 o equivalente) |
| Pipelines sin SLA de freshness | determinista | Todo pipeline crítico tiene `sla_hours` declarado en el contrato del dataset de salida |
| Modelos con exceso de dependencias | determinista | Ningún modelo supera el umbral `max_dependencies` configurado en los principios dorados |
| Modelos huérfanos en capa marts | semi-determinista | Todos los modelos de la capa marts tienen al menos un consumidor downstream documentado |
| Principios personalizados del proyecto | semi-determinista | Los principios declarados en `golden-data-principles.yml` del proyecto se cumplen |

> Nota: "semi-determinista" indica que el check es determinista en su lógica pero depende de datos de configuración (el fichero de principios) que pueden variar por proyecto.

## Feedback emitido

Cuando el sensor detecta una violación, emite al contexto del agente (o al responsable de gobernanza):

```
SENSOR FAIL: DP-01
Asset: {nombre del modelo, tabla o pipeline afectado}
Check: {nombre del principio violado — ver tabla de checks}
Finding: {descripción concreta — p.ej. "Modelo 'marts.fct_revenue' no tiene owner documentado en su contrato de datos. Lleva 14 días sin owner desde la última revisión del catálogo."}
Remediation: {instrucción accionable — p.ej. "Añadir el campo 'owner: {nombre_equipo_o_persona}' al contrato de datos contracts/marts/fct_revenue.yml y hacer commit. Si el propietario no está claro, escalar al lead de datos."}
```

El sensor emite un bloque `SENSOR FAIL` por cada violación encontrada. Si no hay violaciones, no emite nada (silencio = correcto).

## Categoría de implementación (D11)

Este sensor cubre catálogo/gobernanza, **fuera de las cinco categorías de sensor de
D11** (hard_spec.md §5: lint SQL, typecheck/lint Python, validación de schema/contratos,
freshness/nulls/volumetría, tests de pipeline). El perfil de stack del proyecto
(`project-template/stack-profile.yml`) no tiene un campo dedicado a catálogo/gobernanza
— el campo informativo `plataforma_cliente` (nube, warehouse_lake, orquestación, etc.)
es la referencia más cercana si el proyecto declara una herramienta de catálogo
centralizada (DataHub, Atlan, Dataplex). En su ausencia, la cobertura de este sensor es
responsabilidad del perfil de cada proyecto, sin un campo dedicado en el contrato D11.

## Implementaciones de referencia

| Herramienta | Cómo implementa este sensor | Disponibilidad |
|---|---|---|
| Script Python + catálogo de metadatos | Recorre los ficheros de contrato YAML (instanciados según `schema_contracts.engine` del perfil de stack) y evalúa los principios declarados; genera el informe de violaciones | Implementación de referencia mínima, sin dependencias adicionales |
| dbt meta + custom macros | Lee el campo `meta.owner` en `schema.yml`; una macro comprueba su presencia en todos los modelos de marts | Si el proyecto usa dbt como motor de transformación (declarado en `plataforma_cliente`) |
| dbt exposures + custom checks | Detecta modelos sin exposures downstream (huérfanos) en la capa marts | Si el proyecto usa dbt |
| DataHub / Atlan / Dataplex | Herramienta de data catalog con checks de gobernanza integrados | Si el proyecto tiene un catálogo de datos centralizado declarado en `plataforma_cliente` |

> Nota: la lista de principios dorados a evaluar se declara en el fichero de configuración del proyecto, no en este sensor. El sensor es un motor de evaluación genérico — los principios son el input.
