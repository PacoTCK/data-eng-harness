# Reading Pack — Harness engineering para tu arnés (v2)

> Preparado para usarlo como contexto de diseño en un repo de arnés / agent harness.
>
> **Contenido**: bibliografía priorizada, preguntas de lectura, mapa de conceptos y ficheros Markdown por artículo.
>
> **Nota legal**: este paquete incluye **resúmenes y fichas en Markdown**, no el texto completo de los artículos externos. Los tres ficheros del usuario deben añadirse manualmente dentro de `user-supplied-articles/`.

## Objetivo

Este reading pack está pensado para ayudarte a diseñar un arnés portable y compartible, orientado a agentes de coding, con foco en:

- estructura del repo,
- handoff entre sesiones,
- sensores y guardrails,
- diseño de herramientas / MCP,
- evaluación rigurosa de la eficacia del arnés,
- y patrones de orquestación reutilizables.

---

## Orden recomendado de lectura

### 1) Marco conceptual
1. `articles/01-harness-engineering-for-coding-agent-users.md`
2. `user-supplied-articles/260211_openai_harness_engineering_codex.md`

### 2) Trabajo largo, multi-sesión y handoff
3. `articles/02-effective-harnesses-for-long-running-agents.md`
4. `articles/03-harness-design-for-long-running-application-development.md`
5. `user-supplied-articles/260324_anthropic_harness_design.md`

### 3) Calidad, sensores y tooling
6. `articles/04-maintainability-sensors-for-coding-agents.md`
7. `articles/05-writing-effective-tools-for-agents.md`

### 4) Evaluación
8. `articles/06-testing-agent-skills-systematically-with-evals.md`

### 5) Interfaces, configuración y estructura del repo
9. `articles/07-swe-agent-agent-computer-interfaces.md`
10. `articles/08-decoding-the-configuration-of-ai-coding-agents.md`

### 6) Patrón de orquestación / loop operativo
11. `user-supplied-articles/250714_huntley_ralph_wiggum_technique.md`

---

## Qué debe extraer tu agente de cada bloque

### A. Qué es un arnés
- Definición operacional de harness.
- Diferencia entre contexto, guías, sensores y tooling.
- Qué parte del arnés pertenece al core y cuál a la configuración del proyecto.
- Qué decisiones deben vivir en prompts, qué decisiones deben vivir en ficheros y qué decisiones deben convertirse en reglas mecánicas.

**Lecturas clave de este bloque**:
- `articles/01-harness-engineering-for-coding-agent-users.md`
- `user-supplied-articles/260211_openai_harness_engineering_codex.md`

---

### B. Cómo se sostiene en tareas largas
- Artefactos de handoff.
- Progreso incremental y definición de “done”.
- Cuándo usar contexto persistido, context reset, compaction o sesiones separadas.
- Cómo evitar que el agente “declare victoria” demasiado pronto.
- Cómo mantener memoria útil sin depender de una ventana de contexto única.

**Lecturas clave de este bloque**:
- `articles/02-effective-harnesses-for-long-running-agents.md`
- `articles/03-harness-design-for-long-running-application-development.md`
- `user-supplied-articles/260324_anthropic_harness_design.md`

---

### C. Cómo se controla la calidad
- Qué sensores van en tiempo de desarrollo.
- Qué sensores van a CI y cuáles periódicamente.
- Cómo redactar feedback consumible por agentes.
- Qué invariantes arquitectónicas deben estar codificadas y cuáles pueden quedar como guideline.
- Qué elementos mejoran la legibilidad del repo para un agente.

**Lecturas clave de este bloque**:
- `articles/04-maintainability-sensors-for-coding-agents.md`
- `articles/05-writing-effective-tools-for-agents.md`
- `user-supplied-articles/260211_openai_harness_engineering_codex.md`

---

### D. Cómo se prueba
- Qué evals definen éxito del arnés.
- Qué señales son deterministas y cuáles rubric-based.
- Cómo comparar variantes de configuración, prompts o skills.
- Qué métricas deben vigilarse: correctness, process, style, efficiency, drift y maintainability.
- Cómo convertir fallos observados en nuevos tests o constraints del arnés.

**Lecturas clave de este bloque**:
- `articles/06-testing-agent-skills-systematically-with-evals.md`
- `user-supplied-articles/260324_anthropic_harness_design.md`
- `user-supplied-articles/260211_openai_harness_engineering_codex.md`

---

### E. Cómo estructurar el entorno del agente
- Qué relación hay entre interfaz del agente, ergonomía, tooling y rendimiento.
- Qué suele vivir en `CLAUDE.md`, `AGENTS.md` y ficheros equivalentes.
- Qué partes del conocimiento deben ser visibles desde el repo para que el agente razone bien.
- Cómo combinar documentación breve de entrada con documentación profunda enlazada.

**Lecturas clave de este bloque**:
- `articles/07-swe-agent-agent-computer-interfaces.md`
- `articles/08-decoding-the-configuration-of-ai-coding-agents.md`
- `user-supplied-articles/260211_openai_harness_engineering_codex.md`

---

### F. Qué patrón de orquestación conviene adoptar
- Cuándo empezar por un loop simple.
- Cuándo usar un patrón initializer + coding agent.
- Cuándo introducir planner / generator / evaluator.
- Qué trade-offs hay entre simplicidad operativa y sofisticación del arnés.
- Cómo capturar el aprendizaje del loop para que el sistema mejore con el tiempo.

**Lecturas clave de este bloque**:
- `user-supplied-articles/250714_huntley_ralph_wiggum_technique.md`
- `articles/02-effective-harnesses-for-long-running-agents.md`
- `articles/03-harness-design-for-long-running-application-development.md`
- `user-supplied-articles/260324_anthropic_harness_design.md`

---

## Mapa de conceptos para tu repo

### Conceptos estructurales
- **Harness core**: runtime, bucles, contratos, hooks, orchestration.
- **Project config**: arquitectura, comandos, convenciones, tests, overview.
- **State artifacts**: feature list, progress log, sprint contracts, checklists.
- **Sensors**: typecheck, lint, tests, structure, mutation, reviews.
- **Agent interfaces**: tools, MCPs, wrappers, CLI affordances.
- **Knowledge base**: docs versionados, índices, referencias y fuentes de verdad en repo.
- **Evals**: traces, graders, datasets, métricas y scorecards.
- **Loop pattern**: una tarea por iteración, progreso persistido y ajuste continuo del sistema.

### Conceptos de diseño
- **Portable core**: lógica reutilizable entre repos.
- **Repo-local truth**: todo lo importante debe ser accesible desde el repositorio.
- **Progressive disclosure**: un `AGENTS.md` corto que apunte a documentación más profunda.
- **Architecture invariants**: límites y dependencias permitidas, comprobables mecánicamente.
- **Legibility for agents**: estructuras, nombres, tooling y documentación optimizados para que el agente razone bien.
- **Human-on-the-loop**: el humano mejora el loop, no ejecuta manualmente cada paso.

---
