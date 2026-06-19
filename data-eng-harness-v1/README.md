# Data Engineering Agent Harness v3

> Arnés portable para ingenieros de datos de The Cocktail. Implementa un ciclo de 4 agentes especializados (planificador, navegador, implementador, evaluador) con human-on-the-loop. Core model-agnostic; la capa de ejecución es específica de Claude Code y sustituible.

---

## Qué es este arnés

El arnés es un sistema de orquestación de agentes diseñado para que los ingenieros de datos de The Cocktail puedan ejecutar tareas complejas (modelado dbt, pipelines de ingestión, validación de schema, diseño de arquitectura de datos) con agentes de coding de forma controlada, reproducible y auditable.

Cada tarea sigue un ciclo estructurado: el **planificador** decide qué hacer y produce un contrato JSON de handoff; el **implementador** produce los artefactos; el **evaluador** verifica contra los criterios del contrato; el **planificador** cierra o reintenta. El humano supervisa el bucle y toma las decisiones que los agentes no pueden tomar. El estado persiste en el repositorio entre sesiones: si la sesión se interrumpe, la próxima retoma el trabajo desde el punto exacto de interrupción.

---

## Estructura

```
data-eng-harness-v1/               # Raíz del plugin (${CLAUDE_PLUGIN_ROOT}). Autocontenido.
│
├── .claude-plugin/                # Plugin manifest (plugin.json) + marketplace local. plugin.json
│                                  #   apunta agents/skills a adapters/claude-code/ y deja la raíz aquí.
│
├── core/                          # Runtime portable, model-agnostic. No modificar por proyecto.
│   ├── orchestration/             # Ciclo de 4 agentes (cycle.md) y condiciones de parada
│   ├── contracts/                 # Contratos de rol: entradas, salidas, criterio de done
│   ├── state-templates/           # Capa de estado (D10): state.json, progress.md, contrato JSON, dudas
│   ├── sensors/                   # Especificaciones de los 7 sensores de calidad de datos
│   └── evals/                     # Baseline de evals: 3 escenarios de prueba del arnés
│
├── adapters/
│   └── claude-code/               # Capa vendor-specific para Claude Code. Sustituible (O3).
│       ├── agents/                # Los 4 agentes como subagentes Claude Code
│       └── skills/harness/        # Skill del ciclo completo (trigger + 5 pasos)
│
├── project-template/              # Plantilla base para cada nuevo proyecto de datos.
│   ├── CLAUDE.md                  # Índice del proyecto (~80-100 líneas)
│   ├── data-conventions.md        # Naming, capas, contratos de datos, invariantes de calidad
│   └── docs/                      # Architecture, pipelines, quality, references, dudas
│
├── docs/                          # Documentación del propio arnés
│   ├── best-practices.md          # Principios P1-P14 con trazabilidad al corpus
│   └── backlog-decisions.md       # Decisiones abiertas R1-R4 (backlog vivo)
│
├── DESIGN.md                      # Documento de diseño completo con trazabilidad
└── README.md                      # Este fichero
```

| Capa | Propósito | Modificable por proyecto |
|---|---|---|
| `core/` | Runtime portable y model-agnostic | No |
| `adapters/claude-code/` | Ejecución en Claude Code | No (sustituir completo para otro runtime) |
| `project-template/` | Base del proyecto de datos concreto | Sí (copiar y rellenar) |
| `docs/` | Documentación del arnés | Solo el equipo de plataforma |

---

## Instalación y compartición (The Cocktail)

### Vía recomendada: instalación remota desde Bitbucket (`/plugin marketplace add`)

El repositorio Bitbucket del equipo (`https://bitbucket.org/the-cocktail/data-eng-harness-central.git`) expone un marketplace en su raíz (`.claude-plugin/marketplace.json`) que apunta, vía `git-subdir`, a la raíz del arnés (`data-eng-harness-v1/`) — el plugin completo y autocontenido, con `core/`, `project-template/` y `docs/` incluidos. Para instalarlo, ejecutar dentro de Claude Code:

```
/plugin marketplace add https://bitbucket.org/the-cocktail/data-eng-harness-central.git
/plugin install data-eng-harness-v1@data-eng-harness-v1-marketplace
```

El primer comando registra el marketplace remoto; el segundo instala el plugin `data-eng-harness-v1` desde ese marketplace. No requiere clonar el repo manualmente ni configurar rutas locales.

Una vez instalado, el comando `inicia el planificador para realizar el proyecto usando agentes` activa la skill del ciclo de 4 agentes.

### Vía local (desarrollo/pruebas sobre un checkout del repo)

Si ya se tiene el repo clonado localmente, se puede registrar el marketplace del propio arnés (su raíz, no el adaptador) en lugar del marketplace raíz del repo:

```bash
git clone https://bitbucket.org/the-cocktail/data-eng-harness-central.git
cd data-eng-harness
```

```
/plugin marketplace add ./data-eng-harness-v1
/plugin install data-eng-harness-v1@data-eng-harness-v1-marketplace
```

Alternativamente, sin marketplace, apuntando Claude Code directamente a la raíz del plugin (carga solo para esa sesión, útil en desarrollo):

```bash
claude --plugin-dir data-eng-harness-v1/
```

O añadir la ruta `data-eng-harness-v1/` como plugin en la configuración del proyecto de Claude Code (`settings.json` o equivalente). En todos los casos la raíz del plugin es `data-eng-harness-v1/` (contiene `.claude-plugin/plugin.json`, `core/`, `project-template/` y `docs/`); el adaptador Claude Code vive en `adapters/claude-code/` y el manifest lo referencia con los campos `agents`/`skills`.

### Compartición dentro de The Cocktail

El arnés se distribuye como repositorio git. Cualquier miembro del equipo puede instalarlo de forma remota con los dos comandos `/plugin marketplace add` + `/plugin install` de arriba, sin clonar nada. No hay dependencias de runtime adicionales más allá de Claude Code.

Para compartir en un nuevo proyecto interno: incluir el repositorio como submódulo git o copiar el directorio `data-eng-harness-v1/` al repo del proyecto.

---

## Cómo arrancar el ciclo

### Requisito previo: tener hard_spec.md + la capa de estado (state.json/progress.md) en el repo

El planificador lee `hard_spec.md` para obtener los objetivos y criterios de cada bloque, y la capa de estado (`state.json` + `progress.md`, D10) para conocer el estado de avance. Si el proyecto aún no tiene estos ficheros, crearlos antes de iniciar el ciclo a partir de las plantillas en `core/state-templates/`.

### Protocolo de sesión (D9): arranque y cierre

El arnés ejecuta **una tarea por sesión**: cada sesión arranca con una secuencia de re-entrada, ejecuta el ciclo de 4 agentes y termina con una secuencia de cierre. Ver `core/orchestration/session-protocol.md` para la definición completa.

**Al arrancar una sesión** (incluida la primera vez):

1. Leer `state.json` para identificar la tarea activa (`in_progress`) o, si no hay ninguna, la siguiente `pending`.
2. Leer la última entrada de `progress.md` (qué se hizo en la sesión anterior, siguiente paso anotado).
3. Verificar que el repositorio está en un estado consistente (p. ej. `git log` reciente coincide con `state.json`/`progress.md`).
4. Invocar al planificador con ese contexto.

**Al cerrar una sesión:**

1. El evaluador emite veredicto `APTO`/`NO APTO`.
2. Solo si `APTO`: commit de los artefactos producidos.
3. El planificador actualiza el campo de estado en `state.json` y añade una entrada nueva a `progress.md`.
4. Checkpoint humano: revisar veredicto, diff/artefactos y el estado actualizado (`state.json` + `progress.md`).
5. Limpiar el contexto (p. ej. `/clear` en Claude Code) y, si se decide continuar, arrancar una sesión fresca que repite el paso de arranque.

### Invocar el ciclo

Escribir al agente:

```
inicia el planificador para realizar el proyecto usando agentes
```

El ciclo de 4 agentes se ejecuta dentro de la sesión:

1. **Planificador** — identifica la tarea activa o siguiente pendiente en `state.json`, produce el contrato JSON en `tasks/`, marca la tarea como `in_progress` en `state.json`.
2. **Implementador** — lee el contrato JSON, confirma `pre_handoff_validation`, produce los artefactos.
3. **Evaluador** — verifica los artefactos contra los `acceptance_criteria` del contrato, emite `APTO` o `NO APTO`.
4. **Planificador (cierre)** — actualiza el contrato a `complete` o `failed`; actualiza el campo de estado en `state.json` y añade una entrada a `progress.md`.
5. **Parada obligatoria** — al cerrar la sesión (D9), el orquestador ejecuta el checkpoint humano antes de que se decida arrancar la siguiente sesión.

### Dónde ver el estado

| Fichero | Qué muestra |
|---|---|
| `state.json` | Estado de cada bloque/tarea: `pending` / `in_progress` / `complete` / `failed`. |
| `progress.md` | Notas de sesión append-only: qué se hizo, veredicto del evaluador, bugs, siguiente paso. |
| `tasks/{B}-{slug}.json` | Contrato activo: `status`, `acceptance_criteria`, `audit_trail`. |

---

## Usar la plantilla de proyecto

### 1. Copiar la plantilla

```bash
cp -r data-eng-harness-v1/project-template/ {ruta_del_nuevo_proyecto}/
```

### 2. Rellenar CLAUDE.md

Editar `CLAUDE.md` del nuevo proyecto con:

- Nombre y objetivo de negocio del proyecto (1-2 frases).
- Equipo propietario.
- Stack elegido (marcar al confirmar R1).
- Capas de datos elegidas (opción A: landing/staging/marts o opción B: bronze/silver/gold).

### 3. Confirmar R1

Antes de usar los sensores en producción, confirmar el stack del equipo (R1) y actualizar `data-conventions.md §5` y `docs/references/stack.md`. Sin R1 confirmado, las implementaciones de referencia de los sensores quedan como `⟨pendiente⟩`.

### 4. Registrar decisiones de arquitectura

Crear el primer ADR en `docs/architecture/decisions.md` con la decisión de capas (ADR-001) y stack (ADR-002) para que el agente tenga contexto verificable sin preguntar.

---

## Arnés model-agnostic (O3)

### Qué es específico de Claude Code

Todo lo que vive en `adapters/claude-code/` es vendor-specific:

- Los ficheros `agents/*.md` usan el formato de subagente de Claude Code (frontmatter YAML + body con `tools`).
- La skill `skills/harness/SKILL.md` usa el trigger y el formato de Claude Code.
- El plugin manifest `.claude-plugin/` es específico del marketplace de Claude Code.

### Qué es portable

Todo lo que vive en `core/` es model-agnostic:

- `core/contracts/*.md` — contratos de rol en lenguaje natural; no mencionan Claude Code.
- `core/orchestration/` — ciclo y condiciones de parada en lenguaje natural.
- `core/state-templates/` — plantillas JSON/Markdown; compatibles con cualquier runtime.
- `core/sensors/` — especificaciones de sensores; no dependen del modelo.
- `core/evals/` — evals en YAML; independientes del runtime.

### Cómo sustituir Claude Code

1. Crear `adapters/{nuevo-runtime}/` con los cuatro agentes y la skill equivalente.
2. Los agentes del nuevo adaptador referencian los mismos ficheros de `core/contracts/`.
3. No modificar ningún fichero de `core/`.
4. Actualizar el plugin manifest si el nuevo runtime tiene su propio sistema de plugins.

---

## Decisiones abiertas

Estas decisiones no están resueltas y afectan al uso del arnés en producción. Se registran como backlog vivo en `docs/backlog-decisions.md`.

| ID | Descripción | Estado | Impacto en bloques |
|---|---|---|---|
| R1 | Stack de datos del equipo (dbt, Airflow, Spark, Snowflake/BigQuery) | Resuelto por D11 — Perfil de stack por proyecto (`docs/backlog-decisions.md`) | Bloquea implementaciones de referencia de sensores FF-01 a FF-04, DP-01 a DP-03; bloquea `data-conventions.md §5`. |
| R2 | Grado de autonomía / política de aprobación humana por bloque | Resuelto por D12 — Dos políticas de checkpoint (`docs/backlog-decisions.md`) | Afecta al número de paradas obligatorias del ciclo y al diseño de `stop-conditions.md`. |
| R3 | Frontera exacta core/adaptador (vigilar que no filtre Claude Code al core) | Vigilar activamente | Si el adaptador duplica lógica del core, la frontera se erosiona y O3 deja de ser viable. |
| R4 | Alcance ejecutable de evals (baseline conceptual sin proyecto de datos real) | Sin confirmar | Los 3 evals del baseline son conceptuales. Se activan con un proyecto real que provea datasets y stack. |
