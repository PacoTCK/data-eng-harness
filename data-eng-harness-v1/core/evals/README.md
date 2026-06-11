# core/evals — Baseline de evals del arnés

> Este directorio contiene el baseline de evaluación del arnés v3. Los evals comprueban que el arnés
> (el conjunto de agentes, contratos y sensores) produce el comportamiento esperado — no que el modelo
> LLM base sea capaz o no de completar una tarea genérica.

---

## Qué son los evals del arnés (vs evals del LLM base)

| Aspecto | Evals del LLM base | Evals del arnés |
|---|---|---|
| Qué miden | Capacidades del modelo (razonamiento, código, idioma) | Si el arnés dirige correctamente al agente dentro de su contexto de trabajo |
| Quién varía | El modelo o el fine-tuning | La config del arnés, los prompts de los agentes, los contratos |
| Cuándo fallan | El modelo no sabe hacer algo | El arnés no comunica bien la tarea, no activa los sensores correctos, el agente declara victoria prematura |
| Cómo se mejoran | Entrenamiento o cambio de modelo | Ajuste de prompts de agentes, adición de checks al contrato, nuevos sensores |

Los evals del arnés son la herramienta para iterar sobre el arnés mismo: permiten comparar variantes
de config/prompts y detectar regresiones al cambiar cualquier elemento del sistema.

---

## Estructura prompt → run → checks → score

Cada eval sigue este patrón invariable:

```
prompt representativo
    └─► run capturado (log JSON: tool calls + respuestas del agente)
            └─► checks
            │     ├─► deterministas  →  PASS / FAIL  (sin ambigüedad)
            │     └─► rubric-based   →  score 1-5 según rúbrica explícita
            └─► score
                  └─► 4 ejes ponderados → scorecard
```

### Paso 1 — Prompt representativo

Tarea concreta de ingeniería de datos que el arnés orquestaría. Debe ser específica y reproducible
(parámetros fijos, dataset de referencia, criterio de completitud definido).

### Paso 2 — Run capturado

Transcripción del ciclo del agente guardada como JSON (`run_capture`). Contiene:
- las tool calls realizadas y sus argumentos
- las respuestas del agente en cada turno
- los sensores activados y su resultado

En ausencia de ejecución real, la estructura del run se describe como especificación. Ver R4.

### Paso 3 — Checks

Dos tipos, que deben mantenerse separados:

**Checks deterministas** — PASS/FAIL verificable mecánicamente:
- El sensor FF-01 fue invocado antes de continuar
- El artefacto de salida tiene el schema declarado
- Los tests del pipeline pasaron
- No hay columnas con `SELECT *` en el modelo generado

**Checks rubric-based** — criterio subjetivo convertido en rúbrica explícita (escala 1-5):
- Calidad del modelo dimensional generado
- Seguimiento del protocolo del arnés (usó contratos JSON, actualizó `state.json`/`progress.md`)
- Documentación del modelo (descriptions en el YAML de dbt)

### Paso 4 — Score (4 ejes)

| Eje | Qué mide | Peso por defecto |
|---|---|---|
| `outcome` | El artefacto cumple la especificación técnica | 0.40 |
| `process` | El agente siguió el protocolo del arnés | 0.30 |
| `style` | Convenciones de código/SQL/YAML, naming, docs | 0.20 |
| `efficiency` | Tokens consumidos, iteraciones hasta APTO | 0.10 |

El score de cada eje se calcula según el método declarado en el propio eval (fracción de checks
deterministas superados, media de rubric scores / 5, etc.) y se consolida en un scorecard.

**Umbral de aprobación por defecto: 0.70.**

---

## Contenido del baseline

```
core/evals/
  README.md                          este fichero — índice y guía
  scorecard-schema.json              esquema JSON reutilizable para registrar scorecards
  tuning-loop.md                     protocolo para convertir fallos en nuevos evals o constraints
  baseline/
    eval-01-schema-validation.yaml   eval: agente valida schema de un dataset de entrada
    eval-02-pipeline-generation.yaml eval: agente genera un modelo de transformación
    eval-03-harness-protocol.yaml    eval: agente sigue el protocolo del arnés
```

Los tres evals del baseline cubren las tres áreas de riesgo principales del arnés:
- **eval-01** — calidad de los sensores de datos (¿activa FF-01 correctamente?)
- **eval-02** — calidad del código generado (¿produce un modelo correcto y bien documentado?)
- **eval-03** — seguimiento del protocolo (¿usa los contratos, actualiza `state.json`/`progress.md`, no se salta pasos?)

---

## Cómo ejecutar conceptualmente

> **Nota R4:** este baseline es especificación + ejemplos conceptuales. No hay proyecto de datos real.
> Los pasos siguientes describen la ejecución esperada cuando el arnés se use en un proyecto real.

1. Seleccionar el eval a ejecutar (p.ej. `eval-01-schema-validation.yaml`).
2. Iniciar una sesión del arnés con el prompt del campo `prompt` del eval.
3. Capturar el log completo del ciclo (tool calls + respuestas) como JSON en `run_capture`.
4. Ejecutar los checks deterministas contra el log capturado.
5. Puntuar los checks rubric-based según la rúbrica del eval.
6. Calcular el scorecard con `scorecard-schema.json`.
7. Registrar el resultado en `baseline/runs/` (carpeta a crear en ejecución real).
8. Si algún check falla, seguir el protocolo de `tuning-loop.md`.

---

## Cómo añadir un eval nuevo

1. Copiar la plantilla más cercana de los tres evals existentes.
2. Asignar el `id` siguiente en secuencia (`eval-04`, etc.).
3. Definir el `prompt` con una tarea específica y reproducible de ingeniería de datos.
4. Declarar al menos 2 checks deterministas y 1 rubric-based.
5. Completar los pesos de los 4 ejes (deben sumar 1.0).
6. Establecer el `pass_threshold` (recomendado 0.70 por defecto; justificar si se baja).
7. Marcar `status: conceptual` hasta que haya un run real que valide el eval.

---

## Comparación de variantes de config/prompts

Para comparar dos variantes A y B del arnés sobre el mismo eval:

1. Ejecutar el eval con la variante A; registrar el scorecard con `variant: "A"`.
2. Ejecutar el mismo eval con la variante B; registrar el scorecard con `variant: "B"`.
3. Comparar los cuatro ejes por separado — una variante puede mejorar `outcome` y empeorar `efficiency`.
4. Elegir la variante que maximice el eje más importante para el contexto del equipo.
5. Si la variante ganadora introduce una regresión en otro eje, abrir una entrada en `docs/dudas.md`.
