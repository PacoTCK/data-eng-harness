# SLAs de freshness y calidad por dataset

> Propósito: registrar los acuerdos de nivel de servicio de cada dataset producido
> por este proyecto. El sensor FF-02 (Data Quality) del core usa estos valores para
> verificar freshness, volume y calidad en cada ejecución de CI.
> Fuente de verdad: este fichero. Los contratos de datos individuales en `contracts/`
> deben ser coherentes con las entradas de esta tabla.

---

## Cómo rellenar este fichero

1. Añade una fila por dataset al momento de acordar su SLA con el equipo o cliente.
2. El campo `sla_hours` es el tiempo máximo desde que el dato en origen está disponible
   hasta que el dataset destino debe reflejar esa actualización.
3. `min_rows` y `max_rows` son los umbrales de volume. Si `max_rows` no aplica, escribe `—`.
4. `owner` es la persona o equipo responsable de resolver alertas de SLA.
5. El campo `Contrato` apunta al fichero YAML/JSON en `contracts/` que declara el schema
   completo y las reglas de calidad (coherente con sensor FF-01).

---

## Tabla de SLAs

| Dataset | Capa | Owner | sla_hours | min_rows | max_rows | Freshness check | Contrato |
|---|---|---|---|---|---|---|---|
| [RELLENAR] | ⟨pendiente — ver duda #D2⟩ | [RELLENAR] | ⟨pendiente⟩ | ⟨pendiente⟩ | — | MAX(updated_at) | — |

---

## Definición de campos

| Campo | Tipo | Descripción |
|---|---|---|
| `Dataset` | string | Nombre del dataset (tabla o fichero) tal como aparece en el warehouse/lake |
| `Capa` | enum | La capa del dato: `landing`/`staging`/`marts` o `bronze`/`silver`/`gold`, según la familia elegida (ver duda #D2) |
| `Owner` | string | Persona o equipo responsable (nombre + canal de contacto) |
| `sla_hours` | integer | Horas máximas de latencia end-to-end desde el origen |
| `min_rows` | integer | Número mínimo de filas esperadas; 0 si el dataset puede estar vacío |
| `max_rows` | integer o `—` | Número máximo de filas; omitir si no hay umbral superior |
| `Freshness check` | string | Columna usada para verificar freshness (normalmente `updated_at` o `_loaded_at`) |
| `Contrato` | ruta | Ruta relativa al fichero de contrato en `contracts/` |

---

## Proceso de escalada

Cuando un sensor FF-02 emite `SENSOR FAIL` por violación de SLA:

1. El agente notifica al `owner` del dataset.
2. Si el owner no resuelve en [RELLENAR: tiempo de respuesta], el agente escala al tech lead.
3. Si el fallo es sistémico (varios datasets afectados), se abre una duda en `../dudas.md`.
