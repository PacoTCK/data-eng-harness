# core/sensors/ — Catálogo de sensores de calidad

Este directorio contiene la **especificación** del catálogo de sensores del arnés v3.
Los sensores codifican las invariantes mecánicas del trabajo de ingeniería de datos (D4, P8, P9): no hay que explicarle al agente qué revisar a mano — el sensor lo comprueba y le devuelve feedback accionable.

## Qué contiene este directorio

| Fichero/Carpeta | Contenido |
|---|---|
| `catalog.md` | Índice ejecutivo: todos los sensores en una tabla (ID, capa, cuándo corre, invariante) |
| `fast-feedback/` | Sensores deterministas que corren in-session o en CI-pipeline |
| `drift-periodic/` | Sensores periódicos que detectan degradación acumulada |

## Distinción fast-feedback / drift-periódico (P9)

**Fast-feedback** — corren en el momento en que el agente toca un artefacto (al guardar, al hacer commit o en el CI de la PR). Su objetivo es cortar el ciclo de error lo antes posible. Son deterministas: un schema roto es un schema roto, sin ambigüedad.

**Drift periódico** — corren en un scheduler externo al flujo normal de trabajo (diario, semanal). Detectan degradación acumulada que ningún cambio individual dispara: tablas sin owner, assets sin contrato de datos, columnas sin descripción. Son inferenciales o semi-deterministas.

## Formato del feedback (contrato sensor → agente)

Todo sensor que falla emite un bloque estructurado con estas cuatro secciones, siempre en el mismo orden:

```
SENSOR FAIL: {sensor-id}
Asset: {tabla, fichero o pipeline afectado — referencia exacta}
Check: {nombre del check que falló}
Finding: {descripción concreta y referenciable del problema}
Remediation: {instrucción accionable que el agente puede ejecutar}
```

El formato es **idéntico en todos los sensores** para que el agente pueda procesarlo sin lógica condicional. Las propiedades del feedback: referenciable (fichero:línea o tabla:columna), accionable (remediación, no solo diagnóstico), eficiente en tokens (no vuelca output crudo) y estructurado (secciones fijas).

## Cómo añadir un sensor nuevo

1. Decide si es fast-feedback o drift-periódico.
2. Crea un fichero `.md` en la subcarpeta correspondiente siguiendo la plantilla de los sensores existentes.
3. Añade una fila al `catalog.md`.
4. Asigna un ID único con prefijo `FF-` (fast-feedback) o `DP-` (drift-periódico).
5. No codifiques herramientas concretas de stack en el sensor: declara la **categoría** (D11, hard_spec.md §5) que cubre y referencia `project-template/stack-profile.yml` para la implementación concreta que cada proyecto instancia.
