# Protocolo de sesión (D9)

> Definición model-agnostic del protocolo de sesión único (hard_spec.md §3.1.1, D9). Complementa
> `cycle.md` (qué hace el ciclo de 4 agentes dentro de una sesión) y `stop-conditions.md` (cuándo
> el orquestador para o escala). Este documento define **cuándo empieza y cuándo termina una
> sesión**, y qué ocurre en esos dos límites. Incluye D13 (git como mecanismo de recuperación del
> bucle): commit por tarea verificada, política de reintentos con `git reset --hard` de rescate,
> `git log -N` como recuperación y tag opcional por bloque.

## Por qué un único protocolo (D9)

El arnés define **un único protocolo de sesión**, no dos modos distintos ("interactivo" vs
"autónomo"). La política de checkpoint —quién relanza la sesión y qué se le muestra— es un
**parámetro** de ese protocolo, no un diseño alternativo. D12 instancia este parámetro en dos
políticas concretas (ver "Política de checkpoint (D12)" más abajo):

- `validacion-por-tarea` (modo por defecto en v1): el **humano** es quien relanza la sesión y
  revisa el checkpoint completo (ver "Contrato del checkpoint" más abajo).
- `full-auto`: el bucle ejecuta la **misma secuencia**, pero encadena tareas sin checkpoint humano
  intermedio, salvo que se active un checkpoint duro.

Ambas políticas comparten la misma secuencia de re-entrada, el mismo ciclo de 4 agentes
(`cycle.md`) y la misma secuencia de cierre.

---

## Política de checkpoint (D12)

D12 instancia el parámetro de checkpoint de D9 en **dos políticas concretas**. No son dos
protocolos distintos: ambas comparten exactamente la secuencia de re-entrada, el ciclo de 4 agentes
y la secuencia de cierre descritas en este documento; solo difieren en **el paso 4 de la secuencia
de cierre** ("Checkpoint").

### `validacion-por-tarea` (modo por defecto en v1)

- Al final de cada tarea, el **humano** revisa el "Contrato del checkpoint" completo (veredicto del
  evaluador, diff/artefactos, estado actualizado) y cierra la sesión con la limpieza de contexto
  (p. ej. `/clear`) antes de que se arranque una sesión nueva.
- Es la aplicación directa de D9(a): el humano como mecanismo de checkpoint y como fuente del
  *tuning loop* (P13).
- Es el modo por defecto: salvo que el proyecto declare explícitamente la política `full-auto`, el
  orquestador asume `validacion-por-tarea`.

### `full-auto`

- El bucle encadena tareas sucesivas **sin checkpoint humano intermedio** entre ellas: tras el
  cierre de una tarea (commit si `APTO`, actualización de `state.json`/`progress.md`), el
  orquestador continúa directamente con la secuencia de re-entrada de la siguiente tarea, sin
  esperar revisión humana.
- **Implicación — mecanismo de continuación.** `full-auto` requiere uno de estos dos mecanismos:
  1. **Relanzamiento de sesiones**: un proceso externo reinicia la sesión tras cada cierre. Esto
     adelanta parcialmente el runner autónomo evolutivo declarado en D9(c).
  2. **Continuación dentro de la misma sesión**, mientras el contexto lo permita (sujeto a la
     compaction de `state-templates/handoff-protocol.md` §4.2).
  > La elección entre estos dos enfoques **se decide al implementar el Bloque 2** (no se decide en
  > esta enmienda; ver hard_spec.md D12(2)(i) y R7).
- **Implicación — checkpoints duros.** `full-auto` sigue sujeto, sin excepción, a los
  **checkpoints duros** definidos más abajo. Ningún encadenamiento automático puede saltarse un
  checkpoint duro.

> Ambas políticas son un parámetro de **este mismo protocolo de sesión** (D9): la secuencia de
> re-entrada, el ciclo de 4 agentes y la secuencia de cierre no cambian entre políticas.

---

## Checkpoints duros (invariante del protocolo, D12)

Existen dos tipos de operación que requieren **aprobación humana explícita en cualquier política**,
incluida `full-auto`:

1. **Operaciones DDL/DML destructivas, o que actúan contra sistemas de un cliente** (p. ej. `DROP`,
   `TRUNCATE`, `DELETE` sin `WHERE` acotado, escritura sobre entornos productivos o de cliente).
2. **Cualquier modificación de `hard_spec.md`** (la memoria casi inmutable del proyecto, ver D10).

> Los checkpoints duros son un invariante del **protocolo de sesión** (D9, §3.1.1), no de la
> política de checkpoint elegida. Ninguna política —ni siquiera `full-auto`— puede degradarlos,
> automatizarlos ni muestrearlos. Si una tarea implica una operación de este tipo, el orquestador
> **detiene el bucle automático** (incluso en `full-auto`) y escala al humano antes de continuar,
> siguiendo la "Parada por escalada" de `stop-conditions.md`.

---

## Regla "una tarea por sesión"

Cada sesión ejecuta **como máximo una tarea completa** del backlog: un ciclo
planificador → implementador → evaluador → planificador, y termina en la secuencia de cierre
descrita abajo.

- El bucle **no se repite dentro de la misma sesión**: no se encadenan varias tareas en la misma
  ventana de contexto sin pasar por la secuencia de cierre.
- Quien decide si arrancar una nueva sesión (y por tanto procesar la siguiente tarea) depende de la
  política de checkpoint activa (D12): bajo `validacion-por-tarea`, el **humano**, tras revisar el
  checkpoint de cierre; bajo `full-auto`, el **mecanismo de continuación** (ver "Política de
  checkpoint (D12)"), salvo que se active un checkpoint duro, en cuyo caso decide el humano.
- Esto es consistente con P4 (una tarea por iteración, Huntley) y con la economía de contexto de P2:
  una sesión amnésica que reconstruye su contexto desde ficheros de estado cortos es más fiable que
  una sesión larga que acumula historia.

---

## Secuencia de re-entrada (arranque de cada sesión fresca)

Al inicio de cada sesión, el orquestador reconstruye el contexto necesario sin depender de la
memoria de la sesión anterior:

0. **Bootstrap de `project-template/` y artefactos de estado (D14, solo si no se ha hecho antes).**
   Antes de leer `state.json` (paso 1), el adaptador comprueba si `project-template/` (o su
   equivalente ya renombrado/adaptado por el proyecto) y los artefactos de estado de
   `core/state-templates/` (`state.json`, `progress.md`) ya están materializados en el repositorio
   del proyecto del usuario. Si faltan, el adaptador los copia desde la instalación del arnés
   resolviendo la ruta raíz del plugin en el entorno del agente —bajo la cual cuelgan directamente
   `project-template/` y `core/`— y copiándolos desde ahí; si la detección o la copia fallan, cae al
   `cp -r` manual documentado como fallback en `SETUP.md` §2.2. Ver D14
   (`hard_spec.md` §5) para el mecanismo completo. Este paso es **idempotente** y **no
   sobrescribe**: si la detección encuentra los artefactos ya presentes (con cualquier contenido),
   no hace nada y la secuencia continúa directamente en el paso 1.
1. **Leer `state.json`** — índice de bloques/tareas y su campo de estado actual.
2. **Identificar la tarea activa** (`in_progress`) o, si no hay ninguna, **la siguiente `pending`**.
3. **Leer la última entrada de `progress.md`** — qué se hizo en la sesión anterior, bugs conocidos,
   siguiente paso anotado.
4. **Inspeccionar el historial de control de versiones** (p. ej. `git log -N` con las últimas N
   entradas). Esta inspección cumple dos roles (D13(c)):
   - **Confirmación**: verificar que el repositorio refleja lo que dicen `state.json`/`progress.md`.
   - **Recuperación**: si la sesión anterior quedó interrumpida tras un veredicto `NO APTO`, el
     **último commit verificado** (la última "tarea verificada = un commit" de D13(a)) es el punto
     real desde el que reconstruir el estado, con independencia de lo que digan los artefactos sin
     confirmar que puedan quedar en el working tree.
5. **Verificar el baseline**: el repositorio está en un estado consistente, sin cambios sin
   confirmar de una sesión anterior que no cerró correctamente. Si los hay y proceden de una tarea
   `in_progress` con un veredicto `NO APTO` previo, son el punto de partida del reintento (ver
   "Política de reintentos y recuperación con git (D13)" más abajo).
6. **Invocar al `planificador`** con ese contexto reconstruido (tarea activa/siguiente, última
   entrada de progreso, resultado de la verificación de baseline).

> Nota model-agnostic: "inspeccionar el historial de control de versiones" es un ejemplo de la
> operación (comprobar que el repo coincide con lo que dice el estado persistido, y recuperar el
> último estado verificado si hace falta), no una llamada obligatoria a `git` específicamente;
> cualquier sistema de control de versiones del proyecto sirve.

> Nota model-agnostic (paso 0, D14): "la ruta raíz del plugin" (`${CLAUDE_PLUGIN_ROOT}` en Claude
> Code) y "el entorno del agente" (la herramienta de shell disponible, p. ej. `Bash`) son ejemplos
> concretos del adaptador Claude Code. El mecanismo —detectar si los artefactos de plantilla ya
> están materializados, copiarlos condicionalmente desde la instalación del arnés si faltan, y caer
> a una copia manual documentada si la detección o la copia fallan— no depende de una API
> específica de un proveedor; la dependencia de runtime concreta vive en `adapters/`.

---

## Ciclo de 4 agentes

Una vez completada la re-entrada, el orquestador ejecuta el ciclo de 4 agentes descrito en
`cycle.md` para la tarea identificada en el paso 2.

---

## Secuencia de cierre (fin de cada sesión)

1. **El `evaluador` emite veredicto** (`APTO` / `NO APTO` + defectos).
2. **Commit solo tras `APTO`, y antes del checkpoint humano (D13(a)).** Una tarea verificada es un
   commit: el commit consolida el estado que la siguiente sesión fresca reconstruirá en su
   secuencia de re-entrada (paso 4 de arriba), y es precisamente lo que hace seguro ejecutar la
   limpieza de contexto del paso 5 — el contexto se descarta, pero el estado verificado vive en
   git. El orden es siempre **commit (tras `APTO`) → checkpoint → limpieza de contexto**, nunca al
   revés.
   - Si el veredicto es `NO APTO`, no se confirma ningún cambio en el repositorio: los artefactos
     producidos quedan en el working tree y la tarea permanece `in_progress` para la siguiente
     sesión, que reintenta a partir de ellos, con los defectos anotados en el contrato de tarea y
     en `progress.md` (ver "Política de reintentos y recuperación con git (D13)" más abajo).
3. **Actualización de estado**: el `planificador` actualiza el campo de estado de la tarea (y, si
   corresponde, del bloque) en `state.json`, y añade una entrada nueva al final de `progress.md`
   (ver "Cómo añadir una entrada" en `state-templates/progress.md`).
4. **Checkpoint** (ver "Contrato del checkpoint" más abajo): qué ocurre en este paso depende de la
   **política de checkpoint** activa para el proyecto (D12, ver "Política de checkpoint (D12)" más
   arriba) — `validacion-por-tarea` (revisión humana completa, modo por defecto) o `full-auto`
   (encadenamiento sin checkpoint intermedio, salvo checkpoint duro). El checkpoint ocurre
   **siempre después** del commit del paso 2: el humano (o el mecanismo de continuación de
   `full-auto`) revisa un estado ya consolidado en git.
5. **Limpieza de contexto**: en `validacion-por-tarea`, el humano limpia el contexto de la sesión y,
   si decide continuar, arranca una sesión fresca que repite la secuencia de re-entrada. En
   `full-auto`, esta limpieza la realiza el mecanismo de continuación (ver "Política de checkpoint
   (D12)").

> Nota model-agnostic: "limpieza de contexto" (p. ej. el comando `/clear` en Claude Code) es un
> ejemplo de la operación —descartar la ventana de contexto de la sesión actual—, no una API
> exclusiva de un proveedor. La dependencia de runtime concreta vive en `adapters/`.

---

## Política de reintentos y recuperación con git (D13)

Esta sección completa la secuencia de cierre (paso 2) para el caso `NO APTO`, definiendo cómo se
recupera el bucle si se rompe.

- **Reintento desde el working tree.** Si el veredicto es `NO APTO`, los artefactos producidos por
  el `implementador` **no se descartan**: permanecen en el working tree y la siguiente sesión
  reintenta la misma tarea partiendo de ellos, con los defectos del evaluador anotados en el
  contrato de tarea (`tasks/{bloque}-{slug}.json`) y en `progress.md`.
- **Límite de 2 reintentos por tarea.** Una tarea puede recibir como máximo 2 veredictos `NO APTO`
  consecutivos antes de agotar sus reintentos (ver también "Parada por escalada" en
  `stop-conditions.md`).
- **Rescate con `git reset --hard` al agotar los reintentos.** Si tras 2 reintentos la tarea sigue
  sin obtener `APTO`, el orquestador ejecuta `git reset --hard` al **último commit verificado**
  (el commit de la última tarea que sí obtuvo `APTO`, paso 2 de arriba), descartando los artefactos
  sin confirmar de los reintentos fallidos. La tarea vuelve al `planificador` con los defectos
  anotados en el contrato; si el planificador no puede reformularla, se escala al humano (ver
  "Parada por escalada" en `stop-conditions.md`).
- **Por qué elimina el deadlock.** Sin este mecanismo, una tarea que nunca llega a `APTO` dejaría el
  bucle reintentando indefinidamente sobre un working tree cada vez más degradado. El `git reset
  --hard` garantiza que el peor caso siempre vuelve a un estado conocido y verificado, desde el que
  el planificador puede reformular la tarea o el humano puede decidir.

> Nota model-agnostic: `git reset --hard` es el ejemplo concreto para Git; cualquier sistema de
> control de versiones del proyecto que soporte "volver al último estado confirmado descartando
> cambios sin confirmar" cumple el mismo rol.

---

## Tag opcional por bloque (checkpoint de rescate, D13(d))

Al completarse un bloque del backlog (todas sus tareas en `complete`), el orquestador **puede**
crear un `tag` de git (p. ej. `bloqueN-complete`) como checkpoint de rescate adicional.

- Es **opcional**, no obligatorio: no sustituye al commit por tarea verificada (D13(a)), que es la
  unidad mínima de recuperación.
- Su utilidad es ofrecer un punto de referencia adicional, más grueso que el commit por tarea, al
  que volver si se necesita rescatar el estado de un bloque completo (p. ej. antes de empezar un
  bloque experimental o de alto riesgo).

---

## Contrato del checkpoint

Bajo la política `validacion-por-tarea` (modo por defecto), antes de cerrar la sesión (paso 5) el
humano revisa **como mínimo**:

- el **veredicto del evaluador** (`APTO`/`NO APTO` y defectos, si los hay);
- el **diff/artefactos producidos** en la sesión;
- el **estado actualizado** (`state.json` con el campo de estado correcto + la entrada nueva en
  `progress.md`).

> Esta lista **es el contrato del checkpoint**: es lo que `validacion-por-tarea` revisa
> íntegramente en cada cierre de sesión, y lo que `full-auto` (D12) encadena sin revisión humana
> intermedia entre tareas — salvo que la tarea active un **checkpoint duro** (ver "Checkpoints
> duros (invariante del protocolo, D12)" más arriba), en cuyo caso la revisión humana de este
> contrato vuelve a ser obligatoria independientemente de la política activa.

---

## Diferencia entre re-entrada y este protocolo de cierre vs. compaction

Si el contexto se satura **dentro de una sesión activa** (sin haber llegado al cierre de la tarea),
el orquestador puede compactar el contexto sin terminar la sesión: ver "Compaction" en
`state-templates/handoff-protocol.md` §4.2. La compaction no sustituye a la secuencia de cierre; es
un mecanismo para que la sesión actual sobreviva a la presión de contexto sin perder estado
funcional.
