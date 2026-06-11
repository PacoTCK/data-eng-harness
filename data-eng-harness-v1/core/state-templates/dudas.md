# Backlog de dudas — {PROYECTO}

> Registro de gaps, preguntas abiertas y decisiones que requieren confirmación externa (cliente, equipo, humano).
> **Append-only:** nunca se borran entradas. Las dudas resueltas se marcan con estado `resuelta` y se rellena la columna `Resolución`.
> Las dudas que bloquean la redacción o implementación de un artefacto deben citarse en ese artefacto como `⟨pendiente de confirmar — ver duda #{ID}⟩`.

---

## Instrucciones de uso

1. **Al detectar un gap** — cualquier agente puede añadir una entrada al final de la tabla. El ID sigue el patrón `D{numero_secuencial}` (D1, D2, …).
2. **Al referenciar una duda en un artefacto** — usar la notación `⟨pendiente de confirmar — ver duda #{ID}⟩` en el lugar del artefacto donde falta la información.
3. **Al resolver una duda** — añadir la fecha de resolución, cambiar el estado a `resuelta` y escribir la resolución en la columna correspondiente. No eliminar la fila.
4. **Quién puede resolver** — solo quien tenga autoridad para confirmar el dato (cliente, lead del proyecto, humano que aprobó el bloque). Los agentes no marcan dudas como resueltas por su cuenta.

---

## Tabla de dudas

| ID | Fecha apertura | Bloque | Pregunta / Gap                                                                 | Responsable de resolver          | Estado    | Fecha resolución | Resolución                                      |
|----|----------------|--------|--------------------------------------------------------------------------------|----------------------------------|-----------|------------------|-------------------------------------------------|
| D1 | {YYYY-MM-DD}   | {B?}   | {Pregunta concreta en una frase. Qué información falta o qué hay que confirmar.} | {cliente / lead / equipo / humano} | abierta | —                | —                                               |

> Nota: copiar la fila de ejemplo y rellenar. El ID es secuencial y nunca se reutiliza.

---

## Formato extendido (para dudas complejas)

Para dudas que bloquean múltiples artefactos o que requieren contexto adicional, añadir una sección debajo de la tabla con este formato:

### {ID} — {pregunta en una frase}

**Contexto:** dónde surgió la duda y qué desencadenó la pregunta.

**Bloquea:** qué artefactos o secciones no se pueden completar hasta resolver esta duda.

**Pendiente de:** quién puede responderla.

**Opciones posibles** (si aplica):
- Opción A: {descripción} — consecuencias.
- Opción B: {descripción} — consecuencias.

**Resolución** (rellenar al cerrar):
> {YYYY-MM-DD} — {descripción de la decisión tomada y su justificación.}
