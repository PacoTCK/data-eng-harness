# soft_spec.md — [Nombre del proyecto] ⟨pendiente de confirmar⟩

> Documento de entrada del proyecto, en lenguaje natural. Lo escribe el **humano**.
> El planificador del arnés lo lee y deriva de él el `hard_spec.md` (plan curado: bloques,
> criterios de aceptación y decisiones de diseño) en la primera sesión del ciclo (D18).
>
> Reglas de redacción: español; una frase por afirmación clave; fechas absolutas (`YYYY-MM-DD`);
> lo no confirmado se marca `⟨pendiente de confirmar⟩` — no inventes cifras, SLAs ni compromisos.
> Esto es el *qué* y el *por qué* en prosa; el *cómo* estructurado lo produce el planificador.

---

## 1. Objetivo

[RELLENAR: 1-3 frases que describan qué se quiere construir y qué problema de negocio resuelve.]

## 2. Contexto y equipo

[RELLENAR: equipo propietario, contexto de The Cocktail / cliente, sistemas existentes con los que
convive el proyecto.]

## 3. Fuentes de datos

[RELLENAR: de dónde vienen los datos (sistemas origen, ficheros, APIs), su formato y su cadencia.
Marca `⟨pendiente de confirmar⟩` lo que no esté cerrado.]

## 4. Entregables

[RELLENAR: qué produce el proyecto (datasets, modelos, pipelines, dashboards, informes) y para quién.]

## 5. Restricciones y stack

[RELLENAR: stack de datos previsto (o "ver `stack-profile.yml`"), restricciones técnicas, de
seguridad/gobernanza, de coste o de plazos. Capas de datos elegidas si ya se sabe
(landing/staging/marts o bronze/silver/gold).]

## 6. Criterios de éxito de alto nivel

[RELLENAR: cómo se sabrá que el proyecto cumple — en términos de negocio y de calidad de datos.
El planificador los traducirá a `acceptance_criteria` verificables por bloque en el `hard_spec.md`.]

## 7. Presupuesto y gobernanza (opcional)

[RELLENAR si aplica: orden de magnitud de recursos por bloque (tokens/tiempo) que el planificador
usará como base para el bloque `governance.R`/`T` de cada contrato y el presupuesto `R` por bloque
en `state.json` (D17). Si se deja vacío, el planificador propone presupuestos por defecto.]

---

> Siguiente paso: ejecutar la skill del arnés (`/eng-harness` o "usar el arnés de ingeniería").
> En la primera sesión, el planificador derivará el `hard_spec.md` del proyecto a partir de este
> documento. A partir de ahí, este `soft_spec.md` queda como referencia de la intención original.
