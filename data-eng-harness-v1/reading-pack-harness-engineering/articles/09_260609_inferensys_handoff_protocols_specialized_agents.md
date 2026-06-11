# How to Design Handoff Protocols Between Specialized Agents

**Fuente:** [Inference Systems (inferensys.com)](https://inferensys.com/guides/multi-agent-system-mas-orchestration/how-to-design-handoff-protocols-between-specialized-agents)  
**Autor:** Prasad Kumkar — CEO & MD, Inference Systems  
**Fecha de consulta:** 2026-06-09  
**Categoría:** Multi-Agent System (MAS) Orchestration

---

## 1. ¿Qué es un Handoff Protocol?

Un **handoff protocol** es el **contrato estructurado** que gobierna la transferencia de **contexto, datos y responsabilidad** entre agentes en un workflow multi-agente (por ejemplo, de un *planner* a un *executor*).

Un handoff mal diseñado genera **fragilidad sistémica**: los agentes acaban operando con información incompleta o desactualizada. La guía enseña a definir contratos explícitos que incluyan:

- El contexto de entrada necesario.
- Los criterios de éxito.
- Las instrucciones de fallback.

Esto garantiza que el agente receptor tenga todo lo necesario para proceder de forma autónoma.

---

## 2. Tres patrones de handoff comparados

| Característica | **Structured Contract** | **Shared Blackboard** | **Direct Message Passing** |
|---|---|---|---|
| **Transferencia de contexto** | Payload explícito en el contrato | Implícito vía estado compartido | Mensaje punto a punto |
| **Persistencia de estado** | ✅ | ✅ | ❌ |
| **Desacoplamiento** | Alto (vía interfaz de contrato) | Muy alto (posting anónimo) | Bajo (agentes acoplados) |
| **Audit trail** | Built-in en el ciclo de vida del contrato | Requiere capa de logging separada | Depende del message bus |
| **Fallback handling** | Definido en metadatos del contrato | Gestionado por agentes monitor | Ad-hoc, lógica específica del agente |
| **Validación** | Pre-handoff obligatoria | Post-write posible | Pre-receive opcional |
| **Overhead del orquestador** | Medio (gestiona contratos) | Bajo (auto-organización) | Alto (el orquestador enruta todos los mensajes) |
| **Mejor para** | Workflows regulados / financieros | Investigación / problemas complejos | Cadenas lineales simples |

---

## 3. Conceptos clave

### 3.1 Contratos explícitos

Cada handoff debe incluir un contrato con:

- **Task ID & Parent Context** — Identificador único que enlaza con el origen del workflow.
- **Execution Summary** — Qué se hizo, qué decisiones se tomaron y por qué.
- **Artifacts & State** — Referencias a datos generados, archivos modificados o claves de BD.
- **Success Criteria Met** — Validación explícita de que los objetivos de la subtarea se cumplieron.

### 3.2 Validación en cada punto de handoff

Se implementan pasos de validación para detectar errores **antes** de que se propaguen al siguiente agente en la cadena.

### 3.3 Audit trails inmutables

Registros de auditoría inmutables para depurar transacciones complejas entre agentes. Son fundamentales para sistemas que requieren verificación y trazabilidad.

### 3.4 Shared state / Blackboard architecture

Uso de estado compartido (blackboard) como mecanismo de colaboración fiable entre agentes, especialmente útil en escenarios de investigación o resolución de problemas complejos.

---

## 4. Fallback Logic (Lógica de recuperación)

La lógica de fallback define **qué hacer cuando un agente no puede continuar**. Se activa ante condiciones como:

- Timeouts.
- Umbrales de confianza bajos.
- Errores de recursos.

Se codifica **dentro del propio contrato de handoff**, por ejemplo:

```json
{
  "on_timeout": "escalate_to_supervisor",
  "on_low_confidence": "request_human_review"
}
```

**Sin esta lógica, los fallos provocan deadlocks** en el workflow. Se recomienda usar la capa de observabilidad y monitorización del sistema para rastrear las activaciones de fallback, ya que son señales clave para mejorar las capacidades de los agentes.

---

## 5. Ejemplo de contrato de handoff (JSON)

```json
{
  "handoff_type": "planner_to_executor",
  "workflow_id": "wf_abc123",
  "parent_task_id": "task_plan_1",
  "summary": "Generated SQL query for Q3 sales report using customer_db.",
  "artifacts": {
    "generated_query": "SELECT * FROM sales WHERE quarter=3",
    "db_schema_version": "2.1"
  },
  "validation": {
    "query_syntax_check": "passed",
    "schema_compatibility": "confirmed"
  }
}
```

---

## 6. Errores comunes en el diseño de handoffs (Troubleshooting)

| Error | Descripción |
|---|---|
| **Pérdida de contexto** | El handoff se reduce a un mensaje simple como "Task complete", sin historial ni artefactos. El agente receptor arranca desde cero. |
| **Acciones duplicadas o conflictivas** | Múltiples agentes ejecutan la misma tarea sin coordinación. |
| **Falta de audit trail** | Cuando un handoff falla, no hay registro para diagnosticar la causa. |
| **Fallos silenciosos** | Los errores en los puntos de handoff no se detectan ni reportan. |
| **Deadlocks** | Los agentes se bloquean mutuamente esperando handoffs que nunca se completan. |
| **Uso de strings simples como mensaje** | Anti-patrón: un string plano no transporta estructura, metadatos ni validación. |

---

## 7. Referencias

- **Artículo original:** [How to Design Handoff Protocols Between Specialized Agents — Inference Systems](https://inferensys.com/guides/multi-agent-system-mas-orchestration/how-to-design-handoff-protocols-between-specialized-agents)
- **Guía relacionada:** [Architecting a MAS with Built-In Verification and Audit Loops — Inference Systems](https://inferensys.com/guides/multi-agent-system-mas-orchestration/)
- **Guía relacionada:** [How to Architect a Multi-Agent System for Complex Workflows — Inference Systems](https://inferensys.com/guides/multi-agent-system-mas-orchestration/)
