# Capas de datos del proyecto

> Propósito: documentar la arquitectura por capas elegida, sus invariantes y las
> responsabilidades de cada capa.
> Coherente con `../../data-conventions.md` y los sensores de `../../../core/sensors/`.

---

## Cómo rellenar este fichero

1. Elige la familia de capas en `../CLAUDE.md` (decisión independiente del perfil de stack de D11, ver `../dudas.md` duda D2).
2. Elimina la opción descartada de la sección siguiente y desarrolla la elegida.
3. Documenta los invariantes concretos del proyecto (qué puede y qué no puede entrar en cada capa).
4. Si añades capas intermedias (p. ej. una capa "raw" previa a landing), añádela aquí con su descripción.

---

## Opción A — landing / staging / marts

⟨pendiente de confirmar — ver duda #D2⟩

| Capa | Responsabilidad | Qué entra | Qué no entra |
|---|---|---|---|
| **landing** | Datos en bruto tal como llegan de la fuente | Ficheros, tablas de staging externas, raw API responses | Transformaciones, joins, lógica de negocio |
| **staging** | Limpieza y estandarización; 1 modelo por fuente | Renombrado de columnas, cast de tipos, deduplicación básica | Joins entre fuentes, métricas derivadas |
| **marts** | Tablas expuestas al negocio; modelo estrella o plano | Agregaciones, joins entre dominios, cálculos de KPIs | Lógica de extracción o limpieza de fuente |

[RELLENAR: invariantes adicionales del proyecto — qué garantías se mantienen en cada capa.]

---

## Opción B — bronze / silver / gold (medallón)

⟨pendiente de confirmar — ver duda #D2⟩

| Capa | Responsabilidad | Qué entra | Qué no entra |
|---|---|---|---|
| **bronze** | Ingesta fiel a la fuente; sin transformación | Raw data tal cual llega | Modificaciones de valores, joins |
| **silver** | Datos limpios y conformados; schema estable | Limpieza, cast, deduplicación, unificación de fuentes | Lógica de negocio, métricas |
| **gold** | Tablas listas para consumo analítico o de producto | Agregaciones, features, modelos de negocio | Transformaciones de limpieza, raw data |

[RELLENAR: invariantes adicionales del proyecto — qué garantías se mantienen en cada capa.]

---

## Reglas de paso entre capas

> Rellenar al confirmar la opción elegida.

- [RELLENAR: condiciones que un dataset debe cumplir para promoverse de capa.]
- Un dataset no puede saltarse capas sin aprobación de arquitectura (registrar en `decisions.md`).
- Todo cambio de schema entre capas debe reflejarse en el contrato de datos del asset (ver `../quality/slas.md` y sensor FF-01 en `../../../core/sensors/fast-feedback/schema-contract.md`).
