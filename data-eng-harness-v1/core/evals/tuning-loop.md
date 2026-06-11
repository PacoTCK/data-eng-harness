# Tuning loop — Cómo convertir un fallo observado en un nuevo eval o constraint

> El tuning loop es el mecanismo por el que el arnés aprende de sus propios fallos.
> Cada fallo observado durante la ejecución del arnés es una oportunidad para añadir
> un check, un sensor o una restricción que impida que ese fallo se repita.
>
> Regla fundamental: **Never remove or edit existing eval criteria — only add new ones.**
> Esto garantiza que el historial de evals sea comparable entre versiones y que las
> regresiones sean detectables.

---

## Mecanismo 1 — Inmediato: fallo → nuevo check en el eval YAML

**Cuándo usar este mecanismo:** el fallo es específico de un eval concreto o es la primera vez que
se observa. Se codifica directamente en el YAML del eval como un nuevo check.

**Protocolo:**

1. **Observar** — identificar el fallo en el run capturado (campo `failed_checks` del scorecard).
2. **Identificar la divergencia** — ¿qué hizo el agente que no debería haber hecho, o qué dejó de
   hacer? Formular en una frase.
3. **Codificar la corrección** — decidir si el nuevo check es determinista o rubric-based:
   - Si la comprobación es mecánica y sin ambigüedad: check **determinista** (`det-N`).
   - Si requiere juicio pero puede expresarse como rúbrica 1-5: check **rubric-based** (`rub-N`).
4. **Añadir el check al YAML** del eval correspondiente (nunca editar checks existentes, solo añadir).
5. **Reajustar los pesos** si el nuevo check pertenece a un eje que cambia de peso. Los pesos de los
   4 ejes deben seguir sumando 1.0.
6. **Repetir** — en la siguiente ejecución del eval verificar que el nuevo check pasa.

### Ejemplo: antes/después

**Fallo observado (run de eval-02):**
> El agente generó un modelo dbt con `SELECT *` en la capa staging, violando la convención de
> naming y legibilidad. El eval no tenía ningún check que detectase esto.

**Antes — eval-02 (fragmento):**
```yaml
checks:
  deterministic:
    - id: det-01
      description: "El modelo dbt generado compila sin errores"
      verification: "dbt compile devuelve exit code 0"
      expected: pass
    - id: det-02
      description: "El modelo tiene al menos un test declarado en schema.yml"
      verification: "schema.yml contiene al menos 1 entrada bajo tests:"
      expected: pass
```

**Después — eval-02 (check añadido, los anteriores intactos):**
```yaml
checks:
  deterministic:
    - id: det-01
      description: "El modelo dbt generado compila sin errores"
      verification: "dbt compile devuelve exit code 0"
      expected: pass
    - id: det-02
      description: "El modelo tiene al menos un test declarado en schema.yml"
      verification: "schema.yml contiene al menos 1 entrada bajo tests:"
      expected: pass
    - id: det-03
      description: "El modelo no contiene SELECT * en ninguna capa"
      verification: "grep -rn 'SELECT \\*' en los ficheros .sql generados devuelve 0 matches"
      expected: pass
      added_after_failure: "run-eval-02-20260609 — agente usó SELECT * en staging"
```

> Nota: el campo `added_after_failure` es opcional pero recomendado para trazabilidad.

---

## Mecanismo 2 — Diferido: fallo recurrente → sensor determinista en `core/sensors/`

**Cuándo usar este mecanismo:** el mismo fallo aparece en dos o más evals distintos, o en dos
ejecuciones consecutivas del mismo eval después de haber añadido un check. El fallo es sistémico,
no puntual, y necesita ser detectado en tiempo de ejecución (in-session o CI), no solo en la fase
de eval post-hoc.

**Protocolo:**

1. **Detectar recurrencia** — el fallo ya tiene un check en algún eval y sigue ocurriendo, o aparece
   en dos evals distintos con la misma causa raíz.
2. **Identificar el invariante** — formular la regla que el agente siempre debe cumplir, en términos
   verificables mecánicamente.
3. **Crear el sensor** en `core/sensors/fast-feedback/` (si es in-session o CI) o en
   `core/sensors/drift-periodic/` (si es periódico). Seguir el formato de `core/sensors/catalog.md`.
4. **Referenciar el sensor desde el eval** — añadir en el check determinista del eval que el sensor
   fue activado y devolvió el resultado esperado.
5. **Reajustar** — en la siguiente ejecución del eval, el check ahora verifica la salida del sensor,
   no solo el artefacto.

### Ejemplo: antes/después

**Fallo recurrente detectado:**
> El agente avanzó a la transformación sin invocar el sensor de validación de schema (FF-01).
> Ocurrió en eval-01 (run-20260609) y de nuevo en eval-02 (run-20260615) con un dataset distinto.

**Antes — eval-01 check det-01:**
```yaml
- id: det-01
  description: "El agente ejecutó FF-01 (schema-contract) antes de continuar"
  verification: "run_capture contiene llamada a schema-contract sensor"
  expected: pass
```

**Después — sensor añadido en `core/sensors/fast-feedback/schema-contract.md` (entrada nueva):**
```markdown
### Invariante SEN-FF01-GATE
El agente NO puede avanzar de ingestión a transformación sin haber ejecutado FF-01 y recibido
un resultado SENSOR OK. Si FF-01 devuelve SENSOR FAIL, el agente debe reportar el error y detener.
Esta regla se inyecta en el CLAUDE.md del proyecto como restricción explícita.
```

**Después — eval-01 check det-01 (actualizado para referenciar el sensor; det-01 anterior intacto,
nueva entrada det-04 añadida):**
```yaml
- id: det-04
  description: "El sensor SEN-FF01-GATE impidió el avance cuando FF-01 devolvió FAIL"
  verification: "Si run_capture contiene FF-01 = FAIL, no debe haber artefactos de transformación"
  expected: pass
  added_after_failure: "recurrente en run-20260609 y run-20260615 — sensor SEN-FF01-GATE añadido"
```

---

## Reglas de versioning

1. **Never remove or edit existing eval criteria — only add new ones.** Editar un check existente
   equivale a cambiar el criterio de medición retroactivamente; esto hace incomparables los scorecards
   anteriores.
2. **Nunca reutilizar un `id` de check.** Si un check queda obsoleto, marcarlo con
   `deprecated: true` y añadir una nota; no eliminarlo.
3. **Versionado del eval:** cuando se añaden checks suficientes para constituir un cambio mayor,
   crear una nueva versión del eval (`eval-01-v2.yaml`) y mantener la original. Los scorecards
   históricos siguen referenciando la versión con la que se ejecutaron.
4. **Los pesos de los 4 ejes** pueden ajustarse entre versiones, pero el cambio debe documentarse
   en las `notes` del primer scorecard que use los nuevos pesos.

---

## Cuándo escalar al humano

Si después de 2 actualizaciones del mecanismo correspondiente el fallo persiste, el ciclo de tuning
automático se detiene y se escala al humano con el siguiente formato:

```markdown
## YYYY-MM-DD — <descripción del fallo persistente>
**Eval afectado:** eval-XX
**Check fallido:** det-NN / rub-NN
**Historial de correcciones:**
  1. YYYY-MM-DD — añadido check det-NN (mecanismo inmediato)
  2. YYYY-MM-DD — añadido sensor SEN-XXXX (mecanismo diferido)
**Fallo sigue ocurriendo:** sí
**Hipótesis:** <qué podría estar causando el fallo que no es un problema del arnés sino del modelo o del prompt de contexto>
**Acción requerida del humano:** <qué decisión necesita tomar el humano>
```

Esta entrada va al fichero `docs/dudas.md` **del proyecto instanciado**
(`project-template/docs/dudas.md` cuando se trabaja sobre la plantilla; en un proyecto ya
desplegado, `docs/dudas.md` en la raíz de ese proyecto) con la etiqueta `[TUNING-ESCALADO]`.
No confundir con `core/state-templates/dudas.md`, que es la plantilla genérica del core y no
recibe escalados de tuning directamente.

---

## Resumen del ciclo

```
fallo observado en run
    │
    ├─► primera vez o fallo puntual
    │       └─► Mecanismo 1: añadir check al eval YAML
    │
    └─► fallo recurrente (≥2 ocurrencias)
            └─► Mecanismo 2: añadir sensor en core/sensors/
                    │
                    └─► si persiste tras 2 actualizaciones
                            └─► escalar al humano (docs/dudas.md del proyecto instanciado —
                                ver "Cuándo escalar al humano")
```
