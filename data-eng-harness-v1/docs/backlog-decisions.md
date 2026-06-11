# Backlog de decisiones abiertas (R1-R4)

> Registro vivo de las decisiones de diseño pendientes de resolución.
> Actualizar este fichero cuando una decisión se resuelva o cuando aparezcan nuevas decisiones abiertas.
> Formato de fecha: YYYY-MM-DD.

---

## R1 — Stack de datos del equipo

**Descripción:** El stack concreto de ingeniería de datos del equipo no está confirmado. Las opciones estándar del equipo de The Cocktail son: dbt + Snowflake, dbt + BigQuery, Spark + Delta Lake (Databricks). Puede haber otras combinaciones en proyectos específicos.

**Estado:** Resuelto (2026-06-10) por D11 — Perfil de stack por proyecto (`hard_spec.md §6`). El stack confirmado es heterogéneo y multi-cloud (varía por cliente); el core no fija herramientas concretas, sino que define categorías de sensor y un mecanismo de perfil de stack por proyecto que las instancia, con Python + SQL + Spark/PySpark como soporte de primera clase. Propagado a `core/sensors/catalog.md` y `project-template/stack-profile.yml`.

**Impacto:**
- Bloquea las implementaciones de referencia de los sensores FF-01 a FF-04 y DP-01 a DP-03. Los sensores tienen su especificación completa; la capa de implementación en herramientas concretas (dbt tests, Great Expectations, sqlfluff, etc.) se añade cuando R1 queda confirmado.
- Bloquea `project-template/data-conventions.md §5` (tabla de herramientas por categoría).
- Bloquea la elección de capas en `project-template/CLAUDE.md` (opción A vs opción B).

**Quién puede responderla:** equipo de ingeniería de datos de The Cocktail / responsable de arquitectura de datos.

**Acción al resolver:** actualizar `project-template/data-conventions.md §5`, completar la sección "Implementaciones de referencia" en cada fichero de sensor, registrar ADR-002 en `project-template/docs/architecture/decisions.md`.

---

## R2 — Grado de autonomía y política de aprobación humana

**Descripción:** No está definido cuántas paradas obligatorias impone el arnés ni quién aprueba qué. Las opciones van desde "el humano aprueba cada bloque" (máximo control) hasta "el humano solo interviene cuando el evaluador emite NO APTO dos veces" (máxima autonomía).

**Estado:** Resuelto (2026-06-10) por D12 — Dos políticas de checkpoint (`hard_spec.md §6`). Se instancian dos políticas sobre el parámetro de D9: `validacion-por-tarea` (por defecto en v1) y `full-auto`; los checkpoints duros (operaciones DDL/DML destructivas o sobre sistemas de cliente, y modificaciones de `hard_spec.md`) quedan como invariante no delegable del protocolo en cualquier política. Propagado a `core/orchestration/`.

**Impacto:**
- Afecta a `core/orchestration/stop-conditions.md` (cuántas condiciones de parada).
- Afecta a la parada obligatoria de la skill `adapters/claude-code/skills/harness/SKILL.md §5`.
- Afecta a la frecuencia con la que el humano debe interactuar durante un ciclo completo.

**Quién puede responderla:** equipo de ingeniería de datos de The Cocktail / liderazgo técnico.

**Acción al resolver:** actualizar `core/orchestration/stop-conditions.md` con las condiciones acordadas y la skill con el comportamiento de parada correspondiente.

---

## R3 — Frontera exacta core/adaptador

**Descripción:** La frontera entre `core/` (model-agnostic) y `adapters/claude-code/` (vendor-specific) debe vigilarse activamente. La regla es que ningún fichero del adaptador duplique lógica de los contratos del core y que ningún concepto vendor-specific (Claude Code tools, formatos propietarios) filtre al core.

**Estado:** Vigilar activamente (2026-06-09). La frontera está correctamente establecida en la versión actual; el riesgo es de erosión progresiva al añadir funcionalidades.

**Señales de alerta:**
- Un fichero de `adapters/claude-code/agents/*.md` declara criterios de done, condiciones de parada o definiciones de entradas/salidas que ya deberían estar en `core/contracts/`.
- Un fichero de `core/` menciona Claude Code, `Agent()`, `subagent_type` u otros términos propietarios.

**Impacto si se erosiona:** O3 (model-agnostic) deja de ser viable sin un refactoring costoso. El coste de sustituir Claude Code por otro runtime aumenta con cada filtración.

**Quién puede responderla / vigilarla:** equipo de plataforma / responsable de arquitectura del arnés.

**Acción al detectar erosión:** mover la lógica al fichero `core/contracts/{rol}.md` correspondiente y actualizar el adaptador para que solo referencie el core.

---

## R4 — Alcance ejecutable de evals

**Descripción:** Los 3 evals del baseline (`eval-01`, `eval-02`, `eval-03`) son conceptuales: tienen estructura completa (prompt, checks deterministas, rúbricas, scorecard) pero sin ejecución real. Para ejecutarlos se necesita un proyecto de datos real con dataset, stack confirmado (R1) y pipeline funcionando.

**Estado:** Sin confirmar (2026-06-09). Los evals tienen `status: conceptual` en sus ficheros YAML.

**Impacto:**
- El tuning loop definido en `core/evals/tuning-loop.md` no puede activarse hasta ejecutar un run real.
- El scorecard-schema.json está listo para registrar resultados reales; los campos `run_capture_ref` y `axes.*.raw_score` quedan `null` hasta la primera ejecución.
- La mejora del arnés basada en fallos observados (mecanismo 1 y mecanismo 2 del tuning loop) no puede iniciarse sin runs reales.

**Quién puede responderla:** equipo de ingeniería de datos de The Cocktail (proveer un proyecto real con datos y pipeline).

**Dependencia:** R4 se desbloquea después de resolver R1.

**Acción al resolver:** ejecutar los 3 evals con el proyecto real, rellenar los scorecards con resultados observados, activar el tuning loop según los fallos detectados.
