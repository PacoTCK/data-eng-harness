# Effective harnesses for long-running agents

> Nota legal: este archivo es una ficha/resumen, no una copia completa del artículo original.

- **Autor / entidad**: Anthropic Engineering
- **Publicación**: Anthropic
- **Fecha**: 2025-11-26
- **URL original**: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

## Resumen operativo

- Describe el problema de los agentes que trabajan a través de múltiples ventanas de contexto y empiezan cada sesión sin memoria del trabajo anterior.
- Presenta una solución con initializer agent + coding agent, feature_list.json, progress file e init.sh.
- Insiste en progresar una feature cada vez y dejar el entorno en un estado limpio y verificable.

## Uso sugerido en el arnés

- Extraer principios y convertirlos en decisiones de diseño, sensores, estado, tooling o evals.
