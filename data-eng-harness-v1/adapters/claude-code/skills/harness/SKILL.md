---
name: eng-harness
description: |
  Skill del workflow del arnés harness-v3 para ingeniería de datos.
  Ejecuta el protocolo de sesión único (D9): re-entrada → ciclo de 4 agentes
  (planificación → implementación → evaluación, arrancando SIEMPRE por el
  planificador) → cierre de sesión. Procesa UNA tarea por sesión.
  TRIGGER when: el usuario invoca /eng-harness o escribe "usar el arnes de
  ingenieria" o pide explícitamente ejecutar el ciclo del arnés.
---

# Harness Workflow — Skill del protocolo de sesión y ciclo de 4 agentes

Esta skill materializa el protocolo de sesión (D9) definido en
`core/orchestration/session-protocol.md` y el ciclo de 4 agentes definido en
`core/orchestration/cycle.md` como una secuencia ejecutable en Claude Code.
El orquestador (hilo principal de la sesión) ejecuta primero la secuencia de
re-entrada, luego coordina los 4 subagentes especializados en el orden
correcto, y cierra con la secuencia de cierre. Los agentes concretos están
definidos en `adapters/claude-code/agents/`.

> Regla "una tarea por sesión" (D9): esta skill ejecuta como máximo **una**
> vuelta completa del ciclo (un contrato `tasks/{B}-{slug}.json`) por
> invocación. No encadena varias tareas en la misma ventana de contexto.

```
[secuencia de re-entrada]
  │
  ├─► leer state.json → identificar tarea in_progress o siguiente pending
  ├─► leer última entrada de progress.md
  ├─► inspeccionar git log -N (baseline)
  │
  ▼
orquestador (esta sesión)
  │
  ├─► planificador      → decide/confirma la tarea, produce el contrato JSON
  │     └─► [si necesita investigación] navegador → brief → planificador
  │
  ├─► implementador     → produce los artefactos del contrato
  │
  ├─► evaluador         → verifica artefactos, emite APTO / NO APTO
  │
  └─► planificador      → registra veredicto en el JSON y en state.json
        ├─► si APTO    → status: complete
        └─► si NO APTO → reintentar implementador (máx. 2 veces) o failed
  │
  ▼
[secuencia de cierre]
  │
  ├─► commit (solo si APTO)
  ├─► actualizar state.json + progress.md (nueva entrada de sesión)
  ├─► checkpoint humano (veredicto + diff/artefactos + estado actualizado)
  └─► /clear → fin de sesión
```

## Workflow

### 0. Secuencia de re-entrada (arranque de la sesión)

Antes de invocar a ningún subagente, el orquestador reconstruye el contexto
sin depender de la memoria de sesiones anteriores (D9,
`core/orchestration/session-protocol.md`):

1. `Read("state.json")` — leer el índice de bloques/tareas y su `status`.
2. Identificar la tarea activa (`in_progress`) o, si no hay ninguna, la
   siguiente `pending`.
3. `Read("progress.md")` — leer la última entrada (qué se hizo, bugs
   conocidos, siguiente paso).
4. Inspeccionar el historial de control de versiones (p. ej. `git log -N`)
   para confirmar que el repositorio refleja lo que dicen `state.json` y
   `progress.md`.
5. Verificar el baseline: el repositorio está en un estado consistente, sin
   cambios sin confirmar de una sesión anterior que no cerró correctamente.
6. Invocar al planificador (paso 1 del workflow) con ese contexto
   reconstruido (tarea activa/siguiente, última entrada de `progress.md`,
   resultado de la verificación de baseline).

### 1. Arrancar SIEMPRE por el planificador

Invocar al planificador como primer paso obligatorio, sin excepción:

```
Agent(subagent_type: "planificador", prompt: "
  Lee state.json y progress.md (secuencia de re-entrada D9), y hard_spec.md.
  Identifica la tarea in_progress o, si no hay ninguna, la siguiente pending.
  Produce o actualiza el contrato JSON de handoff en tasks/. Actualiza
  state.json con status in_progress para esa tarea. Devuelve: ruta del
  contrato JSON, lista de acceptance_criteria y lista de artifacts.output.
")
```

El planificador puede spawnear al navegador por su cuenta si necesita
investigación. El orquestador espera a que el planificador devuelva el resumen
antes de continuar.

### 2. Invocar al implementador

Con la ruta del contrato JSON devuelta por el planificador:

```
Agent(subagent_type: "implementador", prompt: "
  Lee el contrato JSON en {ruta_contrato}. Produce todos los artefactos de
  artifacts.output. Verifica internamente cada acceptance_criterion. Lista
  todos los ficheros creados o modificados.
")
```

### 3. Invocar al evaluador

Con la ruta del contrato y la lista de artefactos producidos:

```
Agent(subagent_type: "evaluador", prompt: "
  Lee el contrato JSON en {ruta_contrato}. Verifica cada acceptance_criterion
  contra los artefactos producidos: {lista_artefactos}. Emite veredicto
  APTO o NO APTO con defectos concretos y rutas exactas.
")
```

### 4. Cerrar la tarea con el planificador

Pasar el veredicto al planificador para que actualice el estado:

```
Agent(subagent_type: "planificador", prompt: "
  El evaluador ha emitido veredicto {APTO|NO APTO} para la tarea {task_id}.
  Defectos: {lista_defectos_si_los_hay}.
  Actualiza el contrato JSON en {ruta_contrato} (status: complete/failed).
  Actualiza state.json con el campo de estado correspondiente (complete si
  APTO; in_progress si NO APTO con reintentos disponibles; failed si NO APTO
  en el segundo intento). Añade una entrada nueva al final de progress.md
  siguiendo el formato de core/state-templates/progress.md.
")
```

### 5. Secuencia de cierre de sesión (D9)

Tras el paso 4, el orquestador ejecuta la secuencia de cierre de
`core/orchestration/session-protocol.md`:

1. **Veredicto del evaluador** ya disponible (paso 3).
2. **Commit solo si `APTO`.** Si el veredicto es `NO APTO`, no se confirma
   ningún cambio en el repositorio; la tarea permanece `in_progress` para la
   siguiente sesión, con los defectos anotados en `progress.md` (paso 4).
3. **Estado actualizado**: `state.json` y `progress.md` ya actualizados por
   el planificador en el paso 4.
4. **Checkpoint humano**: presentar al usuario, como mínimo:
   - el veredicto del evaluador (APTO/NO APTO + defectos si los hay);
   - el diff/artefactos producidos en la sesión;
   - el estado actualizado (`state.json` + nueva entrada de `progress.md`).
5. **Limpieza de contexto**: tras el checkpoint, indicar al usuario que
   ejecute `/clear` para empezar la siguiente sesión (ver "Contrato de
   checkpoint" más abajo).

Si el veredicto fue `NO APTO` y quedan reintentos (máx. 2), el checkpoint
ofrece al usuario la opción de relanzar el implementador (paso 2) con los
defectos como contexto adicional **dentro de la misma sesión**, antes de
llegar al cierre. Si es el segundo intento fallido, se activa
`fallback.on_blocked: escalate_to_human` y se detiene hasta instrucción
explícita del usuario.

## Reglas del orquestador al ejecutar esta skill

- El ciclo arranca **siempre por el planificador**, nunca por el implementador
  ni el evaluador directamente.
- El orquestador no implementa ni evalúa: solo coordina y transfiere contexto.
- **Una tarea por sesión (D9):** esta skill ejecuta como máximo una vuelta del
  ciclo de 4 agentes (un contrato `tasks/{B}-{slug}.json`). No se encadenan
  varias tareas en la misma ventana de contexto.
- El estado persiste en `state.json` (índice de estado), `progress.md` (notas
  de sesión) y `tasks/*.json` (contratos); si la sesión se interrumpe, la
  siguiente sesión arranca con la secuencia de re-entrada (paso 0) leyendo
  esos ficheros para retomar.

## Contrato de checkpoint (cierre de sesión, D9)

Antes de que el usuario ejecute `/clear`, el orquestador debe haber mostrado:

- el **veredicto del evaluador** (APTO/NO APTO y defectos, si los hay);
- el **diff/artefactos producidos** en la sesión;
- el **estado actualizado** (`state.json` con el campo de estado correcto +
  la entrada nueva en `progress.md`).

Tras el checkpoint, el orquestador escribe al usuario:

> ✅ **Tarea {task_id} cerrada con veredicto {APTO|NO APTO}.** Haz `/clear` y
> ejecuta `/eng-harness` (o escribe "usar el arnes de ingenieria") para
> continuar con la siguiente tarea con contexto limpio.

No se inicia ninguna tarea adicional ni se lanza ningún agente más hasta que
el usuario lo pida explícitamente en una nueva sesión (regla "una tarea por
sesión", D9).

## File Locations

| Fichero | Propósito |
|---------|-----------|
| `state.json` | Capa de estado: índice de bloques/tareas y su `status` (`pending`/`in_progress`/`complete`/`failed`) |
| `progress.md` | Notas de sesión, append-only: qué se hizo, veredicto, bugs, siguiente paso |
| `hard_spec.md` | Plan completo: bloques, criterios, artefactos |
| `tasks/{B}-{slug}.json` | Contrato de handoff activo (producido por el planificador) |
| `core/orchestration/cycle.md` | Definición model-agnostic del ciclo de 4 agentes (fuente de verdad del patrón) |
| `core/orchestration/session-protocol.md` | Definición model-agnostic del protocolo de sesión (D9): re-entrada, ciclo, cierre, checkpoint |
| `core/state-templates/state.json` | Plantilla de `state.json` (D10) |
| `core/state-templates/progress.md` | Plantilla de `progress.md` y formato de entrada (D10) |
| `core/contracts/*.md` | Contratos de cada agente (entradas, salidas, criterios) |
| `adapters/claude-code/agents/*.md` | Definiciones ejecutables de los subagentes en Claude Code |
