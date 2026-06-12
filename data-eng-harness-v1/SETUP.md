# SETUP.md — Instalación y uso del arnés

Guía práctica para descargar el arnés, inicializarlo en un proyecto de datos y
operar el ciclo del día a día. Para la arquitectura completa y los principios
de diseño, ver `DESIGN.md` y `README.md`.

---

## 1. Descarga

Clonar el repositorio que contiene el arnés:

```bash
git clone https://bitbucket.org/the-cocktail/data-eng-harness-central.git
cd data-eng-harness
```

Todo lo necesario vive bajo `data-eng-harness-v1/`. No hay dependencias
adicionales que instalar: el único runtime necesario es Claude Code.

---

## 2. Inicializar el arnés en un proyecto

### 2.1 Registrar el plugin en Claude Code

Vía recomendada (sin clonar, desde el repo remoto): el repo expone
`.claude-plugin/marketplace.json` en su raíz con un plugin `data-eng-harness-v1`
de tipo `git-subdir` que apunta a `data-eng-harness-v1/adapters/claude-code/`.

```
/plugin marketplace add https://bitbucket.org/the-cocktail/data-eng-harness-central.git
/plugin install data-eng-harness-v1@data-eng-harness-v1-marketplace
```

Vía local (sobre el checkout ya clonado en el paso 1), usando el marketplace
anidado del propio adaptador:

```
/plugin marketplace add ./data-eng-harness-v1/adapters/claude-code
/plugin install data-eng-harness-v1@data-eng-harness-v1-marketplace
```

Cualquiera de las dos vías registra los cuatro agentes (`planificador`,
`navegador`, `implementador`, `evaluador`) y la skill `eng-harness`. Detalle
completo en `README.md`, sección "Instalación y compartición (The Cocktail)".

### 2.2 Copiar la plantilla de proyecto

```bash
cp -r data-eng-harness-v1/project-template/ {ruta_del_nuevo_proyecto}/
```

### 2.3 Inicializar la capa de estado

Copiar al proyecto las plantillas de `core/state-templates/`:

- `state.json` — índice de bloques/tareas con su `status` (`pending` /
  `in_progress` / `complete` / `failed`).
- `progress.md` — notas de sesión, append-only.
- `dudas.md` — preguntas abiertas al cliente o al equipo.

### 2.4 Rellenar la configuración del proyecto

- `CLAUDE.md` — objetivo y equipo propietario del proyecto, stack
  (`stack-profile.yml`) y capas de datos elegidas (landing/staging/marts o
  bronze/silver/gold).
- `data-conventions.md` — naming, contratos de datos, invariantes de calidad.
- `docs/architecture/decisions.md` — primeros ADRs (capas y stack), para que
  el agente tenga contexto verificable sin preguntar.

### 2.5 Tener un plan del proyecto (hard_spec.md)

El planificador necesita un fichero equivalente a `hard_spec.md` con los
objetivos y criterios de cada bloque del proyecto. Sin él no hay tareas que
el planificador pueda identificar como activas o pendientes.

---

## 3. Comandos disponibles

| Comando | Qué hace |
|---|---|
| `/plugin marketplace add <url-del-repo>` + `/plugin install data-eng-harness-v1@data-eng-harness-v1-marketplace` | Registra el plugin del arnés (4 agentes + skill `eng-harness`) en Claude Code (ver §2.1). |
| `/eng-harness` (o escribir "usar el arnes de ingenieria") | Dispara la skill del ciclo: ejecuta el protocolo de sesión (D9) — re-entrada, ciclo de 4 agentes, cierre — para **una** tarea. |
| `/clear` | Limpia el contexto de la sesión tras el checkpoint humano de cierre, antes de arrancar la siguiente sesión. |

Cada invocación de `/eng-harness` procesa como máximo una tarea (regla "una
tarea por sesión", D9). El detalle del ciclo está en
`adapters/claude-code/skills/harness/SKILL.md` y `core/orchestration/cycle.md`.

---

## 4. Operación diaria

1. Arrancar sesión → escribir `/eng-harness`.
2. El orquestador ejecuta la secuencia de re-entrada: lee `state.json`,
   `progress.md` y el historial de control de versiones.
3. Ciclo de 4 agentes: planificador → implementador → evaluador →
   planificador.
4. Checkpoint humano: revisar veredicto, diff/artefactos y estado
   actualizado (`state.json` + `progress.md`).
5. `/clear` y repetir desde el paso 1 para la siguiente tarea.

El protocolo completo de sesión está definido en
`core/orchestration/session-protocol.md`.
