Definiciones de los 4 subagentes (planificador, navegador, implementador, evaluador) para Claude Code.

## Patrón de cada agente (D-09)

Cada agente referencia su contrato model-agnostic en `../../../core/contracts/{agente}.md` como
fuente de verdad de rol, entradas, salidas, criterio de "done", criterio de parada y
restricciones. El "Paso 0" del Procedimiento de cada agente obliga a leer ese contrato en
runtime con `Read` antes de actuar. Este fichero (el adaptador) solo añade la capa de ejecución
Claude Code: frontmatter (`name`/`description`/`tools`), llamadas concretas a herramientas en
"Procedimiento" y el "Formato de respuesta". La sección "Reglas" de cada agente conserva
únicamente lo genuinamente vendor-specific o de convención (no reformula las restricciones del
contrato del core).

> **Excepción — campo `description` del frontmatter.** A diferencia del resto del fichero, el
> campo `description` de cada agente NO se reduce a un puntero al core: Claude Code lo usa para
> enrutar el `Task`/`Agent` tool (decidir qué subagente spawnear), por lo que debe seguir siendo
> autocontenido y describir qué es el agente y cuándo usarlo. Es la única parte que
> legítimamente resume el rol, por necesidad de enrutamiento del runtime — no por duplicación.
