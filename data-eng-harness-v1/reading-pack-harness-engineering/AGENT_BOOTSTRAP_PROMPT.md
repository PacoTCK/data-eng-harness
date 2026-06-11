# Prompt de sistema/contexto para un agente constructor del arnés

Usa el contenido de `README.md`, `articles/` y `user-supplied-articles/` como corpus de referencia.

## Objetivo
Diseñar la estructura inicial de un arnés portable para agentes de coding, separando:
- core reusable,
- configuración específica del proyecto,
- artefactos de estado,
- sensores,
- evals,
- y patrón de orquestación.

## Instrucciones
1. Lee `README.md` para entender el mapa de lectura y prioridades.
2. Recorre `articles/` y `user-supplied-articles/` y extrae principios de diseño, no citas largas.
3. Propón una estructura concreta de carpetas y archivos.
4. Justifica cada artefacto con uno o más principios del corpus.
5. Si propones sensores o evals, indica si son in-session, CI/pipeline o periódicos.
6. Evita mezclar core reusable con instrucciones de proyecto.
7. Genera siempre una definición verificable de “done” para cada pieza que diseñes.

## Prioridades especiales
Da prioridad a los ficheros:
- `user-supplied-articles/260211_openai_harness_engineering_codex.md`
- `user-supplied-articles/260324_anthropic_harness_design.md`
- `user-supplied-articles/250714_huntley_ralph_wiggum_technique.md`

## Entregables esperados
- estructura del repo,
- lista de artefactos de estado,
- sensores iniciales,
- baseline de evals,
- backlog de decisiones abiertas.
